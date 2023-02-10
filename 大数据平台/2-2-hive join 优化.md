## Join 优化
**表Join的顺序（大表放在后面）**
在 Hive 的 Join 中， 默认前面的表会被缓存，而后面表是流式传输。 所以最好把大表放在后面。
也可以明确指定哪个表用于流式传输。
```   
SELECT /*+ STREAMTABLE(emp) */ emp.id,name,salary,dept_name FROM dept JOIN emp ON (dept.id = emp.id);
```

**Map Side Join**
足够小的表，在参与 join 的时候， 可以把小表放在内存中， join 过程就不需要reducer。

**Sort-Merge-Bucket (SMB) Map Join**
这个技术的前提是所有的表都必须是**桶分区（bucket）和排序了的（sort）**

```
hive> set hive.enforce.bucketing=true;
hive> set hive.enforce.sorting=true;
```
建表加桶分区
```
create table buck_emp(
    id int,
    name string,
    salary int)
CLUSTERED BY (id)
SORTED BY (id)
INTO 4 BUCKETS;
```
设置SMB Map Join  优化

```
hive>set hive.enforce.sortmergebucketmapjoin=false;
hive>set hive.auto.convert.sortmerge.join=true;
hive>set hive.optimize.bucketmapjoin = true;
hive>set hive.optimize.bucketmapjoin.sortedmerge = true;
hive>set hive.auto.convert.join=false;  // if we do not do this, automatically Map–Side Join will happen
SELECT u.name,u.salary FROM buck_dept d  INNER JOIN buck_emp u ON d.id = u.id;
```