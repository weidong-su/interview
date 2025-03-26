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

### 源码解析：DataPool（内存池封装）

#### 一、核心设计思想
**DataPool** 是一个 **通用内存池的封装模板类**，其核心职责是提供对底层内存池（`TPool`）的统一管理接口，包括内存分配、释放、读写及监控等功能。它通过 **延迟释放机制** 优化内存回收效率，并向上层提供 **对象序列化/反序列化** 的便捷接口。

---

#### 二、类结构与关键成员
```cpp
template <typename TPool>
class DataPool {
public:
    typedef typename TPool::TVaddr TVaddr; // 虚拟地址类型（由 TPool 定义）

private:
    TPool*              _p_pool;     // 底层内存池实例
    bsl::string         _name;       // 内存池名称（用于监控）
    std::vector<TVaddr> _dirty_nodes;// 延迟释放队列
};
```

---

#### 三、核心方法详解

##### 1. **内存管理方法**
| 方法                  | 功能描述                                                                 | 实现细节                                                                 |
|-----------------------|------------------------------------------------------------------------|--------------------------------------------------------------------------|
| `create(TPool* pool)` | 绑定底层内存池                                                          | 将 `_p_pool` 指向外部传入的 `TPool` 实例                                  |
| `malloc(size)`        | 分配指定大小的内存块                                                    | 委托给底层池 `_p_pool->malloc(size)`                                      |
| `free(vaddr)`         | **标记内存为待释放**（加入延迟释放队列）                                 | 将虚拟地址 `vaddr` 存入 `_dirty_nodes` 向量                                |
| `recycle()`           | **批量释放所有标记内存**                                                | 遍历 `_dirty_nodes`，调用 `_p_pool->free(vaddr)` 并清空队列                |
| `clear()`             | 清空整个内存池内容                                                      | 调用底层池的 `_p_pool->clear()`                                           |

##### 2. **数据读写方法**
| 方法                          | 功能描述                                                                 |
|-------------------------------|--------------------------------------------------------------------------|
| `read(vaddr)`                 | 读取虚拟地址指向的内存数据（返回 `const void*` 指针）                     |
| `write(vaddr, buf)`           | 将缓冲区 `buf` 写入虚拟地址 `vaddr` 处                                    |
| `read_object<TObj>(vaddr)`    | 直接读取 `TObj` 类型的对象（需确保内存布局匹配）                           |
| `write_object<TObj>(vaddr, obj)` | 将对象 `obj` 序列化到 `vaddr` 处（内存拷贝）                              |

##### 3. **监控方法**
```cpp
void monitor(bsl::var::Dict& dict, bsl::ResourcePool& rp) const {
    // 收集名称和底层池状态
    dict["NAME"] = ...;
    dict["INNER_POOL"] = _p_pool->monitor(...); 
}
```
- 通过 `bsl::var::Dict` 返回内存池的运行状态（如内存使用量、碎片率等）。

---

#### 四、关键设计点

##### 1. **延迟释放机制**
- **问题**：频繁调用底层池的 `free` 可能带来性能开销（如锁竞争）。
- **方案**：`free()` 仅标记待释放地址，由 `recycle()` 统一批量释放。
- **优点**：减少与底层池的交互次数，提升高频释放场景性能。

##### 2. **对象序列化**
- **直接内存操作**：`read_object`/`write_object` 通过 `memcpy` 实现对象与内存块的转换。
- **注意事项**：仅适用于 **POD类型**（无构造函数、虚函数等），否则可能引发未定义行为。

##### 3. **底层池抽象**
- **接口要求**：`TPool` 必须实现以下接口：
  ```cpp
  class TPool {
  public:
      typedef ... TVaddr; // 虚拟地址类型
      static TVaddr null(); // 空地址标识
      TVaddr malloc(size_t size); 
      void free(TVaddr vaddr);
      void clear();
      int read(TVaddr vaddr, const char*& p_obj); 
      int write(TVaddr vaddr, const Buffer& buf);
  };
  ```

---

#### 五、使用示例

##### 1. **定义底层内存池**
```cpp
// 示例：基于 Slab 分配器的内存池
class SlabMempool32 {
public:
    typedef uint32_t TVaddr;
    static TVaddr null() { return 0xFFFFFFFF; }
    
    TVaddr malloc(size_t size) { /* 实现分配逻辑 */ }
    void free(TVaddr vaddr) { /* 实现释放逻辑 */ }
    // ... 其他接口
};
```

##### 2. **初始化 DataPool**
```cpp
SlabMempool32 slab_pool;
DataPool<SlabMempool32> data_pool("my_pool");
data_pool.create(&slab_pool); // 绑定底层池
```

##### 3. **分配与使用内存**
```cpp
// 分配内存
auto addr = data_pool.malloc(sizeof(int));

// 写入数据
int value = 42;
data_pool.write_object(addr, value);

// 读取数据
const int* p_value = data_pool.read_object<int>(addr);

// 标记释放
data_pool.free(addr);

// 批量回收
data_pool.recycle();
```

##### 4. **监控状态**
```cpp
bsl::ResourcePool rp;
bsl::var::Dict monitor_data;
data_pool.monitor(monitor_data, rp);
std::cout << "Pool Status: " << monitor_data.to_json() << std::endl;
```

---

#### 六、潜在问题与注意事项

1. **线程安全**  
   - 代码未显式处理线程同步，若在多线程环境中使用，需外部加锁。

2. **对象类型限制**  
   - `read_object`/`write_object` 仅适用于 **POD 类型**，复杂对象需自行序列化。

3. **内存泄漏风险**  
   - 需确保 `recycle()` 被定期调用，否则 `_dirty_nodes` 会持续增长。

4. **性能权衡**  
   - 延迟释放提升性能，但可能导致内存峰值升高，需根据场景调整 `recycle()` 调用频率。

# MemDataTable

### 源码解析：MemDataTable（基于 DataPool 的内存数据表）

#### 一、核心设计思想
**MemDataTable** 是基于 **DataPool** 实现的 **内存正排表**，负责结构化数据的存储与管理。其核心特点：
- **哈希索引**：通过 `THashMap` 实现主键快速查找，索引数据存储在 **DataPool** 中。
- **内存池分层**：分 **固定字段池**（`_p_hash_pool`）和 **变长字段池**（`_p_var_pools`），均由 `DataPool` 管理。
- **延迟释放**：利用 `DataPool::free` 的延迟回收机制优化高频更新场景性能。

---

#### 二、关键源码解析（结合 DataPool）

##### 1. **创建内存池与哈希表（`create`）**
```cpp
template <typename TSchema>
int MemDataTable<TSchema>::create(uint32_t row_num, float hash_ratio, const AdTableBaseEnv* adtable_ctx) {
    // 1. 创建哈希表（THashMap）及其底层 DataPool
    if (create_dict(row_num, hash_ratio) < 0) { /* 错误处理 */ }

    // 2. 创建变长字段的 DataPool 数组
    if (create_data_pool() < 0) { /* 错误处理 */ }

    return 0;
}

// 创建哈希表及其 DataPool
template <typename TSchema>
int MemDataTable<TSchema>::create_dict(uint32_t row_num, float hash_ratio) {
    // 初始化底层内存池 SlabMempool32
    _p_hash_inner_pool = new SlabMempool32();
    _p_hash_inner_pool->create(slabs, 2, 1024 * 1024); // 预定义块大小

    // 创建 DataPool 封装
    _p_hash_pool = new DataPool<SlabMempool32>(name);
    _p_hash_pool->create(_p_hash_inner_pool); // 绑定底层池

    // 初始化哈希表 THashMap
    _p_dict = new THashMap(name);
    _p_dict->create(row_num, hash_ratio, _p_hash_pool); // 使用 DataPool
}

// 创建变长字段池
template <typename TSchema>
int MemDataTable<TSchema>::create_data_pool() {
    for (size_t i = 0; i < TSchema::VAR_POOL_NUM; ++i) {
        // 每个变长字段独立 DataPool
        _p_inner_pools[i] = new TInnerPool();
        _p_var_pools[i] = new DataPool<TInnerPool>(pool_name);
        _p_var_pools[i]->create(_p_inner_pools[i]);
    }
}
```
- **关键点**：为哈希表和变长字段分别初始化 `DataPool`，实现内存隔离管理。

---

##### 2. **插入数据（`insert`）**
```cpp
template <typename TSchema>
int MemDataTable<TSchema>::insert(const TDataTuple& tuple) {
    // 1. 转换数据格式（固定+变长字段）
    TInnerTuple inner_tuple;
    DataTable<TSchema>::make_inner_tuple(tuple, &inner_tuple);

    // 2. 为变长字段分配内存（通过 DataPool）
    if (insert_var_pools(inner_tuple, tuple) < 0) { /* 错误处理 */ }

    // 3. 插入哈希表（触发 DataPool::malloc）
    TVaddr vaddr;
    int ret = _p_dict->insert(inner_tuple, &vaddr); // 内部调用 _p_hash_pool->malloc()

    // 4. 记录虚拟地址到数据元信息
    const_cast<TDataTuple&>(tuple)._set_vaddr(vaddr);
}
```

**`insert_var_pools`（变长字段分配）**
```cpp
template <typename TSchema>
int MemDataTable<TSchema>::insert_var_pools(TInnerTuple& inner_tuple, const TDataTuple& tuple) {
    for (size_t pool_idx = 0; pool_idx < TSchema::VAR_POOL_NUM; ++pool_idx) {
        // 分配变长字段内存（如字符串）
        TVaddr var_addr = _p_var_pools[pool_idx]->malloc(tuple.var_size(pool_idx));
        
        // 将数据写入 DataPool
        Buffer buf(tuple.var_data(pool_idx), tuple.var_size(pool_idx));
        _p_var_pools[pool_idx]->write(var_addr, buf);

        // 记录虚拟地址到 inner_tuple
        inner_tuple.set_var_addr(pool_idx, var_addr);
    }
}
```
- **关键点**：每个变长字段通过独立的 `DataPool` 分配内存，写入后记录虚拟地址。

---

##### 3. **删除数据（`remove`）**
```cpp
template <typename TSchema>
int MemDataTable<TSchema>::remove(const TPrimaryKey& pk) {
    // 1. 查找数据
    Iterator iter = seek(pk);
    if (iter.is_null()) return -1;

    // 2. 释放变长字段内存（通过 DataPool::free）
    remove_var_pools(*iter, pk);

    // 3. 从哈希表删除（触发 DataPool::free 延迟回收）
    _p_dict->remove(pk); // 内部调用 _p_hash_pool->free()
}

// 释放变长字段内存
template <typename TSchema>
void MemDataTable<TSchema>::remove_var_pools(...) {
    for (size_t pool_idx = 0; pool_idx < TSchema::VAR_POOL_NUM; ++pool_idx) {
        _p_var_pools[pool_idx]->free(data._var_ptr(pool_idx)); // 标记释放
    }
}
```
- **关键点**：删除操作仅标记内存为脏数据，由 `recycle()` 统一回收。

---

##### 4. **内存回收（`recycle`）**
```cpp
template <typename TSchema>
void MemDataTable<TSchema>::recycle() {
    _p_hash_pool->recycle(); // 批量释放哈希表脏内存
    for (auto pool : _p_var_pools) {
        pool->recycle(); // 释放变长字段脏内存
    }
}
```
- **机制**：调用各 `DataPool` 的 `recycle()`，实际执行底层池的 `free`。

---

##### 5. **监控（`monitor`）**
```cpp
template <typename TSchema>
void MemDataTable<TSchema>::monitor(bsl::var::Dict& dict, bsl::ResourcePool& rp) const {
    // 1. 收集哈希表内存信息
    _p_dict->monitor(dict, rp); // 内部调用 _p_hash_pool->monitor()

    // 2. 收集变长字段池信息
    for (auto pool : _p_var_pools) {
        pool->monitor(var_dict, rp); // 各 DataPool 独立上报
    }
}
```

---

#### 三、使用示例

##### 1. **定义 Schema**
```cpp
// 定义用户表结构
struct UserSchema {
    using TPrimaryKey = int; // 用户ID为主键
    using TDataTuple = User; // 数据行类型

    struct User {
        int id;
        std::string name; // 变长字段
    };

    enum { VAR_POOL_NUM = 1 }; // 一个变长字段（name）
};
```

##### 2. **初始化 MemDataTable**
```cpp
MemDataTable<UserSchema> user_table("user");
user_table.create(1000000, 1.5f, &env); // 预期100万行，负载因子1.5
```

##### 3. **插入数据**
```cpp
User user;
user.id = 1001;
user.name = "Alice";
user_table.insert(user); // 触发 DataPool 分配
```

##### 4. **查询数据**
```cpp
auto iter = user_table.seek(1001);
if (!iter.is_null()) {
    std::cout << "User Name: " << iter->name << std::endl;
}
```

##### 5. **删除数据**
```cpp
user_table.remove(1001); // 标记内存待回收
user_table.recycle();    // 实际释放内存
```

##### 6. **监控状态**
```cpp
bsl::var::Dict stats;
user_table.monitor(stats);
std::cout << "Memory Used: " << stats["MEM_CONSUME"] << " bytes" << std::endl;
```

---

#### 四、性能优化点

1. **批量回收**  
   - 高频删除场景下，避免单次 `free` 调用，利用 `recycle()` 批量释放。

2. **内存池调优**  
   - 根据数据大小调整 `SlabMempool32` 的块尺寸，减少内部碎片。

3. **哈希负载监控**  
   - 定期检查 `HASH_RATIO_PERCENT`，过高时需扩容哈希表（重建 `THashMap`）。

4. **预分配内存**  
   - 初始化时预估行数，调用 `DataPool::malloc` 预分配大块内存。

# PlanTable

### 源码解析：PlanTable（广告计划表）

---

#### 一、核心结构定义

##### 1. **Schema 定义（PlanTableS）**
```cpp
class PlanTableS {
public:
    typedef PlanTableInnerTuple   TInnerTuple;  // 内存存储结构
    typedef PlanTableDataTuple    TDataTuple;   // 外部数据接口
    typedef Uint32Key             TPrimaryKey;  // 主键类型（uint32_t）
    typedef SlabMempool32         TInnerPool;   // 内存池类型
    typedef DataPool<TInnerPool>  TDataPool;    // 内存池封装
    typedef TInnerPool::TVaddr    TVaddr;       // 虚拟地址类型

    static const char* PRIMARY_KEY_FIELDS;      // 主键字段名（"plan_id"）
    enum { VAR_POOL_NUM = 1 };                  // 变长字段数量（city_region）
    
    // 数据转换方法
    static void make_inner_tuple(const TDataTuple &tuple, TInnerTuple* inner);
    static bool is_valid_data_tuple(const TDataTuple &tuple);
    static void make_data_tuple(const PlanTableDataTupleRef& ref, TDataTuple* tuple);
};
```
- **功能**：定义表结构元信息，包括内存布局、主键、数据校验逻辑。

---

##### 2. **内存存储结构（PlanTableInnerTuple）**
```cpp
struct PlanTableInnerTuple {
    uint32_t plan_id;       // 计划ID（主键）
    uint64_t region;        // 区域编码
    TVaddr _var_ptrs[VAR_POOL_NUM]; // 变长字段虚拟地址（city_region）

    TPrimaryKey primary_key() const { return plan_id; }
};
```
- **关键字段**：`_var_ptrs` 存储变长字段在内存池中的地址（如 `city_region`）。

---

##### 3. **数据操作类（PlanTableDataTuple & Ref）**
```cpp
// 数据操作类（用于插入/更新）
class PlanTableDataTuple {
    DEFINE_DATA_TUPLE(PlanTableDataTuple, PlanTableS) // 生成基础方法
    DEFINE_DATA_TUPLE_MEMBER(uint32_t, plan_id);      // 标量字段
    DEFINE_DATA_TUPLE_MEMBER(uint64_t, region);
    DEFINE_DATA_TUPLE_VECTOR_MEMBER(uint32_t, city_region, 128); // 变长向量字段
    // ... 其他方法
};

// 数据引用类（用于查询）
class PlanTableDataTupleRef {
    DEFINE_DATA_TUPLE_REF(PlanTableDataTupleRef, PlanTableS) // 生成引用结构
    DEFINE_DATA_TUPLE_REF_MEMBER(uint32_t, plan_id);
    DEFINE_DATA_TUPLE_REF_MEMBER(uint64_t, region);
    DECLARE_DATA_TUPLE_REF_VECTOR_MEMBER(uint32_t, city_region, 128); // 延迟加载变长字段
};
```
- **差异**：`DataTuple` 用于写操作（含 `set_xxx` 方法），`DataTupleRef` 用于读操作（含延迟加载逻辑）。

---

#### 二、核心方法实现

##### 1. **数据验证（is_valid_data_tuple）**
```cpp
bool PlanTableS::is_valid_data_tuple(const TDataTuple &tuple) {
    if (!tuple.is_set_plan_id()) return false; // 必须字段检查
    if (!tuple.is_set_region()) return false;
    if (!tuple.is_set_city_region()) return false;
    return true;
}
```
- **作用**：确保插入数据包含所有必填字段。

---

##### 2. **数据转换（make_inner_tuple & make_data_tuple）**
```cpp
void PlanTableS::make_inner_tuple(const TDataTuple &tuple, TInnerTuple* inner) {
    inner->plan_id = tuple.plan_id();   // 标量字段直接拷贝
    inner->region = tuple.region();
}

void PlanTableS::make_data_tuple(const PlanTableDataTupleRef& ref, TDataTuple* tuple) {
    tuple->set_plan_id(ref.plan_id());  // 从引用恢复数据
    tuple->set_region(ref.region());
    tuple->set_city_region(ref.city_region());
}
```
- **用途**：在内存结构（`InnerTuple`）和业务结构（`DataTuple`）间转换。

---

##### 3. **变长字段存储（insert_var_pools）**
```cpp
int PlanTable::insert_var_pools(TInnerTuple &inner, const TDataTuple &tuple) {
    // 将 city_region 写入内存池，记录虚拟地址
    int ret = tuple.city_region().save(*_p_var_pools[CITY_REGION], &inner._var_ptrs[CITY_REGION]);
    if (ret < 0) {
        CFATAL_LOG("Insert city_region failed.");
        return -1;
    }
    return 0;
}
```
- **流程**：调用 `VectorVar::save()` 将变长字段序列化到内存池，保存返回的虚拟地址。

---

##### 4. **内存池初始化（do_create_inner_pool）**
```cpp
int PlanTable::do_create_inner_pool(TInnerPool* p_inner_pool) {
    static const uint32_t slabs[] = {5, 8, 13, ..., 65537}; // 预定义内存块大小
    return p_inner_pool->create(slabs, sizeof(slabs)/sizeof(uint32_t), 128*1024);
}
```
- **作用**：配置 Slab 内存池，预分配不同尺寸的块，优化内存分配效率。

---

#### 三、关键调用展开

##### 1. **宏展开示例（DEFINE_DATA_TUPLE_MEMBER）**
```cpp
// 宏定义
#define DEFINE_DATA_TUPLE_MEMBER(type, field_name) \
    public: \
        type const& field_name() const { \
            if (!_is_set_##field_name) { CFATAL_LOG(...); } \
            return _##field_name; \
        } \
        void set_##field_name(type const& value) { \
            _##field_name = value; _is_set_##field_name = true; \
        } \
    private: \
        type _##field_name; \
        bool _is_set_##field_name;

// 展开后（以 plan_id 为例）
public:
    uint32_t const& plan_id() const {
        if (!_is_set_plan_id) { /* 错误处理 */ }
        return _plan_id;
    }
    void set_plan_id(uint32_t const& value) {
        _plan_id = value; _is_set_plan_id = true;
    }
private:
    uint32_t _plan_id;
    bool _is_set_plan_id;
```

##### 2. **变长字段访问（city_region）**
```cpp
// DataTupleRef 中访问 city_region
const VectorVar<uint32_t, 128>& city_region() const {
    // 从内存池加载数据
    _city_region.load(*_p_var_pools[PlanTable::CITY_REGION], _var_ptrs[CITY_REGION]);
    return _city_region;
}
```
- **延迟加载**：仅在访问 `city_region` 时从内存池读取数据，减少不必要的内存占用。

---

#### 四、使用示例

##### 1. **插入数据**
```cpp
PlanTableDataTuple tuple;
tuple.set_plan_id(1001);
tuple.set_region(0x12345678);
tuple.set_city_region(Vector<uint32_t, 128>{1, 2, 3});

PlanTable plan_table("plan");
plan_table.create(/* params */);
plan_table.insert(tuple); // 触发 insert_var_pools
```

##### 2. **查询数据**
```cpp
auto iter = plan_table.seek(1001); // 主键查找
if (!iter.is_null()) {
    const auto& ref = *iter;
    cout << "City Regions: " << ref.city_region() << endl; // 延迟加载
}
```

##### 3. **内存回收**
```cpp
plan_table.recycle(); // 调用 DataPool::recycle() 批量释放标记内存
```

---

#### 五、设计总结

- **内存管理**：通过 `DataPool` 和 `SlabMempool32` 实现高效内存分配，减少碎片。
- **数据抽象**：使用宏生成数据类，分离读写接口（`DataTuple` vs `DataTupleRef`）。
- **变长字段**：通过 `VectorVar` 和独立内存池管理，支持动态扩容。
- **性能优化**：延迟加载、批量回收机制提升高频操作性能。

# 变长字段

### 1. `VectorVar` 概述
`VectorVar` 是一个模板类，用于处理可变长度的向量数据，它继承自 `VarField` 类，使用 `VectorVarS` 作为其模式定义。下面我们将逐步展开其调用链的每一个环节。

### 2. 代码结构与关键组件

#### 2.1 `Buffer` 类
`Buffer` 类用于封装一段内存区域，提供了基本的访问和设置方法。
```cpp
namespace im {
namespace adtable {

class Buffer {
public:
    Buffer() : _start(NULL), _size(0) {}
    Buffer(const char* start, uint32_t size) : _start(start), _size(size) {}
    virtual ~Buffer() {}

    const char* start() const { return _start; }
    char* mutable_start() const { return const_cast<char*>(_start); }
    uint32_t size() const { return _size; }

    void set_buffer(const char* buf, uint32_t n_bytes) {
        _start = buf;
        _size = n_bytes;
    }

    template <typename TObj>
    void set_buffer(const TObj& obj) {
        _start = (const char*) &obj;
        _size = sizeof(obj);
    }

private:
    const char* _start;
    uint32_t    _size;
};

}  // namespace adtable
}  // namespace im
```
这个类主要用于存储和操作一段连续的内存数据，提供了设置和获取内存起始地址和大小的方法。

#### 2.2 `VarField` 类
`VarField` 是一个模板类，用于处理可变字段，它依赖于一个模式类 `TSchema` 来定义字段的头部、尾部和数据体的处理方式。
```cpp
namespace im {
namespace adtable {

template <typename TSchema>
class VarField {
public:
    typedef typename TSchema::THead THead;
    typedef typename TSchema::TTail TTail;

    VarField() : _p_body(NULL), _is_valid(false) {}
    virtual ~VarField() {}

    const THead& head() const { return _head; }
    const TTail& tail() const { return _tail; }
    const char* body() const { return _p_body; }
    bool is_valid() const { return _is_valid; }

    template <typename T>
    void load(const T& src_data) {
        TSchema::make_var_field(src_data, &_head, &_tail, &_p_body);
        _is_valid = TSchema::check(_head, _tail);
    }

    template <typename TPool>
    void load(const TPool& pool, const typename TPool::TVaddr& vaddr) {
        _is_valid = false;
        if (vaddr == TPool::null()) {
            _p_body = NULL;
            _is_valid = true;
            return;
        }

        const char* addr = (const char*) pool.read(vaddr);
        if (addr == NULL) {
            _p_body = NULL;
            return;
        }

        _head = ((const THead*) addr)[0];
        _p_body = addr + sizeof(THead);

        uint32_t body_size = TSchema::body_size(_head);
        _tail = ((const TTail*)(addr + sizeof(THead) + body_size))[0];

        _is_valid = TSchema::check(_head, _tail);
    }

    template <typename TPool>
    int save(TPool& pool, typename TPool::TVaddr* p_vaddr) const {
        if (p_vaddr != NULL) {
            *p_vaddr = TPool::null();
        }

        if (!is_valid()) {
            CFATAL_LOG("is_valid() test failed.");
            return -1;
        }

        uint32_t body_size = TSchema::body_size(_head);
        if (_p_body == NULL || body_size == 0) {
            return 0;
        }

        uint32_t malloc_size = sizeof(_head) + body_size + sizeof(_tail);

        typename TPool::TVaddr vaddr = pool.malloc(malloc_size);
        if (vaddr == pool.null()) {
            CFATAL_LOG(
                    "Failed to malloc %u bytes from DataPool %s.",
                    malloc_size, pool.name().c_str());
            return -1;
        }

        Buffer data[3];
        data[0].set_buffer(_head);
        data[1].set_buffer(_p_body, body_size);
        data[2].set_buffer(_tail);

        if (pool.writev(vaddr, data, 3) < 0) {
            CFATAL_LOG(
                    "Failed to write field into DataPool %s",
                    pool.name().c_str());
            return -1;
        }

        if (p_vaddr != NULL) {
            *p_vaddr = vaddr;
        }

        return 0;
    }

    void load(void* addr) {
        _is_valid = false;

        if (addr == NULL) {
            CFATAL_LOG("illegel load addr");
            _p_body = NULL;
            return;
        }

        _head = ((const THead*) addr)[0];
        _p_body = addr + sizeof(THead);

        uint32_t body_size = TSchema::body_size(_head);
        _tail = ((const TTail*)(addr + sizeof(THead) + body_size))[0];

        _is_valid = TSchema::check(_head, _tail);
    }

    int save(void* addr) const {
        if (addr == NULL) {
            CFATAL_LOG("illegel save address");
            return -1;
        }

        if (!is_valid()) {
            CFATAL_LOG("is_valid() test failed.");
            return -1;
        }

        uint32_t body_size = TSchema::body_size(_head);
        memcpy(addr, &_head, sizeof(_head));
        memcpy(addr + sizeof(_head), _p_body, body_size);
        memcpy(addr + sizeof(_head) + body_size, &_tail, sizeof(_tail));

        return 0;
    }

    uint32_t total_size() {
        uint32_t body_size = TSchema::body_size(_head);
        return sizeof(_head) + body_size + sizeof(_tail);
    }

protected:
    THead           _head;
    const char*     _p_body;
    TTail           _tail;
    bool            _is_valid;
};

}  // namespace adtable
}  // namespace im
```
这个类提供了加载和保存可变字段数据的功能，具体的处理逻辑由 `TSchema` 类定义。

#### 2.3 `VectorVarS` 类
`VectorVarS` 是一个模板类，作为 `VectorVar` 的模式类，定义了向量数据的头部、尾部和数据体的处理方式。
```cpp
namespace im {
namespace adtable {

template <typename T, int MAX_SIZE>
class VectorVarS {
public:
    typedef uint16_t    THead;
    typedef uint16_t    TTail;

    static void make_var_field(
            const Vector<T, MAX_SIZE>& src_data,
            THead* p_head,
            TTail* p_tail,
            const char** p_buf) {
        *p_head = static_cast<uint16_t>(src_data.size());
        *p_tail = static_cast<uint16_t>(src_data.size());

        Buffer buf = src_data.raw_buf();
        *p_buf = buf.start();
    };

    static bool check(const THead& head, const TTail& tail) {
        if (head > MAX_SIZE) {
            return false;   // over flow
        }

        return (head == tail);
    }

    static uint32_t body_size(const THead& head) {
        return head * sizeof(T);
    }
};

}  // namespace adtable
}  // namespace im
```
这个类定义了向量数据的头部和尾部的设置方法，以及数据体大小的计算方法，同时提供了数据有效性检查的方法。

#### 2.4 `VectorVar` 类
`VectorVar` 是一个模板类，继承自 `VarField<VectorVarS<T, MAX_SIZE>>`，用于处理可变长度的向量数据。
```cpp
namespace im {
namespace adtable {

template <typename T, int MAX_SIZE>
class VectorVar : public VarField<VectorVarS<T, MAX_SIZE> > {
public:
    typedef VarField<VectorVarS<T, MAX_SIZE> > TBase;

    uint32_t size() const {
        if (TBase::body() == NULL || !TBase::is_valid()) {
            return 0;
        }
        return TBase::head();
    }

    const T& operator[](uint32_t k) const {
        const char* raw_buf = TBase::body();
        uint16_t size = TBase::head();
        if (k >= size) {
            if (size > 0) {
                k = size - 1;
            } else {
                k = 0;
            }
        }

        const T* buf = (const T*) raw_buf;
        return buf[k];
    }

};

template <typename T, int MAX_SIZE, typename TOstream>
TOstream& operator<<(TOstream& os, const VectorVar<T, MAX_SIZE>& vec)
{
    if (vec.is_valid()) {
        uint32_t size = vec.size();
        os << size << "<";
        for (uint32_t i = 0; i < size; ++i) {
            os << vec[i] << "\1";
        }
        os << ">";
    } else {
        os << "NULL";
    }
    return os;
}

}  // namespace adtable
}  // namespace im
```
这个类提供了获取向量大小和访问向量元素的方法，同时重载了输出运算符，方便输出向量数据。

### 3. 调用链分析

#### 3.1 数据加载
当调用 `VectorVar` 的 `load` 方法时，会调用 `VarField` 的 `load` 方法，而 `VarField` 的 `load` 方法会调用 `VectorVarS` 的 `make_var_field` 方法来设置头部、尾部和数据体的指针，同时调用 `check` 方法来检查数据的有效性。
```cpp
// VectorVar 的 load 方法调用示例
VectorVar<int, 100> vec;
Vector<int, 100> src_data;
// 初始化 src_data...
vec.load(src_data);
// 调用链：VectorVar::load -> VarField::load -> VectorVarS::make_var_field -> VectorVarS::check
```

#### 3.2 数据保存
当调用 `VectorVar` 的 `save` 方法时，会调用 `VarField` 的 `save` 方法，`VarField` 的 `save` 方法会计算数据体的大小，分配内存，将头部、数据体和尾部数据写入内存池，并返回分配的内存地址。
```cpp
// VectorVar 的 save 方法调用示例
VectorVar<int, 100> vec;
// 初始化 vec...
DataPool pool;
typename DataPool::TVaddr vaddr;
vec.save(pool, &vaddr);
// 调用链：VectorVar::save -> VarField::save -> VectorVarS::body_size
```

#### 3.3 数据访问
当调用 `VectorVar` 的 `size` 方法时，会调用 `VarField` 的 `body` 和 `head` 方法来获取数据体指针和头部信息，从而计算向量的大小。当调用 `operator[]` 方法时，会根据数据体指针和头部信息来访问向量的元素。
```cpp
// VectorVar 的 size 和 operator[] 方法调用示例
VectorVar<int, 100> vec;
// 初始化 vec...
uint32_t size = vec.size();
int element = vec[0];
// 调用链：VectorVar::size -> VarField::body -> VarField::head
// 调用链：VectorVar::operator[] -> VarField::body -> VarField::head
```

### 4. 总结
`VectorVar` 类通过继承 `VarField` 类，利用 `VectorVarS` 类作为模式定义，实现了可变长度向量数据的加载、保存和访问功能。其调用链涉及到多个类的方法调用，通过模板类的设计，使得代码具有良好的可扩展性和复用性。

以下是针对 `VectorVar` 类的源码级分析及调用链展开：

---

### **1. 类结构与继承关系**
```cpp
template <typename T, int MAX_SIZE>
class VectorVar : public VarField<VectorVarS<T, MAX_SIZE> > {
public:
    typedef VarField<VectorVarS<T, MAX_SIZE> > TBase;
    // ...
};
```
- **继承自** `VarField<VectorVarS<T, MAX_SIZE>>`，使用 `VectorVarS` 作为 `TSchema` 模板参数。
- **目的**：通过模板特化，将 `Vector<T, MAX_SIZE>` 序列化为可变长度字段（`VarField`）。

---

### **2. `VectorVarS` 模板结构**
```cpp
template <typename T, int MAX_SIZE>
class VectorVarS {
public:
    typedef uint16_t THead;
    typedef uint16_t TTail;

    static void make_var_field(...);
    static bool check(...);
    static uint32_t body_size(...);
};
```
- **职责**：定义 `VarField` 所需的元数据操作规则。
- **关键方法**：
  - **`make_var_field`**：将 `Vector` 转换为 `VarField` 需要的头、尾和缓冲区。
  - **`check`**：验证头尾一致性。
  - **`body_size`**：根据头部计算数据体大小。

---

### **3. 调用链解析**

#### **3.1 数据加载（`load`）**
```cpp
VectorVar<T, MAX_SIZE> vec;
vec.load(src_vector);
```
1. **调用 `VarField::load`**：
    ```cpp
    template <typename T>
    void load(const T& src_data) {
        TSchema::make_var_field(src_data, &_head, &_tail, &_p_body);
        _is_valid = TSchema::check(_head, _tail);
    }
    ```
2. **`VectorVarS::make_var_field`**：
    ```cpp
    static void make_var_field(
        const Vector<T, MAX_SIZE>& src_data,
        THead* p_head,
        TTail* p_tail,
        const char** p_buf) 
    {
        *p_head = src_data.size();     // 头记录元素数量
        *p_tail = src_data.size();     // 尾同样记录数量（校验用）
        *p_buf = src_data.raw_buf();   // 获取原始数据指针
    }
    ```
3. **`VectorVarS::check`**：
    ```cpp
    static bool check(const THead& head, const TTail& tail) {
        return (head == tail && head <= MAX_SIZE); // 校验头尾一致且未越界
    }
    ```

---

#### **3.2 数据保存（`save`）**
```cpp
vec.save(pool, &vaddr);
```
1. **调用 `VarField::save`**：
    ```cpp
    int save(TPool& pool, typename TPool::TVaddr* p_vaddr) const {
        // 分配内存并写入头、体、尾
        uint32_t malloc_size = sizeof(_head) + body_size + sizeof(_tail);
        typename TPool::TVaddr vaddr = pool.malloc(malloc_size);

        Buffer data[3];
        data[0].set_buffer(_head);              // 头
        data[1].set_buffer(_p_body, body_size); // 数据体
        data[2].set_buffer(_tail);              // 尾

        pool.writev(vaddr, data, 3);            // 批量写入
    }
    ```
2. **`Buffer::set_buffer`**：
    ```cpp
    template <typename TObj>
    void set_buffer(const TObj& obj) {
        _start = (const char*)&obj;  // 对象地址转为字符指针
        _size = sizeof(obj);         // 对象大小
    }
    ```

---

#### **3.3 数据访问（`operator[]`）**
```cpp
const T& element = vec[2];
```
1. **调用 `VectorVar::operator[]`**：
    ```cpp
    const T& operator[](uint32_t k) const {
        const char* raw_buf = TBase::body();  // 获取数据体指针
        uint16_t size = TBase::head();        // 获取实际元素数量
        const T* buf = (const T*) raw_buf;    // 转换为元素类型指针
        return buf[k];                        // 返回第k个元素
    }
    ```
2. **`VarField::body()`**：
    ```cpp
    const char* body() const { return _p_body; } // 返回数据体指针
    ```

---

### **4. 内存布局**
序列化后的内存结构如下：
```
| THead (2B) | T data[...] (body_size B) | TTail (2B) |
```
- **Head**：记录元素数量（`uint16_t`）。
- **Body**：实际数据体，大小为 `head * sizeof(T)`。
- **Tail**：与 Head 相同，用于校验数据完整性。

---

### **5. 关键设计点**
1. **可变长度字段**：通过 `VarField` 统一管理头、体、尾，支持动态调整数据大小。
2. **类型安全**：通过模板参数 `T` 和 `MAX_SIZE` 约束数据类型和最大容量。
3. **校验机制**：头尾双重校验防止数据损坏。
4. **内存池集成**：通过 `TPool` 抽象实现内存分配与释放的分离。

---

### **6. 示例调用链**
```
用户代码：vec.load(src_vector)
    → VarField::load() 
        → VectorVarS::make_var_field() 提取头、尾、数据体
        → VectorVarS::check() 校验头尾
用户代码：vec.save(pool, &vaddr)
    → VarField::save()
        → VectorVarS::body_size() 计算数据体大小
        → Buffer::set_buffer() 封装头、体、尾为 Buffer 数组
        → TPool::writev() 批量写入内存池
用户代码：vec[2]
    → VectorVar::operator[] 
        → VarField::body() 获取数据体指针
        → 强制转换为 T* 后访问元素
```

通过此设计，`VectorVar` 实现了高效的类型安全可变长度向量存储。

### 源码解析：VectorVar（变长向量存储）

---

#### 一、核心组件与设计思想

**VectorVar** 是一个 **支持变长存储的向量模板类**，基于 `VarField` 实现数据的内存序列化与反序列化。其核心设计目标：
- **动态长度**：允许存储最多 `MAX_SIZE` 个元素，实际长度在运行时确定。
- **内存安全**：通过 `VarField` 确保数据头尾校验，防止内存越界。
- **高效存储**：使用连续内存块存储数据，减少碎片。

---

#### 二、关键源码解析

##### 1. **VectorVarS（序列化策略）**
```cpp
template <typename T, int MAX_SIZE>
class VectorVarS {
public:
    typedef uint16_t THead; // 头标记（存储元素数量）
    typedef uint16_t TTail; // 尾标记（校验用）

    static void make_var_field(
        const Vector<T, MAX_SIZE>& src_data,
        THead* p_head, TTail* p_tail, 
        const char** p_buf) 
    {
        *p_head = src_data.size(); // 头记录元素数量
        *p_tail = src_data.size(); // 尾与头相同（校验）
        *p_buf = src_data.raw_buf().start(); // 数据指针
    }

    static bool check(THead head, TTail tail) {
        return (head <= MAX_SIZE) && (head == tail); // 校验数据完整性
    }

    static uint32_t body_size(THead head) {
        return head * sizeof(T); // 计算数据部分大小
    }
};
```
- **作用**：定义数据序列化规则及校验逻辑。

---

##### 2. **VectorVar（主类）**
```cpp
template <typename T, int MAX_SIZE>
class VectorVar : public VarField<VectorVarS<T, MAX_SIZE>> {
public:
    typedef VarField<VectorVarS<T, MAX_SIZE>> TBase;

    uint32_t size() const {
        if (!TBase::is_valid()) return 0;
        return TBase::head(); // 头标记即为元素数量
    }

    const T& operator[](uint32_t idx) const {
        const char* buf = TBase::body(); // 数据起始地址
        uint16_t size = TBase::head();
        idx = (idx >= size) ? (size > 0 ? size-1 : 0) : idx; // 边界保护
        return reinterpret_cast<const T*>(buf)[idx];
    }
};
```
- **继承**：复用 `VarField` 的加载/保存逻辑。
- **功能**：提供向量接口（大小、元素访问）。

---

##### 3. **VarField（基础存储类）**
```cpp
template <typename TSchema>
class VarField {
    TSchema::THead _head;   // 头标记
    const char* _p_body;    // 数据指针
    TSchema::TTail _tail;   // 尾标记
    bool _is_valid;         // 校验结果

public:
    // 从数据源加载（如Vector<T>）
    template <typename T>
    void load(const T& src_data) {
        TSchema::make_var_field(src_data, &_head, &_tail, &_p_body);
        _is_valid = TSchema::check(_head, _tail);
    }

    // 从内存池加载
    template <typename TPool>
    void load(const TPool& pool, const typename TPool::TVaddr& vaddr) {
        const char* addr = (const char*) pool.read(vaddr);
        _head = *reinterpret_cast<const THead*>(addr);
        _p_body = addr + sizeof(THead);
        _tail = *reinterpret_cast<const TTail*>(addr + sizeof(THead) + TSchema::body_size(_head));
        _is_valid = TSchema::check(_head, _tail);
    }

    // 保存到内存池
    template <typename TPool>
    int save(TPool& pool, typename TPool::TVaddr* p_vaddr) const {
        uint32_t total_size = sizeof(THead) + TSchema::body_size(_head) + sizeof(TTail);
        typename TPool::TVaddr vaddr = pool.malloc(total_size);

        Buffer data[3] = {
            Buffer(&_head, sizeof(THead)),
            Buffer(_p_body, TSchema::body_size(_head)),
            Buffer(&_tail, sizeof(TTail))
        };

        pool.writev(vaddr, data, 3); // 批量写入
        *p_vaddr = vaddr;
        return 0;
    }
};
```
- **核心方法**：实现数据与内存池的交互。

---

#### 三、调用链示例（插入数据到内存池）

##### 1. **初始化 VectorVar**
```cpp
Vector<uint32_t, 128> vec;
vec.push_back(1);
vec.push_back(2);
VectorVar<uint32_t, 128> var_vec;
var_vec.load(vec); // 调用 VarField::load(const T&)
```
- **流程**：
  1. `VectorVarS::make_var_field` 提取 `vec` 的大小和数据指针。
  2. 设置 `_head` 和 `_tail` 为 2（元素数量）。
  3. `_p_body` 指向 `vec` 的原始数据地址。

---

##### 2. **保存到内存池**
```cpp
DataPool<SlabMempool32> pool;
pool.create(...);
typename DataPool::TVaddr vaddr;
var_vec.save(pool, &vaddr); // 调用 VarField::save()
```
- **流程**：
  1. 计算总大小：`sizeof(uint16_t) + 2*sizeof(uint32_t) + sizeof(uint16_t) = 2 + 8 + 2 = 12`。
  2. 从 `pool` 分配 12 字节内存，获得 `vaddr`。
  3. 分三部分写入内存池：
     - 头标记（2字节）：0x0002。
     - 数据体（8字节）：0x00000001, 0x00000002。
     - 尾标记（2字节）：0x0002。

---

##### 3. **从内存池加载**
```cpp
VectorVar<uint32_t, 128> loaded_vec;
loaded_vec.load(pool, vaddr); // 调用 VarField::load(TPool&)
uint32_t size = loaded_vec.size(); // 2
uint32_t elem = loaded_vec[1];     // 2
```
- **流程**：
  1. 从 `vaddr` 读取内存块。
  2. 解析头标记 `_head=2`，尾标记 `_tail=2`。
  3. 校验 `_head == _tail` 且 `_head <= 128`。
  4. `_p_body` 指向数据部分，通过 `operator[]` 访问元素。

---

#### 四、关键设计点

1. **头尾校验**  
   - 写入时头尾标记相同，加载时校验防止数据损坏。

2. **内存布局**  
   - 连续存储结构：`[Head][Data...][Tail]`，提升访问效率。

3. **变长支持**  
   - 实际存储长度由 `_head` 决定，最大不超过 `MAX_SIZE`。

4. **与内存池交互**  
   - 使用 `DataPool` 分配内存，支持延迟回收（通过 `TVaddr` 虚拟地址）。

---

#### 五、使用示例

```cpp
// 创建原始数据
Vector<uint32_t, 128> cities;
cities.push_back(1001);
cities.push_back(1002);

// 封装为 VectorVar
VectorVar<uint32_t, 128> var_cities;
var_cities.load(cities);

// 保存到内存池
DataPool<SlabMempool32> pool;
pool.create(...);
typename DataPool::TVaddr addr;
var_cities.save(pool, &addr);

// 从池中加载
VectorVar<uint32_t, 128> loaded_cities;
loaded_cities.load(pool, addr);
cout << loaded_cities.size() << endl; // 输出 2
cout << loaded_cities[1] << endl;     // 输出 1002
```

---

#### 六、性能与安全

- **性能优化**：连续内存访问、批量写入（`writev`）减少I/O次数。
- **安全机制**：边界检查（`operator[]`）、头尾校验防止数据错乱。
- **内存控制**：通过 `MAX_SIZE` 限制最大内存使用，避免溢出。
