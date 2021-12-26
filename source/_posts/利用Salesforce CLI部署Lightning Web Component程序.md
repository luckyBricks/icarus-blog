---
title: 利用Salesforce CLI部署Lightning Web Component程序
date: 2021-02-08 17:34:04
categories:
    - CRM
---
部署并测试一个Lightning Web Component程序需要经过三个步骤：

1. 获取源码、通过测试
2. 建立组织/临时组织
3. 导入数据/测试数据
<!-- more -->
## 获取通过测试的源码

在VScode中进行LWC的开发，需要先部署好Salesforce的VScode插件。源码由特定的VCS系统管理。这里记录的是在Trailhead中的示例程序Dreamhouse的部署。

```bash
git clone https://github.com/trailheadapps/dreamhouse-lwc.git
cd dreamhouse-lwc
git checkout -b my_branch
```

为了方便源码的管理，无论是在示例程序还是在实际开发中最好都将代码检出到自己的分支中，便于版本管理和合并。

## 建立组织/临时组织

将程序部署到临时组织中先进行测试和调试，避免对生成环境造成影响。

```bash
sfdx force:org:create -s -f config/project-scratch-def.json -a dreamhouse-org
# 这条语句直接执行后，会出现如下报错
# ERROR running force:org:create:  You do not have access to the [ScratchOrgInfo] object
# 提示我们没有访问临时组织的权限
```

碰到这个情况时，需要先将测试环境的`DevHub`功能打开

![image-20210205160208167](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205160208167.png)

修改成功后，重新运行上述语句之前，需要将既有的测试组织都清空

```bash
# 依次执行这三条
sfdx force:org:list --clean
sfdx force:org:create -s -f config/project-scratch-def.json -a dreamhouse-org
sfdx force:org:list --all 
```

最后显示的结果如下，`(D)`标记的环境代表开启了`DevHub`功能

![image-20210205160604681](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205160604681.png)

```bash
sfdx force:org:open # 打开组织权限集的页面
sfdx force:source:push # 推送本地源码到测试环境中
```

> 利用CLI的功能可以在每次推送源码时比对本地更改和临时组织中的区别，如下例所示
>
> ```bash
> $ sfdx force:source:status
> STATE                     FULL NAME    TYPE        PROJECT PATH
> ─────                  ──────   ───       ────────── 
> Local Deleted             MyClass      ApexClass   /MyClass.cls-meta.xml
> Local Deleted             MyClass      ApexClass   /MyClass.cls.xml
> Local Add                 OtherClass   ApexClass   /OtherClass.cls-meta.xml
> Local Add                 OtherClass   ApexClass   /OtherClass.cls.xml
> Local Add                 Event        QuickAction /Event.quickAction-meta.xml
> Remote Deleted            TempClass    ApexClass   /TempClass.cls-meta.xml
> Remote Deleted            TempClass    ApexClass   /TempClass.cls.xml
> Remote Changed (Conflict) NewClass     ApexClass   /NewClass.cls-meta.xml
> Remote Changed (Conflict) NewClass     ApexClass   /NewClass.cls.xml
> ```
>
> 

## 导入用户数据、测试数据和分配权限集

```bash
# 允许user访问Dreamhouse应用
sfdx force:user:permset:assign -n Dreamhouse
```

出现如下报错信息

![image-20210205162705704](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205162705704.png)

第一次注册权限集会提示`临时组织中没有标记为(D)的权限集`，需要将`user`这个权限集先装入临时组织中

```bash
sfdx plugins:install user # 将user权限集装入当前的临时org中
```

再次注册权限集，成功

![image-20210205163009625](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205163009625.png)

将测试记录数据装入Dreamhouse应用中，成功。

```bash
sfdx force:data:tree:import --plan data/sample-data-plan.json
```

![image-20210205163256014](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205163256014.png)

如果要将真实数据批量导入组织中，需要使用`sObject Tree Save API`；输入`sfdx force:org:open`打开组织。

![image-20210205163411385](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210205163411385.png)