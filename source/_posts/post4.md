---
title: 将ETL工作流部署成自动运行的步骤
date: 2020-07-15 17:45:16
categories:
	- BI
---


## 自动运行的bat脚本编写
<!-- more -->
``` powershell
D:
cd D:\data-integration
kitchen.bat -rep=PentahoLocal -user=Admin -pass=password -dir=/home/admin -job=testJob1 -level=basic>D:\JOB.log
```

1. 首先跳转到存储**PDI**组件的目录中
2. 携带以上参数启动`kitchen.bat`文件，`-rep`是publish Job时kettle中记录连接Pentaho BI server的依赖库名；`-user`和`-pass`是登录Pentaho BI server的登陆账号和密码；`-dir`是存放publish的Job的目录；`-job`是运行转换Job的名称；`-level`是保存运行日志的级别日志存放位置



## 部署前的检查和确认

在将Kettle设计好的Job部署成自动运行的计划任务之前，要对部署的系统环境和转换任务文件做如下的两个确认

* Job和Trans发布的repository是否正确
* Pentaho BI server是否正常运行
* 转换的bat文件是否能正常运行

![保存在Pentaho BI server的文件](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/1.png)

## 将bat文件部署成计划任务自动运行

1. 打开**任务计划程序**，创建一个任务

![计划任务程序](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/2.png)

2. 输入任务名和任务简介

![创建任务](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/3.png)

3. 选择**触发器**选项卡，新建一个触发器，设定ETL转换的时间间隔

![新建触发器](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/4.png)

![触发器的参数设定](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/5.png)

4. 选择**操作**选项卡，新建一个定时操作，将工作流bat的位置路径放入其中

![新建操作](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/6.png)

![操作的设置](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post4/7.png)

