---
title: Kylin学习笔记
date: 2017-05-02 10:40:01
tags:
    - kylin
    - infrastructure
    - data
---

## 基础知识

### OLAP(on-Line AnalysisProcessing)的实现方式

- ROLAP:
基于关系数据库的OLAP实现（Relational OLAP）。ROLAP将多维数据库的多维结构划分为两类表:一类是事实表,用来存储数据和维关键字;另一类是维表,即对每个维至少使用一个表来存放维的层次、成员类别等维的描述信息。维表和事实表通过主关键字和外关键字联系在一起,形成了"星型模式"。对于层次复杂的维,为避免冗余数据占用过大的存储空间,可以使用多个表来描述,这种星型模式的扩展称为"雪花模式"。特点是将细节数据保留在关系型数据库的事实表中，聚合后的数据也保存在关系型的数据库中。这种方式查询效率最低，不推荐使用。
- MOLAP:
多维数据组织的OLAP实现（Multidimensional OLAP。以多维数据组织方式为核心,也就是说,MOLAP使用多维数组存储数据。多维数据在存储中将形成"立方块（Cube）"的结构,在MOLAP中对"立方块"的"旋转"、"切块"、"切片"是产生多维数据报表的主要技术。特点是将细节数据和聚合后的数据均保存在cube中，所以以空间换效率，查询时效率高，但生成cube时需要大量的时间和空间。

- HOLAP: 基于混合数据组织的OLAP实现（Hybrid OLAP）。如低层是关系型的，高层是多维矩阵型的。这种方式具有更好的灵活性。特点是将细节数据保留在关系型数据库的事实表中，但是聚合后的数据保存在cube中,聚合时需要比ROLAP更多的时间,查询效率比ROLAP高，但低于MOLAP。

- kylin的cube数据是作为key-value结构存储在hbase中的，key是每一个维度成员的组合值，不同的cuboid下面的key的结构是不一样的，例如cuboid={brand，product，year}下面的一个key可能是brand='Nike'，product='shoe'，year=2015，那么这个key就可以写成Nike:shoe:2015，但是如果使用这种方式的话会出现很多重复，所以一般情况下我们会把一个维度下的所有成员取出来，然后保存在一个数组里面，使用数组的下标组合成为一个key，这样可以大大节省key的存储空间，kylin也使用了相同的方法，只不过使用了字典树（Trie树），每一个维度的字典树作为cube的元数据以二进制的方式存储在hbase中，内存中也会一直保持一份。

### cube 构建

- Dimension：Mandatory、hierarchy、derived
- 增量cube: kylin的核心在于预计算缓存数据，因此无法达到真正的实时查询效果。一个cube中包含了多个segment，每一个segment对应着一个物理cube，在实际存储上对应着一个hbase的一个表。每次查询的时候会查询所有的segment聚合之后的值进行返回，但是当segment数量较多时，查询效率会降低，这时会对segment进行合并。被合并的几个segment所对应的hbase表并没有被删除。
- cube词典树：cube数据是作为key-value结构存储在HBase中的。key是每一个维度成员的组合值

### Streaming cubing

- 支持实时数据的cub。与传统的cub一样，共享storage engine(HBase)以及query engine。kylin Streaming cubing相比其他实时分析系统来说，不需要特别大的内存，也不需要实现真正的实时分析。因为在OLAP中，存在几分钟的数据延迟是完全可以接受的。于是实现手法上采用了micro batch approach。
- micro batch approach:将监听到的数据按照时间窗口的方式划分，并且为每个窗口封装了一个微量批处理，批处理后的结果直接存到HBase。
- Streaming cubing data 最终会慢慢转换成普通的cubes,因为所有的数据是直接保存到HBase中的，并且保存为一个新的segment，当segment数量到达一定程度时，job engine会将segment 合并起来形成一个大的cube。


### 实战问题总结

由于集群环境是CDH集群，所以选择了kylin CDH 1.6的版本，支持从Kafka读取消息建立Streaming cubes直接写入HDFS中

- 选择一个集群namenode节点，将解压包放入/opt/cloudrea/parcels/目录中。如果是部署单节点，暂时不用更改配置文件。所有的配置加载都在bin/kylin.sh中。
- 直接kylin.sh start/stop 运行脚本，服务就会在7070端口起一个web界面。这个界面是可以进行可视化操作的。

### Hive 数据源

- 直接测试hive数据源是没有问题的，这一功能比较完善，也是主打功能。

### kafka数据源

从kylin 1.6 版本开始正式支持Kafka做数据源，将Streaming Cubes实时写入 HBase中。这一块在测试的时候也出现了问题：

- Kafka版本问题
  - 由于实验环境的CDH集群Kafka版本是0.9的，而kylin 仅支持0.10以上的版本，所以需要对CDH kafka集群进行升级。

- mapreduce运行环境无jar包
  - kylin中提交cube build之后，map reduce任务直接抛错。错误提示是，找不到Kafka的Consumer类。根本原因是kylin默认集群上的map reduce classpath是会加载kafka-clients.jar包的，所以在提交任务的时候没有将kafka-clients.jar包打进去。这时可以有三种做法：
  - 直接修改kylin的源码，将kafka-clients.jar包给包括进去（待尝试）。
  - 可以通过修改集群的HADOOP_ClASSPATH的路径，将jar包给包括进去。
  - hadoop classpath 查看classpath目录信息 将对应jar包直接拷入map reduce classpath中，这方法简单，但是缺点就是需要逐个得对node进行操作。

- Property is not embedded format
  - 现在意识到，使用开源框架不会看其源码是不行的...就在我折腾俩天终于将mapreduce任务跑起来之后，新的错误出现了:"ava.lang.RuntimeException: java.io.IOException: Property 'xxx' is not embedded format"。莫名奇妙的错误。迫使我直接去github上看kylin kafka模块的源码。在TimedJsonStreamParser.java中发现代码逻辑中默认json数据中，如果key存在下划线就会将该key按照下划线split... 然后看key对应的value是不是map类型，如果不是直接抛出标题的错误。

  - 明确了问题之后，如何复写默认下划线split的配置成为问题。由于官网的文档十分鸡肋，很多坑都没有涉及到，所以继续看源码。发现StreamingParser.java这个类中会去写一些默认的配置。

```
public static final String PROPERTY_TS_COLUMN_NAME = "tsColName";
public static final String PROPERTY_TS_PARSER = "tsParser";
public static final String PROPERTY_TS_PATTERN = "tsPattern";
public static final String EMBEDDED_PROPERTY_SEPARATOR = "separator";

static {
        derivedTimeColumns.put("minute_start", 1);
        derivedTimeColumns.put("hour_start", 2);
        derivedTimeColumns.put("day_start", 3);
        derivedTimeColumns.put("week_start", 4);
        derivedTimeColumns.put("month_start", 5);
        derivedTimeColumns.put("quarter_start", 6);
        derivedTimeColumns.put("year_start", 7);
        defaultProperties.put(PROPERTY_TS_COLUMN_NAME, "timestamp");
        defaultProperties.put(PROPERTY_TS_PARSER, "org.apache.kylin.source.kafka.DefaultTimeParser");
        defaultProperties.put(PROPERTY_TS_PATTERN, DateFormat.DEFAULT_DATETIME_PATTERN_WITHOUT_MILLISECONDS);
        defaultProperties.put(EMBEDDED_PROPERTY_SEPARATOR, "_");
    }

```

自然而然会联想到，这个默认的配置肯定是可以在用户设置的时候通过key（separator）去覆盖的...于是发现在构建Streaming table的时候，可以通过Parse Properties去覆盖配置。
于是直接写成如下的形式：

```
tsColName=timestamp;separator=no

//源码中拿到这个配置之后会做覆盖处理，然后执行 getValueByKey：

protected String getValueByKey(String key, Map<String, Object> rootMap) throws IOException {
        if (rootMap.containsKey(key)) {
            return objToString(rootMap.get(key));
        }

        String[] names = nameMap.get(key);
        if (names == null && key.contains(separator)) {
            names = key.toLowerCase().split(separator);
            nameMap.put(key, names);
        }

        if (names != null && names.length > 0) {
            tempMap.clear();
            tempMap.putAll(rootMap);
            //这块如果复写了separator属性的话split后的names数组长度为1会跳过这一步循环，防止解析出错
            for (int i = 0; i < names.length - 1; i++) {
                Object o = tempMap.get(names[i]);
                if (o instanceof Map) {
                    tempMap.clear();
                    tempMap.putAll((Map<String, Object>) o);
                } else {
                    throw new IOException("Property '" + names[i] + "' is not embedded format");
                }
            }
            Object finalObject = tempMap.get(names[names.length - 1]);
            return objToString(finalObject);
        }

        return StringUtils.EMPTY;
    }

```
