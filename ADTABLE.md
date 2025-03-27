# MemDataTable存储
以下详细解析MemDataTable的存储结构实现，通过关键源码展示其核心设计：

---

### **一、哈希表结构（THashMap）**

#### 1.1 节点内存布局
```cpp
// HashMap节点结构（以存储广告信息为例）
template <typename TSchema>
struct HashMap<TSchema>::Node {
    typename TSchema::TPrimaryKey key;  // 主键（4字节）
    typename TSchema::TTuple value;     // 定长数据部分（如32字节）
    uint32_t next;                      // 下个节点虚拟地址（4字节）
    
    // 总大小：40字节/节点
    // 内存布局示例：
    // [ key(4B) | value(32B) | next(4B) ]
};
```

#### 1.2 哈希表初始化
```cpp
// 创建哈希表并分配桶数组
template <typename TSchema>
int HashMap<TSchema>::create(uint32_t row_num, float hash_ratio, DataPool<SlabMempool32>* p_pool) {
    _buckets.resize(row_num, SlabMempool32::NULL_VADDR); // 初始化所有桶为空
    _p_pool = p_pool; // 绑定内存池
}
```

#### 1.3 插入操作流程
```cpp
template <typename TSchema>
int HashMap<TSchema>::insert(const TTuple& value, TVaddr* p_vaddr) {
    // 从内存池分配节点空间
    *p_vaddr = _p_pool->malloc(sizeof(Node)); 
    
    // 构造节点数据
    Node node;
    node.key = value.primary_key();
    node.value = value;
    
    // 写入内存池
    _p_pool->write(*p_vaddr, &node, sizeof(node));
    
    // 计算哈希值并插入链表头部
    uint32_t hash_idx = hash_func(node.key) % _buckets.size();
    node.next = _buckets[hash_idx]; // 原头节点变为next
    _buckets[hash_idx] = *p_vaddr;  // 新节点成为头节点
}
```

---

### **二、内存池管理（SlabMempool32）**

#### 2.1 内存块结构
```cpp
struct SlabMempool32::mem_block_t {
    char*       start;          // 内存块起始地址
    uint32_t    item_size;      // 每个item的固定大小（如40字节）
    uint32_t    free_item;      // 首个空闲item的偏移量
    uint32_t    max_item_num;   // 本块最大item数量
};

// 示例内存块布局：
// +----------------+---------+---------+-----+
// | item0 (40B)    | item1   | ...     |itemN|
// +----------------+---------+---------+-----+
// 空闲链表通过item内部的头4字节链接
```

#### 2.2 虚拟地址编码
```cpp
// 32位虚拟地址组成：
// 高12位 - 块索引 | 低20位 - 块内偏移
uint32_t SlabMempool32::make_vaddr(uint32_t block_idx, uint32_t offset) {
    return (block_idx << _idx2_bits) | (offset & _idx2_mask);
}

// 地址解码
void split_vaddr(TVaddr vaddr, uint32_t& block_idx, uint32_t& offset) {
    block_idx = vaddr >> _idx2_bits; 
    offset = vaddr & _idx2_mask;
}
```

#### 2.3 内存分配过程
```cpp
TVaddr SlabMempool32::malloc(size_t len) {
    // 1. 选择合适slab（最小满足长度的slab）
    for (auto& slab : _slabs) {
        if (slab.item_size >= len) {
            // 2. 查找空闲item
            if (slab.free_list != NULL_VADDR) {
                TVaddr addr = slab.free_list;
                slab.free_list = *(uint32_t*)get_item_ptr(addr); // 更新链表
                return addr;
            }
            // 3. 分配新内存块
            mem_block_t new_block;
            new_block.start = new char[BLOCK_SIZE]; // 分配1MB块
            _blocks.push_back(new_block);
            // ... 初始化块内空闲链表
            return make_vaddr(_blocks.size()-1, 0);
        }
    }
    return NULL_VADDR;
}
```

---

### **三、不定长字段存储（VarField）**

#### 3.1 变长字段内存布局
```cpp
// VectorVar存储示例（存储3个int）
template <typename T, int MAX_SIZE>
class VectorVar : public VarField<VectorVarS<T, MAX_SIZE>> {
    // 序列化后的内存结构：
    // +--------+---------------------+--------+
    // |头(2B)3 |数据(int×3=12B)      |尾(2B)3 |
    // +--------+---------------------+--------+
    // 总长度：2+12+2=16字节
};
```

#### 3.2 写入内存池操作
```cpp
template <typename TPool>
int VectorVar<T, MAX_SIZE>::save(TPool& pool, TVaddr* p_vaddr) const {
    // 计算总长度
    uint32_t body_size = this->head * sizeof(T);
    uint32_t total_size = sizeof(THead) + body_size + sizeof(TTail);
    
    // 分配连续内存
    *p_vaddr = pool.malloc(total_size);
    
    // 分段写入（零拷贝优化）
    Buffer data[3] = {
        Buffer(&this->_head, sizeof(THead)),
        Buffer(this->_p_body, body_size),
        Buffer(&this->_tail, sizeof(TTail))
    };
    pool.writev(*p_vaddr, data, 3);
}
```

---

### **四、MemDataTable整体结构**

#### 4.1 存储架构图
```
MemDataTable
├── **主哈希表 (THashMap)**
│   ├── Bucket数组: [TVaddr]（每个桶指向链表头）
│   └── 节点链表: 
│       ├── Node1 → Node2 → ...（通过next链接）
│       └── 每个Node包含：
│           ├── 主键 (定长)
│           ├── 定长数据 (内联存储)
│           └── 不定长字段指针 (TVaddr数组)
│
├── **定长内存池 (SlabMempool32)**
│   ├── Block0: 存储40字节节点（Node结构）
│   ├── Block1: 存储其他定长数据
│   └── 虚拟地址映射：块索引 + 偏移量
│
└── **变长内存池 (DataPool数组)**
    ├── VarPool1: 存储VectorVar等变长数据
    │   ├── 数据块1: [头|数据体|尾]
    │   └── 数据块2: ...
    └── VarPool2: 其他变长类型...
```

#### 4.2 数据插入流程
1. **转换数据格式**  
   ```cpp
   TInnerTuple inner_tuple;
   make_inner_tuple(data_tuple, &inner_tuple); // 提取定长字段
   ```

2. **分配变长字段内存**  
   ```cpp
   for (每个变长字段) {
       VectorVar<T> var_field;
       var_field.save(*_p_var_pools[i], &vaddr); // 写入DataPool
       inner_tuple.set_var_ptr(i, vaddr); // 记录虚拟地址
   }
   ```

3. **插入哈希表**  
   ```cpp
   TVaddr node_vaddr;
   _p_dict->insert(inner_tuple, &node_vaddr); // 分配节点并插入哈希桶
   ```

4. **建立反向映射**  
   ```cpp
   data_tuple._set_vaddr(node_vaddr); // 记录节点地址供后续查找
   ```

---

### **五、关键设计总结**

1. **虚拟地址空间**  
   - 32位地址编码，支持最多4096个内存块（12位块索引），每块最大1MB（20位偏移）
   - 示例地址：`0x12345678` → 块索引0x123，偏移0x45678

2. **内存隔离策略**  
   - 定长节点与变长数据分别存储在不同池
   - 不同大小的变长字段使用独立DataPool（如4B、8B、16B slab）

3. **数据一致性保障**  
   - 变长字段通过头尾校验值（如`head==tail`）检测内存损坏
   - 释放内存时递归释放关联的变长字段

4. **存储示例**  
   - **插入一条广告数据**：
     - 主键：`ad123`
     - 定长字段：`{price=5, width=300, height=250}`
     - 变长字段：`keywords=["tech", "sports"]`
   - **内存分布**：
     ```
     [哈希节点] 0x1000: key=ad123 | price=5 | width=300 | height=250 | keywords_ptr=0x2000
     [变长池] 0x2000: head=2 | "tech"sports" | tail=2
     ``` 

通过这种分层存储设计，MemDataTable实现了高效的内存管理和灵活的数据存储能力。
