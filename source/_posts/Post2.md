---
title: Salesforce 配置备忘
date: 2020-07-05 02:11:04
categories:
    - CRM
---
##### 字段值之间相互调用

> 使用公式编辑，如`星级 CustomerLevel` 和显示的星星`客户星级 ShowLevel` 
>
> 后一字段属性为公式，显示状态为文本，调用静态资源
>
> 其中公式部分使用的 `TEXT()` 方法调用的是 `CustomerLevel` 中API的值，因此在定义`CustomerLevel` 时务必注意**显示的值** 和**API的值** 二者之间的区别
>
> 公式中`IMAGE()`方法使用的静态资源在**设置->静态资源->查看文件**中可以查询到具体的链接，方便使用
<!-- more -->
##### 页面中插入新的选项卡

> 设置->应用程序->应用程序管理器
>
> 新插入的选项卡需确定所在的应用程序种类，在编辑中查找
>
> 以`潜在客户` 为例，属于`销售->导航项目`类应用程序下的功能，在其中添加入所选项目即可

```markdown
注意：
如果配置好后页面上仍然没有显示，那么要在简档中查看确认用户是否默认显示此选项卡“设置->用户->简档->选项卡设置”
```



## APEX学习笔记

### 数据类型

1. 原始数据类型Primitive( `Integer` `Double` `Long` `Date` `Datetime` `String` `ID or Boolean`)
2. 集合类Collections  `Lists`  `Sets`  `Maps` 
3. 自定义对象（包括继承的默认对象）`sObject`
4. 枚举 `Enums`
5. 类、对象和接口 `Classes` ` Objects`  `Interfaces`

### 变量

1. 从声明开始便生效， 不能重复声明
2. 方法内声明的变量均为局部变量
3. 类变量可通过类访问



#### 字符串

~~~java
public Boolean contains (String substring) //此串中是否包substring
public Boolean equals (Object string) //考虑大小写，判断二进制值是否相等
public Boolean equalsIgnoreCase(String stringtoCompare)//忽略大小写
public String remove(String stringToRemove)//区分大小写，删除指定部分
public String removeEndIgnoreCase(String stringToRemove)//删除末尾部分的指定内容，不区分大小写
public
~~~



# Developer Beginner

## Platform Development Basics

### Develop Without Code

When you look at data in Salesforce, you might think that you're looking at a user interface sitting on top of a run-of-the-mill relational database. But what you’re actually looking at is an abstraction of the database that’s driven by the platform’s metadata-aware architecture.

In this abstraction, objects are our database tables. The fields on those objects are columns, and records are rows in the database. This analogy is true both for **standard objects** that come with Salesforce by default and **custom objects** that you build yourself.

### Lists, Sets and Maps

1. **Lists** : a group of data elements ordered

   ~~~java
   public class Flowers{
       public static void insertFlower(){
           List<String> allFlowers = new List<String>();
           allFlowers.add('Rose');
           allFlowers.add('Yuri');
           allFlowers.add('Rose');
           system.debug(allFlowers);
       }
   }
   ~~~

   

2. **Sets** : elements are unordered, can't have any duplicates ( If you try to add an element already in the set, you don't get error, but the new value is not added to the set. )

   ~~~java
   public class Tea{
       public static void orderTea(){
           Set<String> teaTypes = new Set <String>();
           teaTypes.add('Black');
           teaTypes.add('White');
           teaTypes.add('Herbal');
           system.debug(teaTypes);
       }
   }
   ~~~

3. **Maps** : each items in a map has two parts: a key and a value, known as a *key-value pair*. Key and value can be any data type. Although each key is unique, values **can be repeated **within a map.

   ~~~java
   mapName.put(key,value); //add new item in Map
   
   public class Tea{
       public static void orderTea(){
           Map <String, String> teaTypes = new Map <String, String>();
           teaTypes.put('Black', 'Earthy');
           teaTypes.put('White', 'Sweet');
           teaTypes.put('Herbal', 'Sweet');
           system.debug(teaTypes);
       
       }
   }
   ~~~


### Work with Schema Builder

Schema Builder is an useful way to visualize the data model, it's a handy tool to explaining the way data flows throughout in system.