---
typora-copy-images-to: img
---

# Python For Data Science

[TOC]

## 1. 配置系统

- Python
- JDK
- 创建`C:\Hadoop\bin`
- 在这里下载windows版的hadoop https://github.com/steveloughran/winutils 拷贝winutils到`C:\Hadoop\bin`下面
- 创建`HADOOP_HOME`环境变量，指向`C:\Hadoop`
- 创建`C:\temp\hive`文件夹
- 运行`c:\hadoop\bin\winutils chmod 777 \temp\hive`
- 下载Spark: https://spark.apache.org/downloads.html
- 解压下载的Spark的文件到`C:\SPARK`目录下，其它操作系统的放到`home`目录
- 创建`SPARK_HOME`，指向`C:\SPARK`
- 运行`c:\spark\bin\spark-shell`看看是否安装成功