# Hive问题
1. 使用jdbc连接hive后调用配置参数：
```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.HiveInputFormat;
```
提示报错信息：
```
Error while processing statement: Cannot modify hive.input.format at runtime. It is not in list of params that are allowed to be modified at runtime
```
hive设置权限后，修改追加参数白名单设置
解决方案：
在hive-site.xml中增加如下配置：
```xml
<property>
  <name>hive.security.authorization.sqlstd.confwhitelist</name>
  <value>mapred.*|hive.*|mapreduce.*|spark.*</value>
</property>

<property>
 <name>hive.security.authorization.sqlstd.confwhitelist.append</name>
  <value>mapred.*|hive.*|mapreduce.*|spark.*</value>
</property>
```
