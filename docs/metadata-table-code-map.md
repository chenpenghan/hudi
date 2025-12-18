# Metadata Table 读写代码速查

面向“metadata table 写入在哪里、读取哪个分区”问题的代码定位说明。

## 写入链路
- **常规/表服务提交**：`HoodieBackedTableMetadataWriter#update(HoodieCommitMetadata, …)` 调 `HoodieTableMetadataUtil.convertMetadataToRecords` 生成各分区记录（FILES 必选，BLOOM_FILTERS/COLUMN_STATS 按配置）。  
  代码：`hudi-client/hudi-client-common/src/main/java/org/apache/hudi/metadata/HoodieBackedTableMetadataWriter.java:868-925`，生成分区记录处：`hudi-common/src/main/java/org/apache/hudi/metadata/HoodieTableMetadataUtil.java:269-285`。
- **Clean**：`update(HoodieCleanMetadata, …)` -> `convertMetadataToRecords`，同样写入 FILES 及可选 BLOOM_FILTERS/COLUMN_STATS。  
  代码：`HoodieBackedTableMetadataWriter.java:880-884`，分区生成：`HoodieTableMetadataUtil.java:449-468`。
- **Restore**：`update(HoodieRestoreMetadata, …)` -> `convertMetadataToRecords`（FILES 及可选 BLOOM_FILTERS/COLUMN_STATS）。  
  代码：`HoodieBackedTableMetadataWriter.java:892-897`，分区生成：`HoodieTableMetadataUtil.java:586-608`。
- **Rollback**：`update(HoodieRollbackMetadata, …)` -> `convertMetadataToRecords`（FILES 及可选 BLOOM_FILTERS/COLUMN_STATS）。  
  代码：`HoodieBackedTableMetadataWriter.java:905-925`，分区生成：`HoodieTableMetadataUtil.java:631-654`。
- **首次引导写入**：初始化时 `initialCommit` 会直接构造并提交 FILES 分区记录，若配置启用则同时写 BLOOM_FILTERS 与 COLUMN_STATS。  
  代码：`HoodieBackedTableMetadataWriter.java:1047-1088`。

## 读取链路与分区对应
- **FILES 分区**：  
  - 列出所有分区：`BaseTableMetadata.fetchAllPartitionPaths` 读取 `MetadataPartitionType.FILES`。  
    代码：`hudi-common/src/main/java/org/apache/hudi/metadata/BaseTableMetadata.java:279-298`。  
  - 列出分区内文件：`fetchAllFilesInPartition` / `fetchAllFilesInPartitionPaths` 同样从 FILES 分区读取。  
    代码：`BaseTableMetadata.java:305-359`。
- **BLOOM_FILTERS 分区**：`BaseTableMetadata#getBloomFilter` / `getBloomFilters` 读取布隆过滤器索引。  
  代码：`BaseTableMetadata.java:162-228`。
- **COLUMN_STATS 分区**：`BaseTableMetadata#getColumnStats` 读取列统计索引。  
  代码：`BaseTableMetadata.java:231-274`。

分区名称与前缀定义参考：`hudi-common/src/main/java/org/apache/hudi/metadata/MetadataPartitionType.java`。
