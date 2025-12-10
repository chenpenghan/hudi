<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# Spark 引擎一次 upsert 的执行流程

以下以 `DataFrameWriter.format("hudi").option("hoodie.datasource.write.operation","upsert").save(path)` 为例，给出一次 upsert 的主要执行链路与代码位置。

1. **写入入口**
   - `org.apache.hudi.DefaultSource#createRelation(SQLContext, SaveMode, Map, DataFrame)`（`hudi-spark-datasource/hudi-spark-common/src/main/scala/org/apache/hudi/DefaultSource.scala`）接收 Spark DataFrameWriter 的 save 请求。
   - 根据 `hoodie.datasource.write.operation` 判定 upsert，进入 `HoodieSparkSqlWriter.write(...)`。

2. **DataFrame 转换与提交准备**
   - `HoodieSparkSqlWriterInternal.writeInternal(...)`（同路径）负责合并表配置、生成新的提交时间（`HoodieActiveTimeline.createNewInstantTime`），并创建 `SparkRDDWriteClient`。
   - 在 upsert 分支中通过 `HoodieCreateRecordUtils.createHoodieRecordRdd` 将 DataFrame 转成 `JavaRDD<HoodieRecord>`，必要时 `DataSourceUtils.dropDuplicates` 去重。
   - 通过 `DataSourceUtils.doWriteOperation(..., WriteOperationType.UPSERT, ...)` 分发到写客户端。

3. **写客户端执行 upsert**
   - `SparkRDDWriteClient.upsert`（`hudi-client/hudi-spark-client/src/main/java/org/apache/hudi/client/SparkRDDWriteClient.java`）加载/初始化 `HoodieSparkTable`（COW/MOR），执行 `preWrite`，随后调用表层的 `table.upsert(...)`。
   - COW 表对应 `HoodieSparkCopyOnWriteTable.upsert`（同模块），创建 `SparkUpsertCommitActionExecutor` 开始实际写入。

4. **分区写入与合并**
   - `SparkUpsertCommitActionExecutor.execute` 调用 `HoodieWriteHelper.newInstance().write(...)`，核心逻辑在 `BaseSparkCommitActionExecutor.execute`：
     - 基于记录构建 `WorkloadProfile` 并选择分区器；
     - 在 `mapPartitionsAsRDD` 中对每个分区生成 `WriteStatus`，对于已有文件使用 `HoodieMergeHandle` / `HoodieMergeHelper.runMerge` 合并更新，新增文件则通过 `CreateHandleFactory` 写出（MOR 表会写增量日志块）。

5. **提交与后处理**
   - `SparkRDDWriteClient.postWrite` 收集 `WriteStatus` 后由 `commit` 持久化 `HoodieCommitMetadata`，`HoodieSparkSqlWriterInternal.commitAndPerformPostOperations` 再触发同步、清理等收尾动作。

**调用链小结（按顺序）：**
`DefaultSource#createRelation` → `HoodieSparkSqlWriter.write` → `HoodieSparkSqlWriterInternal.writeInternal` → `DataSourceUtils.doWriteOperation (UPSERT)` → `SparkRDDWriteClient.upsert` → `HoodieSparkCopyOnWriteTable.upsert` → `SparkUpsertCommitActionExecutor.execute` → `HoodieWriteHelper.write` / `BaseSparkCommitActionExecutor.execute` → `HoodieMergeHandle` / `HoodieMergeHelper.runMerge` → `SparkRDDWriteClient.commit`
