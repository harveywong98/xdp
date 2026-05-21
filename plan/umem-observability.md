# UMEM 观测：帧大小使用率 + 帧数量饱和度

## Context

XDP Socket 的 UMEM 有两个关键参数：FrameSize 和 NumFrames。设置不合理时：
- FrameSize 太大 → 内存浪费（小包占大帧）
- FrameSize 太小 → 包被截断，静默丢数据
- NumFrames 太少 → 帧耗尽，Fill ring 饥饿，收包停滞

目标是加可观测性，让调用方能判断这两个参数是否合理。

## 改动文件

- `xdp.go` — 唯一需要改的文件

## 实现方案

### 1. 帧大小使用率：5 秒滑动窗口 ring buffer

**数据结构**：`[5]frameSizeSlot`，每个 slot 对应 1 秒，存该秒内的分布统计。

```go
type frameSizeSlot struct {
    buckets        [4]uint64  // [i]: 利用率落在 [i*25%, (i+1)*25%) 的帧数
    truncatedCount uint64     // len >= frameSize，可能被截断
    maxLen         uint32
    totalSampled   uint64
    ts             int64      // slot 对应的 Unix 秒时间戳
}
```

`Socket` 结构体加字段：
```go
frameSizeSlots [5]frameSizeSlot
```

**更新**：`Receive()` 返回前，对每个 desc 调 `recordFrameLen(desc.Len, now)`。`now` 在循环外取一次 `time.Now().Unix()`，不是每帧都调。写入时用 `now % 5` 取槽位，如果槽位 ts 不等于 now 则清零重用。

**读取**：`FrameSizeStats()` 取当前秒，遍历 5 个 slot，只聚合 `ts >= now-4` 的有效 slot，合并 buckets（相加）、MaxLen（取 max）、TruncatedCount / TotalSampled（相加）。

**体现指标**：
- `Buckets[0]`（0-25% 利用率）占大头 → FrameSize 设太大
- `TruncatedCount > 0` → FrameSize 设太小，包被截断
- `MaxLen` → FrameSize 的合理下限参考值

### 2. 帧数量饱和度（扩展现有 Stats()）

只关心接收方向，扫描 `freeRXDescs`（UMEM 帧池，非 ring 槽位）：

```go
// Stats 结构体新增字段
TotalFrames  int  // options.NumFrames，UMEM 总帧数
UsedRXFrames int  // freeRXDescs 中 false 的数量
```

在 `Stats()` 末尾填充：
```go
stats.TotalFrames = xsk.options.NumFrames
for _, free := range xsk.freeRXDescs {
    if !free { stats.UsedRXFrames++ }
}
```

注意：`TotalFrames` 是 UMEM 帧池大小（`NumFrames`），不是 `FillRingNumDescs` 也不是 `RxRingNumDescs`。

**体现指标**：`UsedRXFrames / TotalFrames` 接近 1.0 → NumFrames 不足。

### 公开类型

```go
type FrameSizeStats struct {
    Buckets        [4]uint64  // [i]: 帧利用率落在 [i*25%, (i+1)*25%) 的帧数，i=3 含 100%
    TruncatedCount uint64     // Len >= FrameSize，包可能被截断
    MaxLen         uint32     // 窗口内最大包长，可作为 FrameSize 下限参考
    TotalSampled   uint64     // 窗口内总采样帧数，0 表示窗口内无数据
}
```

## 关键决策

- Ring buffer slot 宽度 1s，共 5 个 slot，覆盖最近 5 秒
- `recordFrameLen` 中 `time.Now()` 在 `Receive()` 里每次调用取一次，不是每帧取一次
- 不起后台 goroutine
- 只统计接收方向，不动 `freeTXDescs`
- 单线程假设（与现有 Fill/Receive 一致）

## 验证

1. 设 FrameSize=1000，收小包（如 64 字节），5 秒后调 `FrameSizeStats()`，Buckets[0] 有计数，MaxLen ≈ 64
2. 调 `Stats()`，UsedRXFrames / TotalFrames 反映实际 UMEM 占用
3. 截断场景：FrameSize 设小于实际包长，确认 TruncatedCount 增加
