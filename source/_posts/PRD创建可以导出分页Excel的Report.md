---
title: PRD创建可以导出分页Excel的Report
date: 2020-09-15 17:24:16
categories:
	- BI
---

在Pentaho Report Designer中没有提供原生导出Excel分页的设定，但是提供了可以设定导出布局的参数，在这个参数中进行相关的设定可以导出分页Excel。
<!-- more -->
## 页面布局的设定

一份Report的布局是一个xml文件，服从Band（带）的布局关系，子组件可以继承父组件的参数、属性，子布局可以继承父布局的样式。创建一个如下图的布局目录：

![布局目录树](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/1.png)

其中在**详情**带中新建一个子带作为分页带的容器，新建三个分页带在容器中。

![带的布局设定](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/2.png)

**务必要将所以的带均设计为block布局**

在三个带中插入**标签**和**子报表**元素

![插入元素](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/3.png)

## 输出Excel参数的设定

在主报表的查询中插入一个新的**Table**参数，参数值设定如下，Value的部分是在Report的输出画面的按钮中显示的内容：

![Value的值为输出Excel的分页名](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/4.png)

在主报表的Parameter中新建一个单选按钮组，其查询为上一步设定的Table参数

![一定注意值类型为整型](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/5.png)

在分页带的样式中找到visible属性，在其后输入表达式

![](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/6.png)

```vbscript
=IF([PRM_SHEET]=2;"true";IF([PRM_SHEET]=0;"true";"false"))
```

其中第一个分页参数根据分页带的顺序而设定，同时设定Excel显示的SheetName

![](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/7.png)

## 空查询和组件设定

在分页带中使用子报表时，如果没有在主报表中设定查询，则会导致输出画面重复多次，PRD7和PRD9都有这个问题。原因我暂时没找到，因此需要在主报表中设置一个空查询避免出错。

![插入一个Sequence Generator查询](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/8.png)

![空查询的设定如下](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/9.png)

## 效果

布局的效果如下，其中三个子报表带会在Excel中分页输出

![](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/10.png)

输出Excel的效果如下

![报表带1](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/11.png)

![报表带2](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/12.png)

![报表带3](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/%E6%96%B0%E5%BB%BA%E6%96%87%E4%BB%B6%E5%A4%B9/13.png)

三个子报表有独立的查询，互不影响，可以通过主报表Import全局的查询参数。