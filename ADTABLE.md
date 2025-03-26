# MemDataTable存储
MemDataTable的底层存储结构是一个结合哈希表快速索引和Slab内存池高效管理的复合体系，下面从多个维度详细分析其实现细节：

---

### **1. 哈希表结构**
#### **1.1 哈希表实现**
- **模板类HashMap**：MemDataTable通过`THashMap`（即`HashMap<HashMapS>`）实现哈希表，其中`HashMapS`定义了键类型（`TPrimaryKey`）、值类型（`TInnerTuple`）及内存池类型（`DataPool<SlabMempool32>`）。
- **哈希函数**：使用`TPrimaryKey`的`hash_value()`方法生成哈希值，决定数据在哈希桶中的位置。
- **冲突解决**：采用链地址法处理冲突，每个哈希桶维护一个链表，链表节点存储在Slab内存池中。

#### **1.2 哈希表节点**
- **节点结构**：每个哈希节点包含：
  - **键（Key）**：主键（如广告ID）的哈希值。
  - **值（Value）**：数据元组（`TInnerTuple`）的虚拟地址（`TVaddr`），指向内存池中的具体数据。
  - **链表指针**：指向同一哈希桶的下一个节点。
- **节点内存管理**：节点内存通过`SlabMempool32`分配，节点大小固定为`THashMap::node_size()`（由模板参数决定）。

#### **1.3 哈希表创建**
```cpp
// 创建哈希表
THashMap* _p_dict = new THashMap(name);
_p_dict->create(row_num, hash_ratio, _p_hash_pool);
```
- **参数**：`row_num`指定初始哈希桶数量，`hash_ratio`控制哈希表扩容阈值（如负载因子）。
- **内存池绑定**：哈希表节点内存来自`_p_hash_pool`（`DataPool<SlabMempool32>`实例）。

---

### **2. Slab内存池（SlabMempool32）**
#### **2.1 内存池结构**
- **Slab划分**：内存池按不同大小的Slab分类（如4B、8B等），每个Slab划分为多个等长的item。
- **分配策略**：
  - **malloc**：根据请求大小选择合适Slab，从空闲链表分配item。
  - **free**：将item标记为空闲，插入对应Slab的空闲链表。
- **虚拟地址（TVaddr）**：32位地址，高12位表示Slab块索引，低20位表示item偏移。

#### **2.2 关键数据结构**
```cpp
struct mem_block_t {
    char*       start;          // 内存块起始地址
    uint32_t    item_size;      // 每个item的大小
    uint32_t    free_item;      // 空闲item数量
    uint32_t    max_item_num;   // 总item数
};

struct mem_slab_t {
    std::vector<uint32_t> block_list; // 属于该Slab的内存块列表
    uint32_t free_list;               // 空闲item链表头
};
```

#### **2.3 内存分配流程**
1. **计算Slab索引**：根据请求大小找到最小满足的Slab。
2. **分配item**：
   - 若当前Slab有空闲item，直接分配。
   - 若无空闲，创建新内存块并分割为多个item。
3. **返回TVaddr**：组合块索引和偏移生成虚拟地址。

---

### **3. 不定长字段存储（VarField与VectorVar）**
#### **3.1 VarField设计**
- **三段式结构**：每个不定长字段分为头（Header）、体（Body）、尾（Tail），确保数据完整性。
  ```cpp
  struct VarField {
      THead           _head;     // 头部（如数据长度）
      const char*     _p_body;   // 数据体指针
      TTail           _tail;     // 尾部（校验信息）
  };
  ```
- **校验机制**：通过`check(head, tail)`验证头尾一致性（如长度是否匹配）。

#### **3.2 VectorVar实现**
- **动态数组存储**：继承自`VarField`，用于存储可变长数组。
  ```cpp
  template <typename T, int MAX_SIZE>
  class VectorVar : public VarField<VectorVarS<T, MAX_SIZE>> { ... };
  ```
- **序列化示例**：
  ```cpp
  // 数据写入内存池
  Buffer data[3] = {header_buf, body_buf, tail_buf};
  pool.writev(vaddr, data, 3); // 连续写入三段数据
  ```

#### **3.3 内存分配与释放**
- **分配流程**：
  1. 计算总大小：`sizeof(THead) + body_size + sizeof(TTail)`.
  2. 从`DataPool`分配足够内存，返回`TVaddr`.
  3. 将头、体、尾连续写入内存池。
- **释放流程**：调用`DataPool::free(vaddr)`，内存池标记item为空闲。

---

### **4. 数据操作流程**
#### **4.1 插入数据**
1. **构造数据元组**：将外部数据`TDataTuple`转换为内部格式`TInnerTuple`。
2. **分配不定长字段内存**：
   ```cpp
   // 示例：插入VectorVar字段
   VectorVar<T, MAX_SIZE> vec_var;
   vec_var.load(src_data);                  // 加载数据
   vec_var.save(*_p_var_pools[i], &vaddr);  // 保存到内存池，获取vaddr
   inner_tuple.set_var_vaddr(i, vaddr);      // 记录vaddr到元组
   ```
3. **插入哈希表**：将元组地址存入哈希表节点。

#### **4.2 查询数据**
1. **哈希查找**：通过主键哈希值定位哈希桶。
2. **链表遍历**：遍历链表，比较主键找到目标节点。
3. **获取数据地址**：从节点中读取`TVaddr`，转换为内存地址访问数据。

#### **4.3 删除数据**
1. **触发回调**：执行`trigger_remove`通知相关组件。
2. **释放内存**：释放不定长字段占用的内存池item。
   ```cpp
   for (size_t pool_idx = 0; pool_idx < TSchema::VAR_POOL_NUM; ++pool_idx) {
       _p_var_pools[pool_idx]->free(data._var_ptr(pool_idx));
   }
   ```
3. **移除哈希节点**：从哈希表中删除对应节点。

---

### **5. 内存管理优化**
- **Slab预分配**：初始化时创建多个Slab块，减少运行时内存碎片。
- **空闲链表**：每个Slab维护空闲item链表，加速分配。
- **监控告警**：当内存块使用率超过85%时触发告警，防止内存耗尽。

---

### **6. 监控与调试**
- **内存统计**：`monitor()`方法收集各内存池的使用量、节点数、哈希表负载等信息。
- **日志输出**：`print_log()`格式化输出关键指标，如：
  ```
  [WARNING][MyTable][NODE_NUM:1000][TOTAL_MEM:10MB][FIX_MEM:5MB][VAR_MEM:5MB]...
  ```

---

### **总结**
MemDataTable通过哈希表实现快速主键查找，结合Slab内存池高效管理固定大小内存块，利用VarField机制处理不定长字段的三段式存储，确保数据完整性和存取效率。其设计充分考虑了内存利用率、访问速度及系统可维护性，适用于高并发、低延迟的广告检索场景。
