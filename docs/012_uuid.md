# UUID：新项目选 v4 还是 v7

UUID 是一个 128 bit 的标识符，通常写成下面这种形式：

```text
019c2a31-7c4e-7abc-8def-123456789abc
```

中间的 `-` 只是文本格式，不属于 UUID 的有效数据。数据库中应尽量使用原生 UUID 或 16 字节二进制类型存储，而不是字符串。

## UUID v1～v7

| 版本 | 生成方式 | 特点 | 建议 |
| --- | --- | --- | --- |
| v1 | 时间戳 + 时钟序列 + 节点 ID | 大致按时间生成，可能暴露生成时间和机器信息 | 遗留方案，不建议新项目使用 |
| v2 | DCE Security | 在 v1 基础上加入本地用户或组信息，RFC 9562 未定义其具体格式 | 基本不用 |
| v3 | namespace + name 的 MD5 | 相同输入得到相同 UUID | 被 v5 取代 |
| v4 | 随机数 | 简单、通用、不包含时间信息，但写入索引的位置随机 | 仍然是可靠的通用选择 |
| v5 | namespace + name 的 SHA-1 | 相同输入得到相同 UUID，适合确定性映射 | 确实需要相同输入生成相同 ID 时使用 |
| v6 | 重排后的 v1 | 保留 v1 语义，同时让时间位适合排序 | 主要用于兼容 v1，新项目优先 v7 |
| v7 | Unix 毫秒时间戳 + 随机数据 | 趋势递增、适合数据库索引，也能从 ID 看出生成时间 | 新项目默认推荐 |

## v4 和 v7 怎么选

| | UUIDv4 | UUIDv7 |
| --- | --- | --- |
| 唯一性来源 | 122 bit 随机数据 | 48 bit 毫秒时间戳 + 74 bit 随机或单调数据 |
| 时间有序 | 否 | 是，毫秒级趋势有序 |
| 数据库索引 | 插入位置随机 | 新值通常追加在索引尾部，局部性更好 |
| 信息泄露 | 不直接包含生成时间 | 可以解析出毫秒级生成时间 |
| 适用场景 | 不希望 ID 暴露时间；生态暂不支持 v7 | 数据库主键、消息 ID、订单 ID 等大多数业务 ID |

如果不担心 ID 暴露生成时间，新项目基本可以直接选择 UUIDv7。它保留了 UUID 分布式生成的便利，同时比 v4 更适合作为数据库索引。

需要注意两点：

- UUIDv7 只能表示生成器看到的时间，不能代替业务上的 `created_at`。
- UUID 是标识符，不是访问凭证。无论 v4 还是 v7，都不应该把“别人猜不到这个 ID”当作权限控制。

## PostgreSQL

PostgreSQL 的 `uuid` 类型占 16 字节，可以存储任意版本的 UUID，v4 和 v7 都可以直接作为主键：

```sql
CREATE TABLE orders (
    id uuid PRIMARY KEY,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

生成能力与 PostgreSQL 版本有关：

| PostgreSQL 版本 | UUIDv4 | UUIDv7 |
| --- | --- | --- |
| 14～17 | 内置 `gen_random_uuid()` | 业务侧生成后写入 |
| 18 及以后 | `uuidv4()` 或 `gen_random_uuid()` | 内置 `uuidv7()` |

PostgreSQL 18 可以直接定义默认值：

```sql
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    created_at timestamptz NOT NULL DEFAULT now()
);
```

如果 UUID 由 Go 服务生成，PostgreSQL 的版本不影响存储，直接写入 `uuid` 字段即可。

## MongoDB

MongoDB 没有独立的 UUID 字段类型，而是使用 BSON Binary Subtype 4 存储标准 UUID。v4 和 v7 都应该保存为 16 字节的 Subtype 4，而不是普通字符串：

```javascript
{
  _id: UUID("019c2a31-7c4e-7abc-8def-123456789abc")
}
```

MongoDB 的 `UUID()` 不传参数时会生成 UUIDv4。目前没有对应的内置 UUIDv7 生成函数，因此 v7 通常由 Go 等业务代码生成，再以 Binary Subtype 4 写入。

MongoDB 对同长度、同 subtype 的二进制值按字节比较，因此标准字节序的 UUIDv7 可以保持时间排序特性。`_id` 本身有唯一 B-tree 索引，用 v7 作为 `_id` 时也能获得比随机 v4 更好的写入局部性。

不过，如果 ID 只在 MongoDB 内部使用，并不要求跨系统采用 UUID，那么默认的 12 字节 `ObjectId` 已经带有时间顺序，通常也足够好。

## 结论

- 新项目默认使用 UUIDv7。
- 明确不希望 ID 暴露生成时间时，使用 UUIDv4。
- 需要根据固定输入稳定生成相同 ID 时，使用 UUIDv5。
- PostgreSQL 和 MongoDB 都能正确存储 v4、v7；是否能在数据库内直接生成 v7，需要单独看数据库版本和生成函数。

## 参考资料

- [RFC 9562：UUID](https://www.rfc-editor.org/rfc/rfc9562.html)
- [PostgreSQL：UUID 类型](https://www.postgresql.org/docs/current/datatype-uuid.html)
- [PostgreSQL：UUID 函数](https://www.postgresql.org/docs/current/functions-uuid.html)
- [MongoDB：UUID()](https://www.mongodb.com/docs/manual/reference/method/uuid/)
- [MongoDB：BSON 类型](https://www.mongodb.com/docs/manual/reference/bson-types/)
- [MongoDB：BSON 比较与排序](https://www.mongodb.com/docs/manual/reference/bson-type-comparison-order/)
