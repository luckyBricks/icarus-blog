---
title: DataLoader操作手册
date: 2021-04-02 23:59:55
categories:
    - CRM
---
DataLoader是Salesforce官方提供的一款工具，用于本地大批量数据与Salesforce组织之间的加载和导入。除了项目起始需要导入大量基础数据外，也可以利用DataLoader进行数据的批处理导入工作。

## 连接Org，并设定字段映射

1. 打开DataLoader，选择insert

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343374568.jpg)

2. 根据上图选择需要上传数据的对象，以及需要上传的文件

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343334546.jpg)

3. 点击NEXT，会显示CSV里面的数据条数，点击OK

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343389093.jpg)

4. 从后续步骤选择Create or Edit Map

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343398386.jpg)

5. 点击Auto-Match Fields to Columns

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343409079.jpg)

6. 其他未自动匹配字段拖到下面，手动匹配

![img](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343420128.jpg)

7. 点击Save Mapping，将Mapping文件保存到指定的路径

## 获取DataLoader中保存的登录密钥

登录时，不需要暴露登录账户与密码到Mapping文件中，可以通过如下方式找出DataLoader中存放的登录加密文件

1. 找到DataLoader对应的二进制bin文件，右键属性打开后，赋值bin文件的路径

![1586334751(1)](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002.png)

2. 打开CMD，切换到bin目录下，依次输入下面命令

```bash
# 先获取一个key加密文件
encrypt.bat --k <KEY FILE PATH>
# 利用登录密码+Auth Token获取登录密钥
encrypy.bat -e <PASSWORD+AUTH TOKEN> <KEY FILE PATH>
```

![1586335555(1)](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343455902.png)

3. 保存好key文件和加密密码文本，接下来设置配置文件`process-conf.xml`

![1586335796(1)](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343526992.png)

![1586336430(1)](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343538655.png)

![1586336613(1)](D:\01-DOCS\SFDC_Docs\system integration\DataLoader操作手册.assets\clip_image002-1617343545767.png)

4. 启动DataLoader，使用如下命令

```bash
process.bat "<FILE PATH OF process-conf.xml>" <PROPERTY NAME>
```

![1586337094(1)](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/clip_image002-1617343553404.png)

**使用如上命令，就可以利用Windows的定时任务执行这个批处理上传了**