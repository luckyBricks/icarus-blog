---
title: IIS--URL Rewrite功能的debug与日志展现
date: 2021-12-27 18:04:07
tags:
categories:
    - IIS
---
使用URL Rewrite模块时，有时希望查看详细的debug日志，以确认配置是否正常工作。URL Rewrite模块可以与Failed Request Tracing 模块共同工作。
<!-- more -->
> 如果在IIS安装时先安装了Failed Request Tracing 模块，则不方便在UI侧进行URL Rewrite的追踪配置，可以使用**添加/删除Windows功能**重装一下 

## URL重写测试页

1. 在`%SystemDrive%\inetpub\wwwroot`中新建如下页面，命名为`article.aspx`

   ```aspx
   <%@ Page Language="C#" %>
       <!DOCTYPE html
           PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
       <html xmlns="http://www.w3.org/1999/xhtml">
   
       <head>
           <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
           <title>URL Rewrite Module Test</title>
       </head>
   
       <body>
           <h1>URL Rewrite Module Test Page</h1>
           <table>
               <tr>
                   <th>Server Variable</th>
                   <th>Value</th>
               </tr>
               <tr>
                   <td>Original URL: </td>
                   <td>
                       <%= Request.ServerVariables["HTTP_X_ORIGINAL_URL"] %>
                   </td>
               </tr>
               <tr>
                   <td>Final URL: </td>
                   <td>
                       <%= Request.ServerVariables["SCRIPT_NAME"] + "?" + Request.ServerVariables["QUERY_STRING"] %>
                   </td>
               </tr>
           </table>
       </body>
   
       </html>
   ```

2. 建立后，访问`http://localhost/article.aspx`进行访问

3. 配置URL重写功能，在`web.config`的`<system.webserver>`节下加入如下入站规则

   ```xml
   <system.webServer>
       <rewrite>
           <rules>
               <rule name="Fail bad requests">
                   <match url="." />
                   <conditions>
                       <add input="{HTTP_HOST}" negate="true" pattern="localhost" />
                   </conditions>
                   <action type="AbortRequest" />
               </rule>
               <rule name="Rewrite to article.aspx">
                   <match url="^article/([0-9]+)/([_0-9a-z-]+)" />
                   <action type="Rewrite" url="article.aspx?id={R:1}&amp;title={R:2}" />
               </rule>
           </rules>
       </rewrite>
   </system.webServer>
   ```

4. 尝试访问`http://localhost/article/234/some-title`，可以看到访问的URL被重写至`http://localhost/article.aspx?id=234&title=some-title`

   ![测试页面](/images/image-20211007110538631.png)

## Failed Request Tracing的配置

1. 在发布的网站中选择添加Failed Request Tracing配置

   ![配置页面](/images/image-20211007111205088.png)

2. 在配置向导中选择追踪所有的文件请求

   ![All content](/images/image-20211007111426858.png)

3. 自定义希望追踪的状态码

   ![Status Code](/images/image-20211007111520480.png)

4. 在Log Level为verbose的情况下，最好是只选择Rewrite模块这一条的Log Flag

   ![Log-Provider Rewrite](/images/image-20211007111719309.png)

5. 建立好之后，要在FRT配置页的Actions一栏，确认Request Tracing是Enable状态，同时确认日志文件的存放位置

   ![Enable failed request tracing](/images/image-20211007111911750.png)

6. 对应网站的`web.config`文件中，会多出如下一节。此时访问测试页，即可在对应的目录下看到`.xml`和`.xsl`的日志文件了

   ```xml
   <system.webServer>
       <tracing>
           <traceFailedRequests>
               <add path="*">
                   <traceAreas>
                       <add provider="WWW Server" areas="Rewrite" verbosity="Verbose" />
                   </traceAreas>
                   <failureDefinitions statusCodes="200-399" />
               </add>
           </traceFailedRequests>
       </tracing>
   </system.webServer>
   ```

   

## 其他

当希望查看日志时，需要给Edge加上`--allow-file-access-from-files`的启动参数，以绕开Chromium的安全策略，打开本地文件

![添加启动参数](/images/image-20211007112450180.png)

之后日志即可正常浏览

![Failed Request Tracing Log](/images/image-20211007112539300.png)

