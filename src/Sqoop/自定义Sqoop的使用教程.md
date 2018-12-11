---
title: 自定义扩展Sqoop的使用教程
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]

# 开发API参考

## 外部API
Sqoop通过配置自动生成把关系型数据库导入Hadoop系统的类。该类包含导入Hadoop的每个字段。该类的实例保存表的一行数据。生成的类实现Hadoop中使用的序列化Api，即 **Writable** 和 **DBWritable** 接口。以及其他方法：
- 一个解释分隔文本字段的parse()方法
- 一个toString()方法，用于保留用户选择的分隔符

保证在生成的类中完整的方法在抽象类中指定:`org.apache.sqoop.lib.SqoopRecord`

SqoopRecord的实例可能依赖于Sqoop的公共API即：`org.apache.sqoop.lib.*` 
Sqoop的客户端不需要直接与这些类中的任何一个进行交互，尽管Sqoop生成的类将依赖于它们。
RecordParser类将使用可控的分隔符和引号字符将一行文本解析为字段列表。
静态FieldFormatter类提供了一种方法，用于处理将在SqoopRecord.toString()实现中使用的字段中的字符的引用和转义
ResultSet和PreparedStatement对象以及SqoopRecords之间的数据编组是通过JdbcWritableBridge完成的。
BigDecimalSerializer包含一对方法，这些方法有助于在Writable接口上序列化BigDecimal对象


## 开发环境的准备

官方的源码可以进行阅读查看在对其编译比较麻烦，采用的是把集群上的sqoop-1.4.6.2.6.5.0-292.jar来作为依赖开发。

 1. 先把jar文件添加的本地的maven库中。

 2. 在本机的命令行环境运行

``` shell
 mvn install:install-file -DgroupId=com.ambari.sqoop -DartifactId=sqoop -Dversion=1.4.6 -Dpackaging=jar -Dfile=D:\Develop\repository\sqoop\sqoop-1.4.6.2.6.5.0-292.jar
```

![添加成功界面](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544078894416.png)

相关参数简介:
 |属性|描述|
 |---|---|
 |-DgroupId|项目包名|
 |-DartifactId|工程名|
 |-Dversion|版本号|
|-Dpackaging |表示文件的类型|
|-Dfile |表示所有添加文件的绝对路径|


3. 依赖的pom文件

``` xml
 		 <dependency>
            <groupId>com.ambari.sqoop</groupId>
            <artifactId>sqoop</artifactId>
            <version>1.4.6</version>
        </dependency>
```

# 开发Sqoop插件

Sqoop允许用户开发自己的插件。用户可以将他们的插件开发为单独的jar在$SQOOP_LIB中部署它们并使用Sqoop注册。Sqoop架构也是一个基于插件的架构，并且所有的内部工具（导入，导出，合并等）都是作为插件得到支持。用户可以开发自定义工具插件。一旦部署并注册到Sqoop，这些插件就可以像任何其他内部工具一样工作。运行sqoop help命令时，它们也会在工具中列出。

## BaseSqoopTool - 用户定义工具的基类

### 继承与重写

BaseSqoopTool是所有Sqoop工具的基类。如果要开发自定义工具，则需要从BaseSqoopTool继承工具并重写以下方法：

- `public int run(SqoopOptions options)` ：这是该工具的主要方法，并充当自定义工具的执行入口点。
- `public void configureOptions(ToolOptions toolOptions)`：配置我们希望接收的命令行参数。您还可以指定所有命令行参数的描述。当用户执行时 sqoop help <your tool>，将以该方法提供的信息输出给用户。
- `public void applyOptions(CommandLine in, SqoopOptions out)` ：解析所有选项并填充SqoopOptions，它在完成执行期间充当数据传输对象。
- `public void validateOptions(SqoopOptions options)` ：提供您的选项所需的任何验证。

### 支持用户定义自定义选项

Sqoop解析用户传递的参数并存储在SqoopOptions对象中。然后，此对象充当数据传输对象。在运行实际的MapReduce，MapReduce阶段甚至后处理阶段之前，此对象被传递到处理的各个阶段。用户自定义了新的工具就会产生一些新的选项，这些选项不会映射到SqoopOptions类的任何现有成员。用户可以向SqoopOption类添加新成员，这意味着用户必须在sqoop中进行更改并对其进行编译，这对所有用户来说都是不可能的。其他选择是使用extraArgs会员。这是一个字符串数组，其中包含可以直接传递给第三方工具（如mysqldump等）的第三方工具选项。此数组字符串每次都需要解析才能理解参数。支持用户定义工具的自定义选项的最优雅方式是 customToolOptionsmap。这是SqoopOption类的映射成员。开发人员可以解析用户定义的参数，并使用适当的键/值对填充此映射。当SqoopOption对象被传递到处理的各个阶段时，这些值将随时可用，并且每次访问都不需要解析。
示例如下：
- `--hbase-col`
- `--hdfs-line-separator`
- `--hbase-rowkey-separator`
SqoopOption对象中没有这些选项可用。Tool Developer可以覆盖该applyOptions方法，在此方法中，可以在customToolOptions映射中解析和填充用户选项。完成后，SqoopOption对象可以在整个程序中传递，这些值将可供用户使用。


``` java
public static final String HBASE_COL = "hbase-col";
public static final String HDFS_LINE_SEPARATOR = "hdfs-line-separator";
public static final String HBASE_ROWKEY_SEPARATOR = "hbase-rowkey-separator";
```

下面是解析上述选项并填充customToolOptions映射的示例applyOptions示例：

``` java
	@Override
    public void applyOptions(CommandLine in, SqoopOptions out) throws 	SqoopOptions.InvalidOptionsException {
        try {
            Map<String, String> optionsMap = new HashMap<String, String>();
            super.applyOptions(in, out);
            applyHBaseOptions(in, out);
            if (in.hasOption(EXPORT_PATH_ARG)) {
                out.setExportDir(in.getOptionValue(EXPORT_PATH_ARG));
            }
            if (in.hasOption(HDFS_LINE_SEPARATOR)) {
                optionsMap.put(HDFS_LINE_SEPARATOR, in.getOptionValue(HDFS_LINE_SEPARATOR));
            }
            if (in.hasOption(HBASE_COL)) {
                optionsMap.put(HBASE_COL, in.getOptionValue(HBASE_COL));
            }
            if (in.hasOption(HBASE_ROWKEY_SEPARATOR)) {
                optionsMap.put(HBASE_ROWKEY_SEPARATOR, in.getOptionValue(HBASE_ROWKEY_SEPARATOR));
            }
            if (out.getCustomToolOptions() == null) {
                out.setCustomToolOptions(optionsMap);
            }
        } catch (NumberFormatException nfe) {
            throw new SqoopOptions.InvalidOptionsException("Error: expected numeric argument.\n Try --help for usage.");
        }
    }
```
配置我们希望接收的命令行参数。您还可以指定所有命令行参数的描述。

``` java
	@Override
    public void configureOptions(ToolOptions toolOptions) {
        super.configureOptions(toolOptions);
        toolOptions.addUniqueOptions(getHBaseOptions());
        RelatedOptions formatOpts = new RelatedOptions(
                "设置相关mapreduce的参数");
        formatOpts.addOption(OptionBuilder.withArgName("dir")
                .hasArg()
                .withDescription("要导出数据的hdfs地址")
                .withLongOpt(EXPORT_PATH_ARG)
                .create());
        formatOpts.addOption(OptionBuilder.withArgName("char")
                .hasArg()
                .withDescription("hdfs数据的行分隔符")
                .withLongOpt(HDFS_LINE_SEPARATOR)
                .create());
        formatOpts.addOption(OptionBuilder.withArgName("char")
                .hasArg()
                .withDescription("hbase的列簇")
                .withLongOpt(HBASE_COL)
                .create());
        formatOpts.addOption(OptionBuilder.withArgName("char")
                .hasArg()
                .withDescription("hbase的复合rowkey的字段分隔符")
                .withLongOpt(HBASE_ROWKEY_SEPARATOR)
                .create());
        toolOptions.addUniqueOptions(formatOpts);
    }
```








## ToolPlugin插件的基类


开发了Sqoop的扩展工具,你需要使用插件类包装并使用Sqoop注册该插件类。您的插件类应该从org.apache.sqoop.tool.ToolPlugin 和重写getTools()方法扩展 。
示例如下：

``` scala
import org.apache.sqoop.tool.ToolDesc;
import org.apache.sqoop.tool.ToolPlugin;
import java.util.Collections;
import java.util.List;
public class ExtendHBasePlugin extends ToolPlugin {
    @Override
    public List<ToolDesc> getTools() {
        return Collections
                .singletonList(new ToolDesc("extendHbase", ExtendHBase.class, "HDFS上的数据导入到HBase"));
    }
}
```

### 注册用户的自定义插件

 1. 把自定义的jar文件拷贝到相关集群的./Sqoop/lib目录，并在sqoop-site.xml进行注册。

``` xml
<property>
    <name>sqoop.tool.plugins</name>
    <value>org.apache.sqoop.extend.ExtendHBasePlugin</value>
    <description>A comma-delimited list of ToolPlugin implementations
      which are consulted, in order, to register SqoopTool instances which
      allow third-party tools to be used.
    </description>
  </property>
```
注册结束后再命令行输入：sqoop help 显示自定义的命令即注册成功。

![添加成功界面](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543979172525.png)

## Sqoop脚本示例


``` shell
sqoop extendHbase \
    --hbase-table mytable \
    --column-family i \
    --hbase-row-key GDDM,GDXM \
    --export-dir /yss/guzhi/interface/gh.tsv \
    --hbase-rowkey-separator _ \
    --hdfs-line-separator \\t \
    --hbase-col GDDM,GDXM,BCRQ,CJBH,GSDM,CJSL,BCYE,ZQDM,SBSJ,CJSJ,CJJG,CJJE,SQBH,BS,MJBH
```

## 相关配置的简介


|属性|描述|
|---|---|
|extendHbase|导出HDFS上的数据到HBase的指令|
|hbase-table|要导入的HBase表名称,需要提前建表不支持自动建表|
|column-family|HBase表的列簇名称|
|hbase-row-key|HBase表Rowkey的设置,如果是指定多个列作为RowKey,彼此间的分隔符采用逗号间隔|
|export-dir|要导出的HDFS上的文件|
|hbase-rowkey-separator|如果RowKey采用的是复合列组合的方式的默认采用的分隔符是下划线,可以传递参数替换|
|hdfs-line-separator|HDFS上的文件数据行的分隔符|
|hbase-col|HBase的列名称与HDFS上的数据列一一对应,彼此之间采用逗号分隔|

==注意==：在设置RowKey时多个字段必须在hbase-col设置中必须包含。