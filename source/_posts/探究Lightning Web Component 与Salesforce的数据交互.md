---
title: 探究Lightning Web Component 与Salesforce的数据交互
date: 2021-02-26 21:32:16
categories:
    - CRM

---

Salesforce为LWC组件提供了丰富的API用于与调用Salesforce进行数据调用，这篇文章进行简要的学习备忘。
<!-- more -->
## LWC编程模型与传统前后端分离项目模型比较

与一般的Spring Boot程序不同，Salesforce DX项目中不存在DTO层的概念，因此无需手动建立DTO层来Mapping Apex对象与Salesforce组织中的`sObjects`。但这并不意味着必须人工管理`Apex对象`与`sObjects`之间的关系。在初始化一个`sfdx项目`，并将其与组织进行链接后，只要先将组织中`sObject`的元数据拉取到项目中，就可以在使用过程中方便地引用。

![刷新SFDX项目中的sObject定义](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210226162841895.png)

> 注意，在对`sObjects`的元数据进行任何更改后，都应即使将最新的定义拉取回本地SFDX项目中。

可以在刷新后的SFDX项目文件夹下找到存有元数据定义的TypeScript文件，这便在本地SFDX项目中充当了DTO层的存在，以便在进行LWC的开发过程中提供参考。

![LWC引用元数据定义](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210226163257172.png)

![Apex元数据定义](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210226170332365.png)

SFDX项目中，无需手动管理元数据DTO的原因就在这里，已经定义好了。LWC开发模型中数据交互通过Lightning Data Service实现，其包括了一个`@wire`JS装饰器和配套开发好的一套CURD API，存放在`'lightning'`目录之中。使用Lightning Data Servicehttps://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.reference_ui_api需要与Lightning基本UI组件配合，就可以在无需重写Apex Controller的情况下方便取用/改写Salesforce组织中存放的数据。如果基本CURD无法满足的LWC需求，就可以通过自行定义的Apex Controller进行数据的处理和交互。
