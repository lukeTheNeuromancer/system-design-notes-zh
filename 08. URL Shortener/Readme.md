# 第 8 章：设计短链接服务 (Design a URL Shortener)

## 简介
本章讨论类似 TinyURL 的短链接服务的系统设计。系统的主要目标包括**短链接生成**、**重定向**以及**高扩展性**，以应对大规模流量。

### 需求
- 缩短后的 URL 必须**唯一**且**尽可能短**。
- 每天处理 **1 亿次 URL 生成**，支持 10 年容量。
- 支持**高效的读操作**，读写比为 10:1。
- 存储 3650 亿条记录，10 年内约需要 **365 TB** 存储空间。

---

## 步骤 1：高层设计

### API 端点
1. **短链接生成：**
   - 端点：`POST api/v1/data/shorten`
   - 参数：`{longUrl: longURLString}`
   - 返回：`shortURL`

2. **URL 重定向：**
   - 端点：`GET api/v1/shortUrl`
   - 返回用于重定向的 `longURL`。

    <p align="center">
    <img src="./images/url-redirection.png" alt="URL 重定向" width="600">
    </p>

### URL 重定向
- **301 重定向：** 301 重定向表示请求的 URL 已“永久”转移到长 URL。浏览器会缓存该响应，对同一 URL 的后续请求将不再发送到短链接服务。
- **302 重定向：** 临时重定向；适用于点击量统计等分析场景。

### URL 缩短
<p align="center">
    <img src="./images/url-shortening.png" alt="URL 缩短" width="400">
</p>

- 使用**哈希函数**生成短链接，将长 URL 映射为唯一的缩短版本。
- 哈希函数必须满足以下要求：
    - 每个 longURL 必须被哈希为一个 hashValue。
    - 每个 hashValue 都能映射回对应的 longURL。


---

## 步骤 2：设计深挖

### 数据模型
将 `<shortURL, longURL>` 映射存储在关系型数据库中，以优化内存使用。表结构包括：
- `id`（主键）、
- `shortURL`、
- `longURL`。

    <img src="./images/table-schema.png" alt="表结构" width="300">

### 哈希函数
#### 1. Base 62 转换：
- 使用字符 `[0-9, a-z, A-Z]` 对数字编码，共 **62 个可用字符**。
- 进制转换是短链接服务常用的另一种方法。
- 可以为短链接分配一个唯一 id，再将该 ID 做 base 62 转换得到短 URL。
- 7 个字符的哈希可支持最多 **3.5 万亿个唯一 URL**，足以容纳 3650 亿个 URL。

**示例：**
将 ID `2009215674938` 转换为 Base 62：
- `2009215674938` → `zn9edcu`。

#### 2. 哈希 + 冲突解决：
- 使用 CRC32、MD5 或 SHA-1 等哈希函数。

    <img src="./images/hash-function.png" alt="哈希函数" width="500">

- 一种做法是取哈希值的前 7 个字符；但这种方法可能导致哈希冲突。
- 解决冲突的一种方式是递归追加一个新的预定义字符串，直到不再冲突，但这种做法开销较大。
- 使用**布隆过滤器 (Bloom Filters)** 解决冲突，实现高效查找。

    <p align="center">
    <img src="./images/url-lookup.png" alt="URL 查找" width="500">
    </p>

### 对比

-  **哈希 + 冲突解决：**
    - 短 URL 长度固定
    - 不需要唯一 ID 生成器
    - 可能发生冲突，需要解决冲突
    - 无法找到下一个可用的短 URL，因为它不依赖于 ID

- **Base 62 转换**
    - 长度不固定，随 ID 增长而增长
    - 需要唯一 ID 生成器
    - 不会发生冲突
    - 如果 ID 每次递增 1，很容易找到下一个短 URL（可能存在安全隐患）


---

### URL 缩短流程

<p align="center">
    <img src="./images/url-shortening-flow.png" alt="URL 缩短" width="500">
</p>

1. 检查数据库中是否存在 `longURL`。
2. 若存在，返回已有的 `shortURL`。
3. 否则：
   - 使用**分布式 ID 生成器**生成唯一 ID。
   - 用 Base 62 将该 ID 转换为 `shortURL`。
   - 将 `<id, shortURL, longURL>` 映射存入数据库。



---

### URL 重定向流程
<p align="center">
    <img src="./images/url-redirecting-flow.png" alt="URL 缩短" width="600">
</p>

1. 用户点击 `shortURL`。
2. 查询 `<shortURL, longURL>` 映射：
   - 先查**缓存**以加快访问。
   - 缓存未命中则查数据库。
3. 将用户重定向到 `longURL`。


---

## 其他考量
### 限流器
- 按 IP 限制请求量，防止滥用。

### 扩展性
1. **Web 层：** 无状态，通过增减 Web 服务器扩展。
2. **数据库层：** 使用复制和分片。

### 分析
- 收集点击量、来源、时间戳等数据，用于业务洞察。

### 高可用与可靠性
- 使用数据库复制和容错设计，确保服务持续可靠。
