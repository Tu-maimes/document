---
title: Sqoop
tags: 作者:汪帅
grammar_cjkRuby: true
---

# Sqoop的运行流程

 1. Sqoop的主类和实现工具会启动一个新的实例ToolRunner,Sqoop的第一个参数决定了Sqoop要执行的功能(导入、导出、编译代码等)。Sqoop本身驱动来执行用户的请求。
 2. SqoopTool将解析其余参数,在SqoopOptions类中设置适当的字段.然后它将运行。
 3. 在SqoopTool的==run()==方法中,执行导入或导出以及其他正确的操作。
 4. ConnManager是基于SqoopOPtions中的数据来实例化的。ConnFactory用于从ManagerFactory获取ConnManager
 5. 导入导出以及其他大型数据运动通常运行MapReduce来作业，以并行，可靠的方式。导入并不特别需要通过MapReduce作业运行，ConnManager.importTable()方法来确定如何最好地运行导入。
 6. 除了由CompilationManager和ClassWriter完成的代码生成之外，每个主要操作实际上都由ConnMananger控制。（两者都在 com.cloudera.sqoop.orm包中。）也可以通过com.cloudera.sqoop.hive导入Hive。在importTable()完成之后导入HiveImport类。这与使用的ConnManager实现无关。
 7. ConnManager的importTable()方法接收一个ImportJobContext类型的参数，该参数包含方法的参数。将来可以使用附加参数扩展该类，这些参数可以进一步指导导入操作。类似地，exportTable()方法接收ExportJobContext类型的参数。这些类包含要导入/导出的表的名称、对SqoopOptions对象的引用以及其他相关数据。