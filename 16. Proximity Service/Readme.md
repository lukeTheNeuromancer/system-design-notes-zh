# 第 16 章：地理位置邻近服务 (Proximity Service)

## 简介
**地理位置邻近服务 (proximity service)** 用于查找附近的地点，例如餐厅、酒店、加油站和其他商家。Google Maps 和 Yelp 等应用使用这类功能，帮助用户发现指定半径内的地点。


## 第一步：理解问题并明确范围

### **功能需求**
1. 基于用户位置（纬度、经度）和搜索半径**搜索商家**。
2. **允许商家**新增、更新或删除商家信息（非实时）。
3. 请求时**提供商家详细信息**。

### **非功能需求**
- **低延迟**：用户应能快速得到响应。
- **数据隐私**：遵守 GDPR 和 CCPA 法规。
- **高可用性**：在繁忙地点的用餐高峰期能应对流量尖峰。

### **粗略估算 (Back-of-the-Envelope Estimation)**
- **1 亿日活跃用户**。
- 系统中有 **2 亿商家**。
- **搜索 QPS 计算**：
  - 每位用户每天搜索 **5 次**。
  - **搜索 QPS** = (100M × 5) / 86,400 ≈ **5,000 QPS**。

---

## 第二步：高层设计

### **API 设计**
#### **搜索附近商家**
GET /v1/search/nearby

- **请求参数**：
  - `latitude`：用户所在位置的纬度。
  - `longitude`：用户所在位置的经度。
  - `radius`：搜索半径（默认：5000m）。

#### **商家 API**
| API 接口                          | 描述                                           |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}`         | 获取商家详细信息                    |
| `POST /v1/businesses`             | 新增商家                              |
| `PUT /v1/businesses/{id}`         | 更新商家信息                         |
| `DELETE /v1/businesses/{id}`      | 从系统中删除商家               |


### **数据模型**
- 由于以下两个功能的使用频率非常高，读量很大，关系型数据库（如 MySQL）比较合适。
  - 搜索附近商家
  - 查看商家详细信息

### **数据结构**
- 关键的数据库表是商家表 (business table) 和地理空间索引表 (geospatial index table)。
- 商家表包含商家的详细信息。

### **高层系统架构**
系统由两部分组成：基于位置的服务（LBS，Location based service）和商家相关服务。

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="高层设计" width="400" />
</div>

- **基于位置的服务 (LBS，Location-Based Service)**：
  - 处理基于位置的搜索查询。
  - 读多写少的服务，无写入请求。
  - 尤其在人口密集地区的用餐高峰期 QPS 很高，且系统是无状态的。
- **商家服务 (Business Service)**：处理两类请求。
  - 商家新增、更新或删除商家。
  - 顾客查看商家详细信息。
- **负载均衡器**：将流量路由到 LBS 和商家服务。
- **数据库集群**：
  - 对读多写少的负载使用**主从架构 (primary-replica architecture)**。
  - LBS 读取的数据与主数据库写入的数据之间可能存在不一致。
  - 这种不一致不是问题，因为商家信息不是实时更新的。


---

## 第三步：获取附近商家的算法

### **方案 1：二维搜索（朴素方法）**

<div style="margin-left:3rem">
    <img src="./images/2d-search.png" alt="二维搜索" width="250" />
</div>

最直观的方法是：以预设半径画一个圆，找出圆内所有商家。

**SQL 查询：**
```
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```
**问题：**
- **效率低下**：需要扫描整个数据库。
- 受限于**一维索引**（纬度/经度）。

一个潜在改进是在经度和纬度列上建索引，虽然会稍好一些，但仍然很慢。

### 更好的方法
- 上一个方案的问题是，数据库索引只能在一个维度上加速搜索。
- 最佳方法是使用地理空间索引 (geospatial indexing)，把二维数据表示成一维。
  - 哈希：均匀网格、Geo Hash
  - 树：四叉树 (Quadtree)、Google S2、RTree

  <div style="margin-left:3rem">
    <img src="./images/geospatial-index-types.png" alt="二维索引" width="500" />
  </div>


### **方案 2：均匀划分网格**

  <div style="margin-left:3rem">
    <img src="./images/even-grid.png" alt="均匀网格" width="400" />
  </div>

- **将整个世界划分成固定大小的网格**。
- **问题**：商家分布不均匀（城市密集，农村稀疏）。

### **方案 3：Geohash**
- 沿着本初子午线和赤道把地球分成四个象限，再把每个网格分成四个更小的网格。
- 每个网格可以通过经度和纬度交替的比特位来表示。
- 重复这个细分过程。

  <div style="margin-left:3rem">
    <img src="./images/geohash.png" alt="Geohash" width="300" />
    <img src="./images/geohash-1.png" alt="Geohash" width="285" />
  </div>


- **将经纬度编码成单个字母数字字符串**。它有 12 种精度（层级）。
- **层级式网格结构**支持高效搜索。
- 根据表格选择合适的最小 geohash 长度来匹配精度。
  <div style="margin-left:3rem">
    <img src="./images/geohash-radius-mapping.png" alt="Geohash 半径" width="400" />
  </div>
- Geohash 保证两个 geohash 的公共前缀越长，它们在地理位置上越接近。

- **挑战**：
  <div style="margin-left:3rem">
    <img src="./images/boundary-issue.png" alt="边界问题" width="300" />
  </div>

  - **边界问题**（靠近网格边缘的商家可能被排除）。
    - 两个地点可能非常接近，但没有任何公共前缀（可能位于赤道两侧）。
    - 两个地点可能有很长的公共前缀，但属于不同的 geohash。
  - 解决方案：需要同时搜索相邻网格。


### **方案 4：四叉树 (Quadtree)**

  四叉树是一种树形数据结构，递归地把二维空间分成四个象限，每个内部节点恰好有四个子节点，代表该空间的四个子区域。
  - 四叉树是内存数据结构，运行在每台 LBS 服务器上，在服务器启动时构建。

  <div style="margin-left:3rem">
    <img src="./images/quadtree.png" alt="四叉树" width="500" />
  </div>

  - 根节点递归地被拆成 4 个象限，直到所有节点包含的商家数都不超过 x（本例中为 100）。

  <div style="margin-left:3rem">
    <img src="./images/building-quadtree.png" alt="构建四叉树" width="500" />
  </div>

- 四叉树索引占用内存不大（通常是 GB 级别），很容易装入一台服务器。
- 由于建树的时间复杂度是 nlogn，可能需要几分钟来建树。
- **适合 k 近邻搜索查询**（例如查找最近的加油站）。

  <div style="margin-left:3rem">
    <img src="./images/realworld-quadtree.png" alt="真实世界的四叉树" width="400" />
  </div>

#### 运维考虑
 - 对于约 2 亿商家，服务器启动时建四叉树可能需要几分钟。
 - 建树期间无法对外提供服务，因此新版本应增量发布到部分服务器。
 - 更新商家或新增商家时，最简单的方法是增量重建四叉树。（会引发大量缓存失效）
 - 也可以在运行时直接更新四叉树，但实现更复杂。（需要加锁机制）

### **方案 5：Google S2**
它基于 Hilbert 曲线把球面映射到一维索引。Hilbert 曲线上彼此接近的两个点，在一维空间中也接近。


  <div style="margin-left:3rem">
    <img src="./images/hilbert-curve.png" alt="Hilbert 曲线" width="300" />
    <img src="./images/geofence.png" alt="地理围栏" width="355" />
  </div>

- **使用 Hilbert 曲线把地球划分成小单元**。
- 非常适合地理围栏 (geofencing)，因为它可以用不同层级覆盖任意形状的区域。
- 地理围栏还可以定义围绕目标区域的参数。
- 另一个优点是，无需使用固定精度层级，可以指定 S2 的最小层级、最大层级和最大单元数。


## 权衡对比

#### Geohash
- 简单易用、易于实现——无需建树/重建树
- 支持固定半径的结果
- 更新索引很容易。
- 不能根据人口密度动态调整网格大小。

#### 四叉树
- 实现稍难一些。
- 支持获取 k 近邻商家。
- 可以根据人口密度动态调整网格大小。
- 更新索引更复杂，可能需要重建整棵树。

---

## 第四步：数据库扩容与缓存策略

### **扩容商家表**
- **按商家 ID 分片**保证数据均匀分布。
- 表中每个商家单独占一行。

| Geohash | Business ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **扩容地理空间索引**
- 对 geohash 表来说，分片可能不合适。这种情况下所有数据都能装进一台服务器，所以从技术上讲没有分片的必要。
- 更好的方法是使用读副本来分担读负载。



---

### **缓存策略**
最直观的缓存键选择是位置坐标，但它有几个问题：
 - GPS 的位置坐标不够精确。
 - 用户会移动，导致位置坐标变化。
 - 更好的键是 geohash。

| 缓存键  | 缓存值 |
|------------|------------|
| `geohash`  | 该网格内的商家 ID 列表 |
| `business_id` | 商家详情（名称、地址、评价等） |

---

## 第五步：部署策略与最终架构

### **区域与可用区**
- 在**多个区域**部署 LBS 和商家服务。

### **处理实时更新**
- **商家更新每天批量处理一次**。

### **最终系统架构**


  <div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="最终设计" width="500" />
  </div>


最终算法如下：

## 获取附近商家的步骤
1. **用户请求：**
   - 用户搜索 **500 米内**的餐厅。
   - 客户端向**负载均衡器**发送**纬度 (37.776720)、经度 (-122.416730) 和半径 (500m)**。

2. **请求转发：**
   - **负载均衡器 (LB)** 将请求转发到**基于位置的服务 (LBS)**。

3. **Geohash 计算：**
   - LBS 确定与半径匹配的 **geohash 长度**。
   - 查参考表可知，**500m 对应的 geohash 长度为 6**。

4. **获取相邻 Geohash：**
   - LBS 计算**相邻的 geohash**，以覆盖周边区域。
   - 结果是一个列表：
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```

5. **从 Redis 获取商家 ID：**
   - 对列表中的每个 geohash，LBS 查询 **Geohash Redis 服务器**获取**商家 ID**。
   - 使用并行查询以最小化延迟。

6. **获取并排序商家：**
   - LBS 从**商家信息 Redis 服务器**获取**完整商家详情**。
   - 商家按距离用户的**距离排序**。
   - **排序后的结果**返回给客户端。

## 关键优化
- **并行 Redis 调用**：减少响应时间。
- **Geohash 索引**：保证高效的空间查询。
- **缓存**：加速商家数据的查找与读取。

这种方法保证了用户附近商家的检索**低延迟、可扩展**。

---

### **选择最佳索引方法**
| 索引方法 | 优点 | 缺点 |
|----------------|------|------|
| **Geohash** | 易于实现，邻近搜索高效 | 边界问题，网格大小固定 |
| **四叉树 (Quadtree)** | 根据密度动态调整，支持 k 近邻查询 | 更复杂，需要树的再平衡 |
| **Google S2** | 地理围栏功能强大，Google Maps 在用 | 更难实现 |

---

## 参考资料
1. [Geohash 算法](https://www.movable-type.co.uk/scripts/geohash.html)
2. [四叉树索引](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 几何库](https://s2geometry.io/)
