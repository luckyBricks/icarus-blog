---
title: 使用CumulusCI部署Salesforce.org开源项目(非托管软件包)
date: 2021-03-08 19:02:26
categories:
    - CRM
---

## 前提概要

`Education Data Architecture(EDA)`是由Salesforce.org组织开发的开源项目，通过托管软件包的形式发布在AppExchange之中。EDA每两个礼拜（文档中记载是Biweekly）向所有安装了EDA托管软件包的组织/沙盒推送最新版本。

但是，要将EDA进行集成开发时，自动更新功能很可能覆盖掉自行开发的代码。虽然所有由Salesforce.org创立的开源项目都是**可选升级**，即每次有新版本推送时都需要管理员手动确认是否更新，但是以防万一，还是对特定的EDA版本进行非托管部署比较安全。

> **托管软件包形式 Managed Packages**：发布于App Exchange中、会自动向安装了的组织推送升级的软件包
>
> **非托管软件包 Unmanaged Packages**：将开源的托管软件包的特定版本直接安装到Org中

Salesforce.org对EDA、NPSP等开源项目都推荐采用`CumulusCI`进行开发与管理，`CumulusCI`是由Salesforce提供的一款持续集成工具。对于类似于EDA、NPSP的大型开源项目，其源码（指SFDX项目中）不可能有完整的依赖，因此必须通过CumulusCI进行托管。

## 环境准备

1. VScode，需要已经配置好SFDX项目的开发环境
2. Python Environment，要求版本3.8.0以上
3. Cumulus CI，官方推荐使用`pipx`进行全局托管
4. Salesforce组织，要求是Developer版本或者Enterprise版本
5. git与GitHub账号

## 详细步骤

这里只记录可以成功将`EDA 1.111`版本以非托管形式部署到Org中的办法，不对原理进行详细介绍。

1. `git clone https://github.com/SalesforceFoundation/EDA`，将最新版本的EDA源码Clone到本地，删除掉其中的`Feature Parameters`后，将git库加回自己的Github中。

2. VScode打开该文件夹，运行`SFDX: Create Project`命令，要求创立`Empty Templete`并选择Overwrite掉现有工程中各个组件的config；运行`SFDX: Authorize an Org`，链接到需要部署的Org中

3. 在进行部署前，要确认组织中的`Account`对象至少有一条记录

4. 编辑根目录下的`cumulusci.yml`文件如图，修改`project→dependencies`一节，保存编辑

   ![修改成自定义存储库](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/image-20210403002447907.png)

5. 在根目录终端中输入`cci org connect --org <要部署的组织略称>`，将CumulusCI与要部署的组织相链接

6. 在根目录终端中输入`cci task list`，确认持续集成任务中有`update_dependencies`一项

   ![update_dependencies](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210308183915568.png)

7. 在根目录终端中依次运行如下四个流程

   ```bash
   # 安装EDA必须的Org环境配置
   cci run task deploy_pre --org <要部署的组织略称>
   # 以非托管形式部署元数据
   cci run flow deploy_unmanaged --org <要部署的组织略称>
   # 安装所有EDA依赖，包括deploy_post的部分
   cci run flow dependencies --org <要部署的组织略称> 
   # 进行post-install阶段的部署
   cci run flow config_dev --org <要部署的组织略称> 
   ```

8. 完成EDA的非托管部署

## 原理简要说明

EDA是一个使用`CumulusCI`托管的项目，按照Salesforce软件包开发模型的设计思想（简要介绍可以翻看我另一篇博客），管理EDA在组织中开发的全面生命周期。通过查看EDA的task与flow定义，可以确定`update_dependencies`这一task即拥有将项目中`unpackaged/pre`目录中的元数据部署到组织中的能力，并且在安装完EDA项目本身依赖后可以部署`unpackaged/post`中的依赖；通过在`cumulusci.yml`中将项目的依赖源设置成unmanaged，确保软件包是以非托管方式推送到组织中的。

之所以不使用`CumulusCI`中的`dev_org`这一flow，是因为它部署的元数据并非推送的软件包中的元数据。

## 附录

如果可能，还是建议使用托管软件包的形式按照EDA，再通过`CumulusCI`建立并维护EDA的集成项目，使用EDA设计的TDTM开发模型完成部分侵入式更改，能少写很多Trigger。