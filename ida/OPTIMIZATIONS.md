# `seh_parse.py` 性能优化笔记

**日期**: 2026-04-26
**目标**: segment 模式下从 ~40h 量级缩到小时级
**影响范围**: **仅 `seh_parse.py`**。其余 3 个 script (`patch_jump.py` / `seh_clear.py` / `constant_propagation.py`) 是单次交互式调用，不在长跑路径上，未改动。

---

## 4 项改动

### #1. `ExecutionContext` 类属性 → 实例属性 (致命 bug，预期最大收益)

位置: `seh_parse.py:313`

之前:
```python
class ExecutionContext:
    blocks = []          # ← 类属性
    patches = []         # ← 类属性
    branches_pending = []
    def __init__(self, ...):
        self.range_start = range_start
        ...
```

`__init__` 没给 `self.blocks` 等赋值，`self.blocks.append(...)` 走属性查找链命中**类列表**——所有 `ExecutionContext` 实例共享。

后果: 每处理一个函数，`patches` 累积；末尾 `apply_patches` 把之前所有函数的 patch **重新写一遍** IDB（每次都是 `del_items` + `patch_byte` + `patch_dword` + `create_insn` + `add_hidden_range`）。复杂度从 O(N) 涨到 O(N²)，N = 段内函数数。

修复: 移到 `__init__` 实例属性，加注释说明。

---

### #2. `FakeBranching.find_sequence_end` 加 memoization

位置:
- `seh_parse.py:14` — module-level `_find_sequence_end_cache = {}`
- `seh_parse.py:223` — wrapper (cache 检查)
- `seh_parse.py:234` — `_find_sequence_end_impl` (原 body 重命名)
- `seh_parse.py:440-443` — `simulate_all` 入口 `cache.clear()`

`simulate_one` 对每个非 nop 指令都跑一次 `FakeBranching().find_sequence_end(ea)`，内部 `process_conditional_jump` 还会 fork 两条路径递归。同地址被多路径反复 walk，无缓存。

正确性: 该函数对 `(ea, flags_known, flags_value, have_jump)` 是**纯函数**——只通过 `decode_insn` 读 IDB bytes，sim 阶段 bytes 不变（patches 推迟到 `apply_patches`）。

cache 生命周期 = 单次 `simulate_all`，入口 `cache.clear()`。下一函数 `simulate_all` 调用之前已经 `apply_patches` 改了 bytes，必须清。

---

### #3. `find_unwind_info` 线性扫描 → 二分搜索

位置: `seh_parse.py:94`

PE exception directory 条目按 `start_rva` 排序（PE 规范），原 TODO 注释已经标了。

**注意**: 这条仅 `process_single_function` 路径走；`process_segment` 直接遍历，不调用 `find_unwind_info`。所以对 40h segment-mode 不是关键，但顺手做掉。

---

### #4. 整体包 `ida_auto.enable_auto(False)`

位置: `seh_parse.py:557`

每次 `del_items` / `create_insn` / `patch_byte` 都触发 IDA 对该范围排队 auto-analysis。几十 K 函数下来开销不小。

```python
ida_auto.enable_auto(False)
try:
    if process_entire_segment:
        process_segment(idaapi.get_screen_ea())
    else:
        process_single_function(idaapi.get_screen_ea())
finally:
    ida_auto.enable_auto(True)
    ida_auto.auto_wait()
```

`auto_wait()` 在末尾把所有延后的分析做一次完成，最终 IDB 状态等价。

---

## 可靠性论证

| 改动 | 行为是否等价 | 论证 |
|---|---|---|
| #1 | 单函数行为完全不变 | 仅消除跨函数泄露的"幽灵状态"。每个 `ExecutionContext` 独立干净 |
| #2 | 等价 | 函数纯读 IDB bytes，sim 阶段不写。cache key 包含全部 4 个输入参数。返回值是 `end_ea` int 或 `None`，无副作用被观察（已验证 3 个调用点都不读 self post-call 状态） |
| #3 | 等价 | 二分搜索同结果。chain-walk 后续逻辑未变 |
| #4 | 最终 IDB 等价 | `auto_wait()` flush 全部事件。中间状态可见性不影响 batch 处理结果 |

### 顺带修的正确性问题

#1 不仅是性能 bug——**不重启 IDA 重跑 script** 时，旧 `blocks/patches` 还在类列表里，可能触发 `apply_patches` 内的 `raise Exception(f'Failed to patch ...')`，因为 patch 的前置 block 检查会被脏数据搞坏。修了之后每次都是干净状态。

---

## 测试建议

### Step 1: 冒烟测试（必做）

1. 把整个 `C:\User\sc2rtwp` 目录拷过去
2. **先备份 .idb**
3. 在 IDA 里打开目标 binary
4. 确认 `seh_parse.py` 顶部 `process_entire_segment = False` （默认值）
5. 选一个已知典型函数（之前跑过的），光标放上面，运行 `seh_parse.py`
6. 对比 patch bytes / hidden ranges / func 边界 / SEH xref 跟之前一致

如有差异，**先别开 segment 模式**，把具体差异点记下来。

### Step 2: 完整 segment 跑

1. 再备份一次 idb
2. 改 `process_entire_segment = True`
3. 计时跑

如果还非常慢，可以临时把 `log_level` 调到 0 看到底卡在哪一步。

---

## 单条回滚（做对照实验时）

4 项改动彼此独立，可以单独还原任意一条：

- 还原 #1: 把 `self.blocks = []` 等 3 行删掉，移回类层 `blocks = []`
- 还原 #2: `find_sequence_end` 改回直接是原 body（即把 `_find_sequence_end_impl` 改回叫 `find_sequence_end`、删 wrapper 和 module-level cache）
- 还原 #3: 二分循环还原成 `for i in range(count):` 线性扫描
- 还原 #4: 删 `ida_auto.enable_auto/auto_wait` 那段 try/finally，留裸调用

预期单条收益排序:
```
#1 (O(N²) 消除)  >>  #4 (auto-analyzer)  >  #2 (memoize)  >>  #3 (binary search, segment 模式不走)
```

---

## 当前 .py 文件关键 line 号速查

| 项 | line |
|---|---|
| `_find_sequence_end_cache` 声明 | 14 |
| `find_unwind_info` (二分版) | 94 |
| `class FakeBranching` | 141 |
| `find_sequence_end` wrapper | 223 |
| `_find_sequence_end_impl` (原 body) | 234 |
| `class ExecutionContext` | 313 |
| `simulate_all` (含 `cache.clear()`) | 440 |
| 入口 `ida_auto.enable_auto(False)` | 557 |
