---
title: Pentaho CTools 使用踩坑
date: 2020-07-03 16:20:37
categories:
    - BI
---

### 关于CTools和Pentaho其他套件的关系

1. Pentaho server中集成的CTools用于创建和浏览Dashboards和Reports，两种文件区别在于：前者可以设置组件的自动刷新，如大屏展板更新数据；后者使用PRD(Pentaho Reports Designer)设计并上传到Pentaho Server中，在浏览Reports时请求更新数据并渲染到静态的报告中。
<!-- more -->
2. CTools使用CDA的形式记录Dashboards和数据库链接的方式，是一串XML文件，可以在首页的`Create New`中建立一个新的数据连接，在XML中修改更方便。

### 关于数据连接的问题

1. 在进行报表设计的时候根据情况可以使用JDBC驱动让Dashboards和数据库进行连接，需要注意的是**使用JDBC的数据库最好唯一，在同一个Dashboard中，使用不同端口的JDBC链接串会不可避免地引发CORS错误。如果要这么使用，可以在lib中添加`cors-filter-1.7.jar`，`java-property-utils-1.9.jar`两个包，在Tomcat的`web.xml`中进行如下的配置** 

``````xml
 <filter>       
       <filter-name>CORS</filter-name>
       <filter-class>com.thetransactioncompany.cors.CORSFilter</filter-class>
       <init-param>
        <param-name>cors.allowOrigin</param-name>
           <param-value>*</param-value>
       </init-param>
       <init-param>
        <param-name>cors.supportedMethods</param-name>
           <param-value>GET, POST, HEAD, PUT, DELETE</param-value>
       </init-param>
       <init-param>
        <param-name>cors.supportedHeaders</param-name>
           <param-value>Accept, Origin, X-Requested-With, Content-Type, Last-Modified</param-value>
       </init-param>
       <init-param>
           <param-name>cors.exposedHeaders</param-name>
           <param-value>Set-Cookie</param-value>
       </init-param>
       <init-param>
           <param-name>cors.supportsCredentials</param-name>
           <param-value>true</param-value>
       </init-param>
   </filter>
  
   <filter-mapping>
       <filter-name>CORS</filter-name>
       <url-pattern>/*</url-pattern>
   </filter-mapping>
``````

这是让Tomcat在启动时允许跨域请求的配置方法。

2. CDE使用JDBC方式链接MySQL数据库时，如果在`SQL Query`中有在查询时携带的变量，那么就要在JDBC的链接url中添加`generateSimpleParameterMetadata=true`这一参数。这是因为MySQL的5.7版JDBC驱动并不清楚CDE匹配好的参数所指向数据表的metadata元数据类型。添加这一参数后，拼凑好的SQL中参数所在的部分会被反射成Varchar类型，**在高频次查询的情景下性能开销可能过大**

3. 如果要使用的数据库的JDBC驱动比较新，就要在`server/pentaho-server/tomcat/lib`目录下将对应的JDBC的jar包放入。同理，使用Pentaho其他套件时需要更新JDBC的操作类似。**如果要使用SQL Server作为数据库，那么需要使用名为JTDS的工具。**参考这个链接https://help.pentaho.com/Documentation/9.0/Setup/Set_up_native_(JDBC)_or_OCI_data_connections_for_the_Pentaho_Server
4. 在配置数据源中的Parameters一项时，需要注意工具使用的数据类型和`SQL Query`从数据库中取出值的数据类型的转换关系，切不可出错。
5. 如果非要**通过传统的JDBC-ODBC桥方式连接SQL Server数据库**，有两点大坑：一是Pentaho Server要求的JVM环境要在Java8以上，而Java8的第一个版本开始就把JDBC-ODBC桥的方式弃用了，也就是说Java8环境本身不再支持ODBC，可以通过手动添加Java7的jdk中关于JDBC-ODBC桥的jar包实现；二是微软的JDBC-ODBC驱动依赖于手动添加的Visual Studio C++预编译环境，而大多数情况我们难以确定安装的数据库版本使用的ODBC桥驱动依赖于哪一个版本的VSC++预编译环境，就需要手动将13-17年的累积更新安装好才能正常运行。

### 关于CDE中Dashboard编辑的一些问题

1. CDE的可视化套件集成了名为CCC Charts的图表模板，这个东西是一个名为[Protovis.js](http://mbostock.github.io/protovis/)的可视化工具的加壳，这个Protovis在2011年就停止了更新，是大名鼎鼎的D3.js的前身。社区版的Pentaho不提供CTools及其相关可视化工具的文档，因此就无从知道每个组件中浅浅带过的DataSource参数中传入数据的结构，只能不断地尝试。大多数在Protovis的画廊中展现的形式都能使用CCC Charts实现，可以通过这个提供一些可视化展示的思路。官方给出了部分CCC Charts的图表展示效果，但是并不全，没有解释DataSource传入数据结构的相关问题。[CCC Playground](https://webdetails.github.io/ccc/#type=treemap&anchor=flare-library-modules)
2. 下面的图展示了CDE中组件的渲染生命周期，这个在打开Console之后也能看到。其中草绿色的方法能直接在组件的配置项中添加对应的代码，控制组件渲染过程中数据的更新和传递，一些常用的可以直接在配置项中修改，比如是否在打开页面时就加载。**输入的JS会被CDE构建掉，不支持ES6语法，不能使用箭头函数，我曾以为哪里错了半天没法运行**。黄色的方法可以在Dashboard中添加全局JS调用。

![CCC_Charts生命周期](Post1/PentahoCTools使用踩坑1.png)

3. Dashboard如果需要自动更新，可以在组件的`Refresh Period`属性中设置组件的更新时间，单位是秒，这个更新是从页面的静态缓存中提取数据，因此需要在全局JS中写好Ajax请求，并在组件中设置相应的listener，由Dashboard按照既定的时间向数据库请求数据。使用组件的局部刷新可以减少数据库的访问压力。如果需要将页面中所有组件的数据全部更新，按照一般思路，经常使用的`window.location.reload()`方法也可以刷新整个页面，但**会造成短暂的白屏**。在全局JS中加入如下方法可以让Pentaho使用内置的Ajax API刷新Dashboard中的全部组件数据。

```javascript
function updateAllDashboardComponents() {
    Dashboards.updateAll(Dashboards.components);
} 
```

### 关于页面布局

1. CDE中已经集成了Bootstrap前端框架，可以在创建页面布局的时候选择Bootstrap控件，也可以在设定好页面Dom后通过手动定义Css Class，在全局CSS中对相应的Bootstrap Class加以引用。
2. 页面布局的Row只是一个概念，如果在其中存放组件，调整Row的相关样式如`margin`,`padding`等与内容物相关的属性时不会生效，**因为Row不会被实际地渲染入页面Dom中**，只有将组件放入Row中的Column时，通过定义Column的样式才能对内容物产生作用。
3. 如果想对一些CDE组件的样式进行调整，如采用Bootstrap风格的按钮，**需要将组件放入Freeform中，并调整Freeform的样式**，直接放入Column是不会生效的。
4. Pentaho是可以加载Echarts和AntV等常用的可视化控件的，只需要在布局中加入HTML Object并将对应图表的Div放入其中，在全局JS中加载`echarts.min.js`和`antv.js`等就能使用了。图表需要的数据可以在Dashboard对应的CDA文件中找到，是一个JSON文件。在图表显示的JS中从这里访问数据即可。

![查询URL——API](Post1/PentahoCTools使用踩坑2.png)


