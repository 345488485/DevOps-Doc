###### Elasticsearch 备份还原迁移

1、📦 查看所有已注册的快照仓库（Snapshot Repositories）

````shell
GET _snapshot/_all
````

2、安装repository-s3插件

https://www.elastic.co/guide/en/elasticsearch/plugins/index.html

https://www.elastic.co/guide/en/elasticsearch/plugins/7.17/repository-s3.html

https://support.huaweicloud.com/intl/zh-cn/bestpractice-css/css_07_0001.html

````shell
1、离线安装

wget https://artifacts.elastic.co/downloads/elasticsearch-plugins/repository-s3/repository-s3-7.17.29.zip
````

```shell
2、在线安装

bin/elasticsearch-plugin install repository-s3
```

```shell
删除插件

bin/elasticsearch-plugin remove repository-s3
```

3、创建快照仓库

S3存储库配置

````shell
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "es-backup",
    "endpoint": "s3.ap-east-1.amazonaws.com",
    "protocol": "https",
    "storage_class": "STANDARD_IA",
    "server_side_encryption": true,
    "buffer_size": "128mb",
    "max_restore_bytes_per_sec": "200mb",
    "max_snapshot_bytes_per_sec": "150mb"
  }
}
````

MinIO存储库配置

```shell
PUT _snapshot/minio_repo
{
  "type": "s3",
  "settings": {
    "bucket": "elastic-backups",
    "endpoint": ":9000",
    "protocol": "https",
    "path_style_access": true,
    "access_key": "minioadmin",
    "secret_key": "minioadmin",
    "compress": true,
    "chunk_size": "64mb"
  }
}
```

存储参数对照表

|         参数名称          |  S3推荐值   | MinIO推荐值 |     作用说明     |
| :-----------------------: | :---------: | :---------: | :--------------: |
|        buffer_size        |    128mb    |    64mb     |  上传缓冲区大小  |
|        chunk_size         |     无      |    64mb     |   数据分块尺寸   |
| max_restore_bytes_per_sec |    200mb    |    500mb    |   恢复速率限制   |
|       storage_class       | STANDARD_IA |      -      | 存储类型优化成本 |
|  server_side_encryption   |    true     |    true     |    服务端加密    |

多级备份策略

| 多级备份策略 |  频率  | 保留周期 |  存储类型  | 恢复优先级 |
| :----------: | :----: | :------: | :--------: | :--------: |
|   即时快照   | 每小时 |   7天    |  本地NVMe  |     P0     |
|   日常备份   |  每天  |   30天   |  企业SSD   |     P1     |
|   周度归档   |  每周  |   1年    | S3标准存储 |     P2     |
|   月度冷备   |  每月  |   5年    | S3 Glacier |     P3     |

自动化备份配置

Snapshot Lifecycle Management (SLM) 策略配置

每天 23:30 自动备份所有索引到名为 minio_repo 的仓库，并保留 7–30 个快照，超出 30 天的快照会被自动删除。

`````shell
PUT _slm/policy/daily_backup
{
  "schedule": "0 30 23 * * ?",       // 每天 23:30 执行
  "name": "<daily-snap-{now/d}>",   // 快照命名格式
  "repository": "minio_repo",       // 使用的仓库名（需事先注册）
  "config": {
    "indices": ["*"],               // 备份所有索引
    "ignore_unavailable": true,     // 忽略不可用索引
    "include_global_state": false   // 不包含集群元数据
  },
  "retention": {
    "expire_after": "30d",          // 超过 30 天删除
    "min_count": 7,                 // 至少保留 7 个
    "max_count": 30                 // 最多保留 30 个
  }
}
`````

手动启动一次 SLM 策略用于测试：

````shell
POST _slm/policy/daily_backup/_execute
````

查看快照策略状态

```shell
GET _slm/policy/daily_backup
```

查看是否执行成功：

````shell
GET _slm/stats
````

查看失败信息（如果有）：

````shell
GET _slm/status
````

全量恢复流程

````shell
# 查看可用快照列表
GET _snapshot/minio_repo/_all

# 执行全量恢复
POST _snapshot/minio_repo/snapshot_2023-08/_restore
{
  "indices": "*",
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1"
}
````

部分恢复场景

|  恢复类型  |     适用场景     |          命令示例           |         注意事项         |
| :--------: | :--------------: | :-------------------------: | :----------------------: |
|  指定索引  |  误删除单个索引  | POST …?indices=logs-2023-08 |    检查索引映射兼容性    |
| 时间点恢复 | 数据污染需要回滚 |      结合快照+事务日志      |   需要开启索引版本控制   |
| 跨版本恢复 | 集群升级失败回退 |    使用兼容版本集群恢复     |     验证版本兼容矩阵     |
|  异地恢复  |   灾难恢复演练   | 在新集群注册原仓库执行恢复  | 网络带宽需满足数据量要求 |

