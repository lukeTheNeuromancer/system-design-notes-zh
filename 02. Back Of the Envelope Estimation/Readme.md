# 第 2 章：封底估算 (Back-of-the-Envelope Estimation)

## 引言 (Introduction)
封底估算 (Back-of-the-envelope estimation) 是系统设计面试中的一项关键技能。它包括快速、粗略的计算，以评估系统容量或性能。据 Google 高级研究员 (Senior Fellow) Jeff Dean 所言，这些估算有助于通过思维实验和常见的性能基准来判断设计是否满足需求。

本章涵盖关键概念、方法和示例，帮助培养可扩展性与估算能力。

---

## 第 1 节：关键概念 (Key Concepts)

### 2 的幂 (Power of Two)
以 2 的幂来理解数据量是基础：

<img src="./images/power-of-two.png" alt="2 的幂" width="500" />

这些知识有助于进行准确的存储和带宽计算。

---

### 每个程序员都应该知道的延迟数字 (Latency Numbers Every Programmer Should Know)
延迟数字表示计算系统中各种操作所花费的时间。它们提供了相对性能的直观认识：

| 操作 (Operation) | 延迟 (2020) (Latency (2020)) |
|--------------------------|----------------|
| L1 缓存访问 (L1 Cache Access)          | 0.5 ns         |
| L2 缓存访问 (L2 Cache Access)          | 7 ns           |
| 主内存访问 (Main Memory Access)       | 100 ns         |
| SSD 随机读取 (SSD Random Read)          | 150 µs         |
| HDD 随机寻道 (HDD Random Seek)          | 10 ms          |
| 数据中心内往返 (Round-Trip in Data Center)| 500 µs         |
| 跨区域数据中心 (Inter-Region Data Center) | 150 ms         |

**关键洞察 (Key Insights)：**
- 内存快，磁盘慢。
- 尽可能避免磁盘寻道。
- 在互联网上传输数据前先压缩，以节省带宽。


---

### 可用性数字 (Availability Numbers)
高可用性 (High availability，HA) 确保停机时间最小化。可用性用 **9 的个数** 来表示：
- **99%（两个 9）：** 每年约停机 3.65 天
- **99.9%（三个 9）：** 每年约停机 8.8 小时
- **99.99%（四个 9）：** 每年约停机 52 分钟
- **99.999%（五个 9）：** 每年约停机 5.3 分钟
- **99.9999%（六个 9）：** 每年约停机 31.56 秒


像 Amazon、Google 和 Microsoft 这样的云服务商的目标 SLA（服务等级协议，Service Level Agreements）为 **99.9% 或更高**。

---

## 第 2 节：估算示例——Twitter 的 QPS 与存储需求 (Example Estimation - Twitter QPS and Storage Requirements)

### 假设 (Assumptions)
- **3 亿月活用户 (MAU，monthly active users)。**
- **50% 日活用户 (DAU，daily active users)。**
- **每用户平均每天发推：** 2 条。
- **10% 的推文包含媒体。**
- **数据保留：** 5 年。

### 估算 (Estimations)
1. **每秒查询数 (Query Per Second，QPS)：**
   - 日活用户 = \( 300M x 50\% = 150M \)
   - 发推 QPS = \( 150M x 2 tweets / 24 hour / 3600 seconds = ~3500 )
   - 峰值 QPS = \( 2 x 3500 = ~7000 \)

2. **媒体存储 (Media Storage)：**
   - **推文大小组成部分 (Tweet Size Components)：**
     - `tweet_id`: 64 bytes
     - `text`: 140 bytes
     - `media`: 1 MB
   - **每日媒体存储 (Daily Media Storage)：** \( 150M x 2 x 10\% x 1MB = 30TB per day \)
   - **5 年存储 (5-Year Storage)：** \( 30TB x 365 x 5 = ~55PB \)

---

## 第 3 节：高效估算的技巧 (Tips for Effective Estimation)

### 1. 取整与近似 (Rounding and Approximation)
精确度不重要，重点是估算过程。使用整数简化复杂计算。例如：
- \( 99987 / 9.1 \) 可近似为 \( 100,000 / 10 = 10,000 \)。

### 2. 写下假设 (Write Down Assumptions)
把假设清楚地记录下来，以备将来参考。

### 3. 标注单位 (Label Units)
通过标注单位避免歧义（例如用 `5 MB` 而不是 `5`）。

### 4. 常见估算场景 (Common Estimation Scenarios)
- **QPS（每秒查询数）：** 衡量流量强度。
- **峰值 QPS (Peak QPS)：** 考虑流量高峰。
- **存储需求 (Storage Requirements)：** 估算总数据需求。
- **缓存需求 (Cache Requirements)：** 评估缓存的内存需求。
- **服务器数量 (Number of Servers)：** 根据工作负载计算硬件需求。
