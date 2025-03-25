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
