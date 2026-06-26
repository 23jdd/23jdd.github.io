+++
date = '2026-06-26T14:35:00+08:00'
draft = false
title = '深入理解 HyperLogLog：从抛硬币到UV统计'
+++


## 引言

假设你有一个电商网站，老板问你："今天有多少独立访客？"

简单。把所有用户ID存到 HashSet 里，最后数一下 size。但如果每天有上亿访客呢？HashSet 可能要吃掉几个G的内存。

有没有办法用**极小的内存**估算出**近似去重数量**？HyperLogLog 就是答案——只需 12KB 内存，误差控制在 1% 以内。

---

## 1. 从抛硬币说起

### 一个有趣的实验

我背着你抛一枚公平硬币，直到出现正面为止。我告诉你："最长一次连续抛出了 5 次反面。"

你猜我一共抛了多少轮？

- 如果只抛了 10 轮，遇到连续 5 次反面的概率很小
- 如果我抛了 10000 轮，几乎肯定会遇到连续 5 次反面

**核心直觉**：遇到"罕见事件"的难度，间接反映了试验总次数。

连续 k 次反面的概率是 1/2^k。如果我最多遇到了连续 k 次反面，那么我大概抛了 2^k 轮。

### 数学化这个想法

把每个元素哈希成一个 64 位的二进制串，就像抛了 64 次硬币：

```
hash("user_123") = 0100 1101 ... 0010 0001
```

从最低位开始数，第一个 1 出现的位置（也就是前导零的个数），就是"反面的次数"。

如果所有元素中，最长前导零是 20，那么大概有 2^20 ≈ 100 万个不同元素。

这就是 LogLog 算法的核心思想。

---

## 2. 分桶：减少方差

只用一个值估计，太不稳定了。万一某个元素碰巧哈希出了超长前导零，估算就会严重偏高。

**解决方案**：分桶。

把哈希值的前几位用来决定放入哪个桶，剩余位用来算前导零。每个桶独立记录自己见过的"最长前导零"，最后取平均。

HyperLogLog 使用 16384 个桶（2^14），这样单个桶的运气成分被大大稀释。

```
哈希值: [前14位: 桶索引] [后50位: 计算前导零]
└── 0~16383 ──┘ └── 取值 1~51 ──┘
```

---

## 3. 调和平均：抑制极端值

现在有 16384 个桶，每个桶都有一个"最长前导零"的值。怎么把它们合成一个总数？

**算术平均**？不行。假设桶值为 [2, 3, 100]，100 会把平均值拉得很高 → 2^平均 ≈ 2^35，严重高估。

**调和平均**天然偏向小值，能有效抑制离群桶的影响：

```
H = m / (1/2^R₁ + 1/2^R₂ + ... + 1/2^Rₘ)
```

HyperLogLog 的"Hyper"就是指：相比 LogLog 用的几何平均，换成了调和平均，精度更高。

### 完整公式

```
E = α · m² / Σ(2^(-Rᵢ))
```

其中 α 是校正常数。为什么需要 α？因为即使用了调和平均，公式本身还是有一点系统性偏差（会高估约 20-40%）。α 是通过数学推导和实验测出来的修正因子。

| 桶数 m | α 值 |
|--------|------|
| 16 | 0.673 |
| 32 | 0.697 |
| 64 | 0.709 |
| 128+ | 0.7213 / (1 + 1.079/m) |

对于 16384 个桶，α ≈ 0.7213。

---

## 4. 修正：边界情况

### 小基数修正

当元素很少时（< 40000），大量桶是空的。调和平均会低估，此时直接用"空桶比例"来估算更准：

```
E = m · ln(m / 空桶数)
```

### 大基数修正

当估算值接近 2^32 时，哈希碰撞概率增大，需要特殊公式校正。

---

## 5. Merge：并集的神奇性质

HLL 的一个强大特性是**可以合并**。

场景：上午的 UV 和下午的 UV，想算全天 UV。

上午 HLL：桶 0 最长前导零 = 3
下午 HLL：桶 0 最长前导零 = 4

全天桶 0 取 max(3, 4) = 4。因为：

```
max(上午所有元素的极值, 下午所有元素的极值) = 全天所有元素的极值
```

数学上等价于 `max(max(A), max(B)) = max(A ∪ B)`。

这就是为什么 HLL 可以轻松实现"小时合并为天、天合并为周"的 UV 聚合，内存始终只有 12KB。

---

## 6. Go 实现

完整可运行的 HyperLogLog Go 实现：

```go
package hll

import (
	"hash/fnv"
	"math"
	"math/bits"
)

const (
	precision  = 14
	numBuckets = 1 << precision // 16384
)

type HyperLogLog struct {
	registers [numBuckets]uint8
}

func New() *HyperLogLog {
	return &HyperLogLog{}
}

// Add 添加元素
func (h *HyperLogLog) Add(data []byte) {
	hash := fnvHash64(data)

	// 前14位决定桶索引
	bucket := hash >> (64 - precision)

	// 后50位计算前导零
	remaining := hash << precision
	leadingZeros := uint8(1)
	if remaining != 0 {
		leadingZeros = uint8(bits.LeadingZeros64(remaining) + 1)
	} else {
		leadingZeros = uint8(64 - precision + 1)
	}

	// 保留最大值
	if leadingZeros > h.registers[bucket] {
		h.registers[bucket] = leadingZeros
	}
}

func (h *HyperLogLog) AddString(s string) {
	h.Add([]byte(s))
}

// Count 估算基数
func (h *HyperLogLog) Count() uint64 {
	sum := 0.0
	emptyRegs := 0

	// 调和平均分母
	for i := 0; i < numBuckets; i++ {
		val := h.registers[i]
		sum += 1.0 / math.Pow(2.0, float64(val))
		if val == 0 {
			emptyRegs++
		}
	}

	alpha := 0.7213 / (1.0 + 1.079/float64(numBuckets))
	estimate := alpha * float64(numBuckets*numBuckets) / sum

	// 小基数修正
	if estimate <= 2.5*float64(numBuckets) && emptyRegs > 0 {
		estimate = float64(numBuckets) *
			math.Log(float64(numBuckets)/float64(emptyRegs))
	}

	return uint64(estimate)
}

// Merge 合并另一个 HLL（取桶最大值）
func (h *HyperLogLog) Merge(other *HyperLogLog) {
	for i := 0; i < numBuckets; i++ {
		if other.registers[i] > h.registers[i] {
			h.registers[i] = other.registers[i]
		}
	}
}

func fnvHash64(data []byte) uint64 {
	h := fnv.New64a()
	h.Write(data)
	return h.Sum64()
}
```

### 使用示例

```go
// 模拟 100 万独立用户
h := hll.New()
for i := 0; i < 1_000_000; i++ {
	h.AddString(fmt.Sprintf("user_%d", i))
}
fmt.Println(h.Count())
// 输出: 996578（接近100万，误差约0.34%）

// 合并上午和下午的数据
morning := hll.New()
morning.AddString("user_A")
morning.AddString("user_B")

afternoon := hll.New()
afternoon.AddString("user_B")
afternoon.AddString("user_C")

morning.Merge(afternoon)
fmt.Println(morning.Count()) // 输出: 3 (A,B,C 去重)
```

---

## 7. HLL 的局限性

**无法判断某个用户是否访问过。**

HLL 只记录"最长前导零"这个聚合值，原始信息不可逆地丢失了。桶[42]=4 可能是 user_123 产生的，也可能是 user_789 产生的。

如果需要"判断元素是否存在"，应该用 Bloom Filter 或 HashSet。

---

## 8. 与 Bloom Filter、HashSet 的对比

| 维度 | HyperLogLog | Bloom Filter | HashSet |
|------|-------------|--------------|---------|
| **核心目的** | 统计总数 | 查询存在性 | 精确存储 |
| **内存 (100万元素)** | 12 KB | ~1 MB | ~50 MB |
| **统计总数** | ✅ 估算 (±0.81%) | ❌ | ✅ 精确 |
| **查询存在性** | ❌ | ✅ 概率性 | ✅ 精确 |
| **误报率** | - | 可配置 (0.1%~1%) | 0% |
| **漏报率** | - | 0% | 0% |
| **删除支持** | ❌ | ❌ | ✅ |
| **合并支持** | ✅ (桶取max) | ✅ (按位OR) | ❌ |

### 选型指南

```
统计总数？
├─ 是 → 必须 100% 精确？
│       ├─ 是 → HashSet（或数据库）
│       └─ 否 → HyperLogLog
└─ 否 → 查询存在性？
        ├─ 必须 100% 精确 → HashSet
        └─ 可容忍误判 → Bloom Filter
```

### 一句话选型

| 需求 | 方案 | 原因 |
|------|------|------|
| "今天大概多少独立访客？" | **HyperLogLog** | 12KB，1%误差，可合并 |
| "这个IP在黑名单里吗？" | **Bloom Filter** | 1MB，绝不漏判 |
| "这个订单号已存在吗？" | **HashSet** | 0%误差，必须精确 |

---

## 9. 实际工程架构

```
┌────────────────────────────────────────────┐
│                  业务系统                   │
├────────────────────────────────────────────┤
│  大盘UV看板    →  HyperLogLog   (12KB)     │
│  新用户判重    →  Bloom Filter  (1MB)      │
│  会话管理      →  HashSet       (按需)     │
│  精确审计      →  数据库         (磁盘)    │
└────────────────────────────────────────────┘
```

**UV 聚合的典型用法**：

```go
// 小时 HLL → 天 HLL
daily := hll.New()
for _, hourly := range hourlyHLLs {
    daily.Merge(hourly)
}

// 天 HLL → 周 HLL
weekly := hll.New()
for _, daily := range weekDays {
    weekly.Merge(daily)
}
```

Redis 用户直接用：
```redis
PFADD uv:2024-01-01 user1 user2 user3
PFCOUNT uv:2024-01-01
PFMERGE uv:week1 uv:2024-01-01 uv:2024-01-02 ...
```

---

## 10. 总结

HyperLogLog 用**12KB 固定内存**解决了"海量数据近似去重计数"的问题。

它的精妙之处在于：
- 用**哈希值的随机性**模拟抛硬币
- 用**前导零长度**衡量罕见程度
- 用**分桶**减少方差
- 用**调和平均**抑制离群值
- 用**取最大值合并**实现并集

代价是不能查询单个元素是否存在。但如果你只需要一个"大概多少"的答案，HyperLogLog 是目前最优的工程方案。
