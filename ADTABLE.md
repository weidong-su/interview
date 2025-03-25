# 内存池

SlabMempool32 的设计中同时存在 `_blocks` 和 `_slabs` 两个核心结构，其设计逻辑和内存管理策略如下：

---

### **1. 为什么需要 _blocks 和 _slabs 两个结构？**
#### **设计目标**
- **内存分类管理**：将不同大小的内存请求分类到不同的 Slab 中，每个 Slab 管理特定大小的内存分配。
- **大块内存池**：通过 `_blocks` 统一管理所有实际分配的大块内存，避免频繁调用系统内存分配。
- **地址编码**：虚拟地址（`TVaddr`）通过高位编码 Block 索引，低位编码 Block 内的偏移，需统一管理 Block 索引。

#### **结构分工**
1. **`_slabs`（Slab 管理）**：
   - 每个 `mem_slab_t` 对应一种固定大小的内存分配（如 8B、16B、32B）。
   - 维护该 Slab 的空闲链表（`free_list`）和已分配的 Block 列表（`block_list`）。
   - 示例：若存在 8B 的 Slab，则所有 8B 的内存请求都由该 Slab 处理。

2. **`_blocks`（内存块池）**：
   - 每个 `mem_block_t` 是一块连续内存，被切割为多个相同大小的 Item（由所属 Slab 决定大小）。
   - 所有 Block 的实际内存指针（`start`）统一存储在 `_blocks` 中，方便通过虚拟地址快速定位内存。
   - 示例：一个 1MB 的 Block 可能被切割为 1024 个 1024B 的 Item。

#### **为何不能只用 _slabs？**
- **地址编码限制**：虚拟地址的高位需编码 Block 索引，低位编码偏移，这要求所有 Block 的索引必须全局唯一。
- **内存复用**：统一回收所有 Block 的内存（如析构时遍历 `_blocks` 释放 `start` 指针）。

---

### **2. malloc 逻辑解析**
#### **源码流程**
```cpp
TVaddr SlabMempool32::malloc(size_t len) {
    // 1. 根据请求大小找到对应的 Slab
    int slab_idx = find_slab(len);
    mem_slab_t& slab = _slabs[slab_idx];
    uint32_t item_size = _slab_lens[slab_idx];

    // 2. 优先从空闲链表分配
    if (slab.free_list != NULL_VADDR) {
        return malloc_from_freelist(slab);
    }

    // 3. 从现有 Block 或新 Block 分配
    return malloc_from_block(slab, item_size);
}
```

#### **关键步骤**
1. **查找 Slab**：
   - `find_slab(len)` 找到第一个满足 `item_size >= len` 的 Slab。
   - 示例：若请求 10B，找到 16B 的 Slab。

2. **空闲链表分配**：
   - 若 Slab 的 `free_list` 非空，直接从链表头部取出一个空闲地址。
   - 空闲链表通过内存复用实现：释放的 Item 内存中存储下一个空闲地址（类似链式栈）。

3. **Block 分配**：
   - 若当前 Slab 的最后一个 Block 有剩余空间，分配其中的下一个 Item。
   - 若所有 Block 已满，创建新 Block 并分配第一个 Item。

#### **创建新 Block**
```cpp
uint32_t SlabMempool32::malloc_from_new_block(mem_slab_t& slab, uint32_t item_size) {
    // 计算 Block 大小（不超过 MAX_BLOCK_SIZE）
    mem_block_t new_block;
    uint64_t n_bytes = calc_block_size(new_block, item_size);

    // 分配内存并初始化
    new_block.start = new char[n_bytes];
    new_block.item_size = item_size;
    new_block.free_item = 1; // 0 号 Item 将被返回

    // 更新数据结构
    _blocks.push_back(new_block);
    slab.block_list.push_back(_blocks.size() - 1);

    return make_vaddr(new_block_idx, 0); // 组合虚拟地址
}
```

---

### **3. free 逻辑解析**
#### **源码流程**
```cpp
int SlabMempool32::free(const TVaddr& vaddr) {
    // 1. 解析虚拟地址得到 Block 索引和偏移
    uint32_t block_idx, offset;
    split_vaddr(vaddr, block_idx, offset);

    // 2. 找到对应的 Block 和 Slab
    mem_block_t& block = _blocks[block_idx];
    int slab_idx = find_slab(block.item_size);
    mem_slab_t& slab = _slabs[slab_idx];

    // 3. 将地址插入空闲链表头部
    push_into_freelist(slab, vaddr);
    return 0;
}
```

#### **关键步骤**
1. **地址解析**：
   - 虚拟地址高位为 Block 索引，低位为 Block 内偏移。
   - 示例：`vaddr = 0x12345678`，`block_idx = 0x1234`，`offset = 0x5678`。

2. **找到所属 Slab**：
   - 通过 Block 的 `item_size` 确定其所属的 Slab。

3. **回收内存**：
   - 将释放的地址插入 Slab 空闲链表的头部。
   - 内存复用：释放的 Item 内存中存储原链表头地址，形成链式结构。

---

### **4. 示例场景**
#### **场景描述**
- **Slab 配置**：包含 8B 和 16B 的 Slab。
- **操作序列**：
  1. `malloc(10)`：找到 16B Slab，分配新 Block（假设 Block 可容纳 100 个 Item）。
  2. `malloc(10)`：从同一 Block 分配第二个 Item。
  3. `free(addr1)`：将 `addr1` 插入 16B Slab 的空闲链表。
  4. `malloc(10)`：直接从空闲链表分配 `addr1`。

#### **虚拟地址示例**
- Block 0 属于 16B Slab，虚拟地址计算：
  ```cpp
  vaddr = make_vaddr(0, 0); // 0x00000000
  vaddr = make_vaddr(0, 1); // 0x00000001
  ```

#### **内存布局**
```
Block 0 (16B Slab):
+---------+---------+-----+---------+
| Item 0  | Item 1  | ... | Item 99 |
+---------+---------+-----+---------+
```

---

### **总结**
- **`_blocks`**：统一管理所有大块内存，解决内存分配和地址编码问题。
- **`_slabs`**：按内存大小分类管理分配策略，提高内存利用率。
- **malloc/free**：通过空闲链表和 Block 分配策略实现高效内存复用。

# HASHMAP
我将从源码角度详细解析 `HashMap` 的实现，并展开其调用的核心函数。以下以 **插入（insert）** 操作为主线，串联关键函数调用关系。

---

### 一、数据结构关系图
```plaintext
+----------------+       +----------------+       +----------------+
|   HashMap      |       |   LinkList     |       |   DataPool     |
|----------------|       |----------------|       |----------------|
| - _p_hash[]    | ----> | - _p_head      | ----> | - malloc()     |
| - _p_pool      |       | - _p_pool      |       | - free()       |
| - _hash_size   |       +----------------+       | - read()       |
+----------------+                               +----------------+
```

---

### 二、核心函数调用链（以`insert`为例）
```plaintext
HashMap::insert()
│
├── hash_entry()              // 计算哈希桶索引
│
├── TList::_load()           // 加载对应桶的链表
│   ├── DataPool::read()     // 读取链表头节点地址
│
├── TList::insert()          // 向链表插入数据
│   ├── find_insert_pos()    // 查找插入位置
│   │   ├── real_address()   // 虚拟地址转真实地址
│   │   └── TCmp::sort_op()  // 比较节点数据
│   │
│   ├── malloc_node()        // 分配新节点内存
│   │   └── DataPool::malloc() 
│   │
│   ├── set_data()           // 写入数据到节点
│   │   └── memcpy()         // 内存拷贝
│   │
│   └── set_next()           // 调整链表指针
│
├── TList::_unload()         // 卸载链表（仅存储头节点地址）
│
└── DataPool::write()        // 可选：持久化数据到存储
```

---

### 三、关键函数源码解析

#### 1. `HashMap::create()` - 初始化哈希表
```cpp
template <typename TSchema>
int HashMap<TSchema>::create(uint32_t item_num, float hash_ratio, TDataPool* p_pool) {
    // 选择素数作为桶大小
    _hash_size = hash_size(item_num, hash_ratio); 
    // 分配桶数组内存
    _p_hash = new typename TDataPool::TVaddr[_hash_size]; 
    // 初始化每个桶为空链表
    for (uint32_t i = 0; i < _hash_size; ++i) {
        _p_hash[i] = TDataPool::null();
    }
    _p_pool = p_pool; // 关联内存池
    return 0;
}
```
- **关键点**：使用素数桶减少哈希冲突，依赖 `TDataPool::null()` 表示空指针。

---

#### 2. `HashMap::insert()` - 插入数据
```cpp
template <typename TSchema>
int HashMap<TSchema>::insert(const TTuple& tuple, typename TDataPool::TVaddr* vaddr) {
    uint32_t bucket = hash_entry(tuple.primary_key()); // 计算桶索引
    TList list;
    list._load(_p_hash[bucket], _p_pool); // 加载链表
    
    int ret = list.insert(tuple, true, vaddr); // 插入排序
    if (ret >= 0) ++_node_num;
    
    _p_hash[bucket] = list._head(); // 更新桶头节点
    list._unload();
    return ret;
}
```
- **关键调用**：`TList::insert()` 实现排序插入。

---

#### 3. `TList::insert()` - 链表插入
```cpp
template <class T, class TCmp, class TDataPool>
int LinkList<T, TCmp, TDataPool>::insert(const T& value, bool need_sort, TVaddr* vaddr) {
    if (!need_sort) return insert_head(value, vaddr); // 头部插入
    
    ListNode* prev_node = nullptr;
    find_insert_pos(value, prev_node); // 查找插入位置
    
    return insert_after(value, prev_node, vaddr); // 插入到prev_node后
}
```
- **排序逻辑**：通过 `TCmp::sort_op` 确定节点顺序。

---

#### 4. `find_insert_pos()` - 查找插入位置
```cpp
template <class T, class TCmp, class TDataPool>
int LinkList<T, TCmp, TDataPool>::find_insert_pos(const T& value, ListNode*& prev_node) const {
    TVaddr cur = _p_head;
    while (cur != TDataPool::null()) {
        ListNode* cur_node = real_address(cur); // 虚拟地址转真实地址
        if (TCmp::sort_op(cur_node->data(), value)) { // 比较
            prev_node = cur_node;
            cur = cur_node->next();
        } else {
            break;
        }
    }
    return 0;
}
```
- **虚拟地址转换**：`real_address()` 调用 `DataPool::read()` 获取真实内存指针。

---

#### 5. `DataPool::malloc()` - 内存分配
```cpp
template <typename TPool>
typename DataPool<TPool>::TVaddr DataPool<TPool>::malloc(size_t size) {
    return _p_pool->malloc(size); // 委托给具体内存池实现
}
```
- **实际行为**：由 `TPool`（如 `MemPool`）实现具体的内存分配策略。

---

### 四、示例：插入操作的完整流程
假设插入数据 `TTuple{id=5}`，哈希桶索引为 `2`：
1. **加载链表**：通过 `list._load(_p_hash[2], _p_pool)` 加载桶2的链表。
2. **查找位置**：遍历链表，找到第一个不小于5的节点位置。
3. **分配节点**：调用 `DataPool::malloc(node_size)` 分配新节点内存。
4. **写入数据**：`memcpy` 将数据拷贝到节点内存。
5. **调整指针**：将新节点的 `_next` 指向后续节点，前驱节点指向新节点。
6. **更新桶头**：将链表的新头节点地址写回 `_p_hash[2]`。

---

### 五、关键设计思想总结

1. **虚拟地址（TVaddr）**  
   - 抽象物理内存地址，使数据存储可扩展（如支持磁盘存储）。
   - 通过 `DataPool::read()` 和 `DataPool::malloc()` 实现地址转换。

2. **内存池（DataPool）**  
   - 统一管理内存分配/释放，减少碎片。
   - 支持 `malloc`、`free`、`read`、`write` 等基础操作。

3. **排序链表（LinkList）**  
   - 插入时保持有序，优化查询效率。
   - 通过模板参数 `TCmp` 实现灵活的比较逻辑。

4. **素数哈希桶**  
   - 使用预定义的素数表选择桶大小，降低冲突概率。
   - 提供 `PRIME_LIST` 和 `REFINED_PRIME_LIST` 两种策略。

---

### 六、性能优化点

1. **缓存友好性**  
   - `ListNode` 将 `_next` 置于结构体头部，提升CPU缓存命中率。
   - 使用 `_mm_prefetch` 预取下一节点数据。

2. **批量操作**  
   - `DataPool::writev` 支持批量写入，减少I/O次数。

3. **延迟释放**  
   - `free` 操作将节点加入 `_dirty_nodes`，由 `recycle` 统一释放，减少锁竞争。

---

通过此解析，可以清晰看到从 `HashMap` 的插入操作到底层内存分配的全链路实现，理解其如何通过组合 `LinkList` 和 `DataPool` 实现高效、灵活的数据存储。

简单示例：
```
#include <iostream>
#include "bsl/string.h"
#include "HashMap.h"

// 定义一个简单的Schema
struct Schema {
    typedef int TPrimaryKey;
    typedef std::pair<int, std::string> TTuple;
    typedef std::less<int> TCmp;
    typedef DataPool<DefaultPool> TDataPool;

    static uint64_t hash_value(const TPrimaryKey& pk) {
        return std::hash<int>()(pk);
    }
};

int main() {
    // 创建HashMap实例
    HashMap<Schema> hashmap("example_map");

    // 创建数据池
    Schema::TDataPool data_pool("example_pool");
    if (data_pool.create(new DefaultPool())) {
        std::cerr << "Failed to create data pool" << std::endl;
        return 1;
    }

    // 创建HashMap
    if (hashmap.create(10, 0.75, &data_pool)) {
        std::cerr << "Failed to create hash map" << std::endl;
        return 1;
    }

    // 插入数据
    Schema::TTuple tuple1(1, "value1");
    Schema::TTuple tuple2(2, "value2");
    if (hashmap.insert(tuple1) != 0 || hashmap.insert(tuple2) != 0) {
        std::cerr << "Failed to insert tuples" << std::endl;
        return 1;
    }

    // 查找数据
    Schema::TPrimaryKey pk1 = 1;
    auto iter = hashmap.seek(pk1);
    if (!iter.is_null()) {
        std::cout << "Found tuple: " << iter->first << ", " << iter->second << std::endl;
    } else {
        std::cerr << "Tuple not found" << std::endl;
    }

    // 移除数据
    hashmap.remove(pk1);

    // 再次查找数据
    iter = hashmap.seek(pk1);
    if (iter.is_null()) {
        std::cout << "Tuple removed successfully" << std::endl;
    } else {
        std::cerr << "Failed to remove tuple" << std::endl;
    }

    return 0;
}
```

# DATAPOOL

### DataPool 源码解析

#### **核心设计目标**
`DataPool` 是内存池的 **高级封装模板类**，用于统一管理不同底层内存池（如 `SlabMempool32`）的 **内存分配、释放、监控**，并支持 **延迟回收** 机制。其核心职责是：
1. **抽象内存池接口**：封装底层内存池的 `malloc`、`free` 等操作。
2. **批量回收优化**：通过 `_dirty_nodes` 暂存待释放地址，减少频繁释放的开销。
3. **监控与统计**：收集内存池的使用指标（如内存消耗、节点数量）。

---

#### **关键代码解析**

##### **1. 类定义与成员变量**
```cpp
template <typename TPool>
class DataPool {
    TPool* _p_pool;            // 底层内存池实例（如 SlabMempool32）
    bsl::string _name;         // 内存池名称（用于监控标识）
    std::vector<TVaddr> _dirty_nodes; // 延迟释放的虚拟地址列表
};
```
- **`TPool`**：模板参数，需实现 `IDataPool` 接口（如 `malloc`、`free`）。
- **`_dirty_nodes`**：延迟释放队列，避免高频 `free` 操作影响性能。

---

##### **2. 核心方法实现**
###### **初始化与销毁**
```cpp
int create(TPool* p_pool) {
    if (!p_pool) return -1;
    _p_pool = p_pool; // 绑定底层内存池
    return 0;
}

void recycle() {
    for (auto vaddr : _dirty_nodes) {
        _p_pool->free(vaddr); // 批量释放内存
    }
    _dirty_nodes.clear();
}
```
- **延迟释放**：调用 `free` 仅标记地址，`recycle` 时统一释放。

---

###### **内存分配与释放**
```cpp
TVaddr malloc(size_t size) {
    return _p_pool->malloc(size); // 委托底层池分配
}

void free(const TVaddr& vaddr) {
    _dirty_nodes.push_back(vaddr); // 加入延迟释放队列
}
```
- **分配代理**：直接调用底层池的 `malloc`。
- **延迟释放**：释放操作被推迟到 `recycle`，减少锁竞争和碎片化。

---

###### **数据读写**
```cpp
const void* read(const TVaddr& vaddr) const {
    const char* p_obj = NULL;
    _p_pool->read(vaddr, p_obj); // 读取物理地址
    return p_obj;
}

int write(const TVaddr& vaddr, const Buffer& buf) {
    return _p_pool->write(vaddr, buf); // 写入数据
}
```
- **直接转发**：读写操作透传到底层内存池。

---

###### **对象序列化**
```cpp
template <class TObj>
const TObj* read_object(const TVaddr& vaddr) const {
    return static_cast<const TObj*>(read(vaddr)); // 反序列化对象
}

template <class TObj>
int write_object(const TVaddr& vaddr, const TObj& obj) {
    Buffer buf(reinterpret_cast<const char*>(&obj), sizeof(TObj));
    return write(vaddr, buf); // 序列化对象
}
```
- **类型安全**：通过模板将对象转换为二进制缓冲区。

---

##### **监控与统计**
```cpp
void monitor(bsl::var::Dict& dict, bsl::ResourcePool& rp) const {
    bsl::var::Dict& inner_dict = rp.create<bsl::var::Dict>();
    _p_pool->monitor(inner_dict, rp); // 收集底层池指标
    dict["NAME"] = rp.create<bsl::var::String>(_name);
    dict["INNER_POOL"] = inner_dict; // 汇总到外层字典
}
```
- **指标聚合**：将底层池的监控数据（如内存消耗、节点数）整合到统一字典。

---

#### **设计亮点**
1. **延迟释放优化**  
   - **减少锁竞争**：批量释放降低多线程环境下的锁争用。
   - **提升性能**：避免频繁调用底层 `free` 的系统开销。

2. **统一接口抽象**  
   - **适配多种内存池**：通过模板参数 `TPool` 支持不同实现（如 `SlabMempool32`、`BuddyPool`）。

3. **监控集成**  
   - **层级化统计**：通过 `monitor` 方法汇总内存池状态，便于系统级资源分析。

---

#### **使用示例**
##### **场景**：广告系统的标题存储
```cpp
// 1. 初始化内存池
SlabMempool32 slab_pool;
DataPool<SlabMempool32> title_pool("title_pool");
title_pool.create(&slab_pool);

// 2. 分配内存并写入数据
const char* title = "夏季大促销";
Buffer buf(title, strlen(title));
TVaddr vaddr = title_pool.malloc(buf.size());
title_pool.write(vaddr, buf);

// 3. 读取数据
const char* stored_title = static_cast<const char*>(title_pool.read(vaddr));
std::cout << "标题: " << stored_title << std::endl;

// 4. 标记释放（延迟到 recycle）
title_pool.free(vaddr);

// 5. 批量回收
title_pool.recycle();
```

---

#### **总结**
`DataPool` 通过 **模板封装** 和 **延迟释放机制**，为内存池提供了高效、易用的管理接口。其设计适用于需要高频内存操作且关注性能监控的系统（如广告检索、实时计算），是构建高性能C++中间件的关键组件。
