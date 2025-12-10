<!--
* Licensed to the Apache Software Foundation (ASF) under one
* or more contributor license agreements.  See the NOTICE file
* distributed with this work for additional information
* regarding copyright ownership.  The ASF licenses this file
* to you under the Apache License, Version 2.0 (the
* "License"); you may not use this file except in compliance
* with the License.  You may obtain a copy of the License at
*
*      http://www.apache.org/licenses/LICENSE-2.0
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an "AS IS" BASIS,
* WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific language governing permissions and
-->

# Description of the relationship between each module

This repo contains the code that integrate Hudi with Spark. The repo is split into the following modules

`hudi-spark`
`hudi-spark3.3.x`
`hudi-spark3.4.x`
`hudi-spark3.5.x`
`hudi-spark4.0.x`
`hudi-spark3-common`
`hudi-spark-common`

* hudi-spark is the module that contains the code that spark3 version would share.
* hudi-spark3.3.x is the module that contains the code that compatible with spark3.3.x versions.
* hudi-spark3.4.x is the module that contains the code that compatible with spark 3.4.x versions.
* hudi-spark3.5.x is the module that contains the code that compatible with spark 3.5.x versions.
* hudi-spark4.0.x is the module that contains the code that compatible with spark 4.0.x versions.
* hudi-spark3-common is the module that contains the code that would be reused between spark3.x versions.
* hudi-spark-common is the module that contains the code that would be reused between spark3.x and spark4.x versions.

## Spark 引擎一次 Upsert 写入流程（简要拆解）
This section offers a brief Chinese-language breakdown of the Spark datasource upsert path for quick reference.
1. **入口（DataFrameWriter -> DefaultSource）**：在 Spark 中执行 `df.write.format("hudi")...save(path)` 时，首先进入 `hudi-spark-common/src/main/scala/org/apache/hudi/DefaultSource.scala`（DataSource V1），随后调用 `HoodieSparkSqlWriter.write`。
2. **参数合并与表元数据**：`HoodieSparkSqlWriterInternal.writeInternal` 会检查表是否存在、合并用户参数与表配置（例如表类型、索引、KeyGen 配置），并推断操作类型为 `UPSERT`（若用户未强制覆盖）。
3. **构建 MetaClient 与 WriteClient**：根据表是否存在创建/加载 `HoodieTableMetaClient`，再通过 `DataSourceUtils.createHoodieClient` 生成 `SparkRDDWriteClient`，同时启动一次新的提交时间戳（`startCommit`）并校验必须使用 `KryoSerializer`。
4. **数据准备**：将 DataFrame 按 Avro Schema 转为 `HoodieRecord`（`HoodieCreateRecordUtils.createHoodieRecordRdd`），如果是预处理写入则保留 meta 列；插入场景会在这里做去重，Upsert 依赖索引无需额外去重。
5. **核心写路径（UPSERT）**：`DataSourceUtils.doWriteOperation` 在 `UPSERT` 情况下调用 `SparkRDDWriteClient.upsert`（或 `upsertPreppedRecords`），由索引定位目标文件/日志，使用 payload/record merger 完成行级合并：COW 重写 Parquet 数据文件，MOR 将更新先写入日志块（必要时再触发压缩）。
6. **提交与表服务**：写入返回 `HoodieWriteResult` 后，`commitAndPerformPostOperations` 负责 `client.commit` 落地时间线、可选调度异步 compaction/clustering，并执行 Hive/Glue 等元数据同步，最后刷新 Spark catalog 缓存。

## Description of Time Travel
* `HoodieSpark3_2ExtendedSqlAstBuilder` have comments in the spark3.2's code fork from `org.apache.spark.sql.catalyst.parser.AstBuilder`, and additional `withTimeTravel` method.
* `SqlBase.g4` have comments in the code forked from spark3.2's parser, and add SparkSQL Syntax  `TIMESTAMP AS OF` and `VERSION AS OF`.

### Time Travel Support Spark Version:

| version | support |
| ------  | ------- |
| 2.4.x   |    No   |
| 3.0.x   |    No   |
| 3.1.2   |    No   |
| 3.2.0   |    Yes  |

### To improve:
Spark3.3 support time travel syntax link [SPARK-37219](https://issues.apache.org/jira/browse/SPARK-37219). 
Once Spark 3.3 released. The files in the following list will be removed:
* hudi-spark3.3.x's `HoodieSpark3_3ExtendedSqlAstBuilder.scala`, `HoodieSpark3_3ExtendedSqlParser.scala`, `TimeTravelRelation.scala`, `SqlBase.g4`, `HoodieSqlBase.g4`
Tracking Jira: [HUDI-4468](https://issues.apache.org/jira/browse/HUDI-4468)

Some other improvements undergoing:
* Port borrowed classes from Spark 3.3 [HUDI-4467](https://issues.apache.org/jira/browse/HUDI-4467)

