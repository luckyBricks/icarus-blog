---
title: Salesforce借助多种API与外部系统对接的汇总和样例
date: 2021-04-03 00:00:15
toc: true
categories:
    - CRM
---
Salesforce中本身提供了多种API共开发者和外部系统对接使用。面对不同的需求场景可以使用不同种类的对接方式。

<!-- more -->1. 数据从Salesforce导出
   1. 自定义开发，Salesforce建立REST Controller或SOAP Controller，外部系统通过轮询捕获数据
   2. Salesforce中建立Trigger，并在其中CallOut外部系统的REST Api，将数据变更推送到外部系统
   3. Saleforce中建立Streaming API，外部系统握手后监听该API，捕获实时的数据变更，这一点还可以实现不同Org之间的数据同步
2. 外部数据导入Salesforce
   1. 自定义开发，Salesforce建立REST Controller或SOAP Controller，外部系统推送格式化好的数据
   2. 大批量数据通过Bulk API，少量数据通过SOQL API都可以更新入Salesforce中
   3. 利用Salesforce Connect可以在不将外部数据上传入Salesforce中的前提下，直接操作外部数据，此时外部数据库表被视为External Object

下面给出几例，分别是1.1、1.2和1.3的代码实现



## Salesforce中建立Restful API的简要步骤

在Salesforce中构建一套Restful API很容易，基本与Spring Boot中Controller的写法类似，使用注解的方式构建基本的Http方法。Apex的包装类也可以作为方法的返回类型，对应的即为相应格式的json。参考如下写法，对Contact进行基本的操作。

```java
@RestResource(urlMapping='/Contact/*')
global with sharing class ContactManager {

    @HttpDelete
    global static String deleteRecordById(){
        String Id = RestContext.request.params.get('Id');
        List<Contact> contacts = [SELECT Id FROM Contact WHERE ID = :Id];
        delete contacts;
        return '删除成功';
    }

    @HttpGet
    global static List<Contact> getAllContacts(){
        List<Contact> allContacts = [SELECT Id,AccountId,Lastname FROM Contact];
        return allContacts;
    }

    @HttpPost
    global static Contact createNewContact(String name){
        Contact contact = new Contact();
        contact.Lastname = name;
        insert contact;
        return [SELECT Id,Name FROM Contact WHERE Lastname=:name];
    }
    
    @HttpPut
    global static String updateContactNameById(String Id, String newName){
        Contact contact = [SELECT Id, Lastname FROM Contact WHERE ID = :Id];
        contact.Lastname = newName;
        update contact;
        return '更新成功';
    }

}
```

构建好的API可以通过`https://<INSTANCE URI>/services/apexrest/<CLASS MAPPING ENTRANCE>`格式的路径进行访问。需要注意的是，所有访问实例内自定义API的来源也都需要鉴权，鉴权的方式可以根据获取API key/SSO/yaml等等Salesforce支持的形式。如下例即在Request Header中加入Bearer鉴权字符串的方法，其中的感叹号加了转义反斜杠进行转换。

```bash
curl https://<INSTANCE URI>/services/data/v51.0/ 
-H "Authorization: Bearer 00D50000000IehZ\!AQcAQH0dMHZfz972Szmpkb58urFRkgeBGsxL_QJWwYMfAbUeeG7c1E6
LYUfiDUkWe6H34r1AAwOR8B8fLEz6n04NPGRrq0FM"
```

> Salesforce官方提供了一个调试RestAPI的工具网站，参考[Workbench](https://workbench.developerforce.com/login.php)

## 在Apex Trigger中调用外部API的方法

在trigger中直接使用`HttpTemplate`对外部进行Callout是不被允许的，因此只能是将触发相关CRUD操作的Callout方法建立在单独的Apex类中，在trigger中引用这些方法。Callout方法必须加上`@future`注解。参考如下两例

```java
trigger BookTriggers on Book__c (after insert, after update, after delete,after undelete) {
    IntegrationTriggerMgmt__c bookTig = [select Id,Name,Active__c from IntegrationTriggerMgmt__c where Name='BookTriggers'];
    if (!bookTig.Active__c) {
        return;
    }

    if (Trigger.isAfter) {
        if (Trigger.isInsert) {
            List<BookClass> bL = new List<BookClass>();
            for (Book__c newBook : Trigger.new) {
                BookClass b = new BookClass(newBook.Id, newBook.Author__c, newBook.Name, newBook.Price__c);
                bL.add(b);
            }
            for (BookClass myb : bL) {
                String bString = JSON.serialize(myb);
                BookCallouts.insertBook(bString);
            }
        }
        else if(Trigger.isUndelete){
            List<BookClass> bL = new List<BookClass>();
            for (Book__c newBook : Trigger.new) {
                BookClass b = new BookClass(newBook.Id, newBook.Author__c, newBook.Name, newBook.Price__c);
                bL.add(b);
            }
            for (BookClass myb : bL) {
                String bString = JSON.serialize(myb);
                BookCallouts.insertBook(bString);
            }
        }
        else if(Trigger.isUpdate) {
            List<BookClass> bL = new List<BookClass>();
            for (Book__c newBook : Trigger.new) {
                BookClass b = new BookClass(newBook.Id, newBook.Author__c, newBook.Name, newBook.Price__c);
                bL.add(b);
            }     
            for (BookClass myb : bL) {
                String bString = JSON.serialize(myb);
                BookCallouts.updateBook(bString);
            }
        }
        else if(Trigger.isDelete){
            List<Book__c> listTriggerOld = Trigger.old;
            List<Book__c> listAllNew = [SELECT Id FROM Book__c WHERE Id != NULL];
            List<Book__c> deletedRows = [SELECT Id,Author__c,Name,Price__c FROM Book__c WHERE Id IN :listTriggerOld AND Id NOT IN :listAllNew ALL ROWS];
            List<BookClass> bL = new List<BookClass>();
            for (Book__c newBook : deletedRows) {
                BookClass b = new BookClass(newBook.Id, newBook.Author__c, newBook.Name, newBook.Price__c);
                bL.add(b);
            }
        
            for (BookClass myb : bL) {
                String bString = JSON.serialize(myb);
                BookCallouts.deleteBook(bString);
            }
            
        }
    }
}
```

下面是实现相关方法的`BookCallouts.cls`

```java
public class BookCallouts {

    @future(callout=true)
    public static void insertBook(String b) {
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('http://servicedevtest.natapp1.cc/book');
        request.setMethod('POST');
        request.setHeader('Content-Type', 'application/json;charset=UTF-8');
        request.setBody(b);
        HttpResponse response = http.send(request);
        if (response.getStatusCode() != 201) {
            //TODO：ErrorHandler还未定义
        }
    }

    @future(callout=true)
    public static void updateBook(String b) {
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('http://servicedevtest.natapp1.cc/book');
        request.setMethod('PUT');
        request.setHeader('Content-Type', 'application/json;charset=UTF-8');
        request.setBody(b);
        HttpResponse response = http.send(request);
        if (response.getStatusCode() != 201) {
            //TODO：ErrorHandler还未定义
        }
    }

    @future(callout=true)
    public static void deleteBook(String b) {
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('http://servicedevtest.natapp1.cc/book');
        request.setMethod('DELETE');
        request.setHeader('Content-Type', 'application/json;charset=UTF-8');
        request.setBody(b);
        HttpResponse response = http.send(request);
        if (response.getStatusCode() != 201) {
            //TODO：ErrorHandler还未定义
        }
    }

}
```

上述方式是采用Apex原生的API进行HTTP Callout；当然，这样的方式仅是作为探究，实际开发中可以利用一些现成的Http Client实现，参考[HTTPCalloutFramework](https://github.com/rahulmalhotra/HTTPCalloutFramework)，这样可以大幅度简化调用外部API时需要考虑的鉴权、token时效性等问题。

## 构建Streaming API实现对sObject的CDC监听

Salesforce中提供了基于WebSocket的Streaming API构建方式，能够实现CDC(Change Data Capture)的形式将sObject的变动推送至外部，免去了写Trigger的麻烦。当然，Trigger中推送变更的方式也很重要，通过Apex可以在推送中藏叠更多的业务逻辑。下例是Salesforce中发布一套PushTopic形式的Streaming API共外部调用的方法。

```java
//PushTopic也是一种sObject
PushTopic pushTopic = new PushTopic();
pushTopic.Name = 'BookStatementHandler';
//在SOQL中选出希望监听的范围
pushTopic.Query = 'SELECT Id, Name,Author__c,Price__c FROM Book__c';
pushTopic.ApiVersion = 51.0;
//定义增删改查时是否推送变更到流中
pushTopic.NotifyForOperationCreate = true;
pushTopic.NotifyForOperationUpdate = true;
pushTopic.NotifyForOperationUndelete = true;
pushTopic.NotifyForOperationDelete = true;
pushTopic.NotifyForFields = 'Referenced';
insert pushTopic;
```

将上述代码以匿名Apex的方式在Org中运行后，既可以通过SOQL查出这条PushTopic的概况

![image-20210402230451683](https://bricksite-1257393063.cos.ap-shanghai.myqcloud.com/image-20210402230451683.png)

在外部调用PushTopic时，绝大多数集成工具都提供了这样的选项。当然，也可以在自己的代码里调用。下列给出了利用Salesforce提供的Java测试工具EMPConnector调用PushTopic获取变更的方法。

```bash
# 将EMPConenctor的源码下载后编译成jar包，可以通过【用户名/密码】的方式，也可以通过AuthToken的方式启动进程监听PushTopic;下句提出的是利用AuthToken鉴权的办法
java -jar emp-connector-0.0.1-SNAPSHOT-phat.jar bricks9711@brave-bear-8fiuh0.com Mukendai1997tT33xSFRUa5dgka1fXXnifUh /topic/BookStatementHandler
```

## 拾遗

1. 关于Salesforce内构建SOAP API的方式，在Salesforce Developer中提到，现代微服务架构更多地采用RESTful API的方式进行消息处理。尽管SOAP API可以生成WSDL文档，方便其他系统添加Client消费API，但是SOAP API在事务处理上有着比较大的劣势。如果希望生成SOAP API，可以采用Apex提供的`@WebService`注解，这样做好的Apex Class就能直接导出WSDL文件。
2. 关于系统间集成的平台工具，除了Talend、Mulesoft等商业软件，在项目中发现Stream Set等开源ESB/ETL软件对Salesforce的支撑也非常好。除此以外，还可以利用Apache Camel等开源消息集成框架，能快速方便地构建逻辑复杂的集成微服务。下列代码给出一个示例，通过Apache Camel集成Telegram，实现自动截获Telegram robot的工单信息，并将其与Service Cloud联动

```java
public class MainApp {
    
    public static void main(String[] args) throws Exception {
    	
    	// 检查端口冲突
    	String webPort = System.getenv("PORT");
        if(webPort == null || webPort.isEmpty()) {
            webPort = "8084";
        }
        //创建一个简单的Server并将Camel的上下文Servelet注入其中
        Server server = new Server(Integer.valueOf(webPort));
        ServletContextHandler context = new ServletContextHandler();
        context.setContextPath("/camel");
        context.addServlet(new ServletHolder("CamelServlet", new CamelHttpTransportServlet()),"/*");
        server.setHandler(context);
        
        CamelContext camelContext = new DefaultCamelContext();
        
        // 创建路由
        camelContext.addRoutes(new RouteBuilder() {
			public void configure() {

				from("servlet:///cases")
					.log("Receive body = ${body}")
					.process(new Processor() {
						@Override
						public void process(Exchange exchange) throws Exception {
							ExchangeManager.saveExchange(exchange);
						}
					})
					// 允许以流式获取数据到后续步骤
					.streamCaching()					
					.unmarshal(new ListJacksonDataFormat(CaseInfo.class))
					// 分割获取到的多个Cases
					.split().body()					
					// 将消息处理，并回送到Telegram的bot中
					.process(new Processor() {
						@Override
						public void process(Exchange exchange) throws Exception {
							exchange.setProperty("OriginalExchangeId", exchange.getProperty("CamelCorrelationId", String.class));
							CaseInfo c = exchange.getIn().getBody(CaseInfo.class);
							exchange.setProperty("CaseId", c.id);
							OutgoingTextMessage messageOut = new OutgoingTextMessage();
							messageOut.setText("Case " + c.id + " is now " + c.status.toString());
							messageOut.setChatId(c.telegram_chat_id);
							exchange.getIn().setBody(messageOut);
						}
					})
					.to("telegram:bots/12345")
					.aggregate(constant(true), new GroupedExchangeAggregationStrategy()).completion(new Predicate() {
						
						@Override
						public boolean matches(Exchange exchange) {
							@SuppressWarnings("unchecked")
							List<Exchange> groupedExchange = (List<Exchange>)exchange.getProperty(Exchange.GROUPED_EXCHANGE);
							int total = groupedExchange.get(0).getProperty(Exchange.SPLIT_SIZE, Integer.class);
							int current = exchange.getProperty(Exchange.AGGREGATED_SIZE, Integer.class);
							return total == current;
						}
					})
					
					// 等待Salesforce的Response
					.process(new Processor() {
						@Override
						public void process(Exchange exchange) throws Exception {
							@SuppressWarnings("unchecked")
							List<Exchange> groupedExchange = (List<Exchange>)exchange.getIn().getBody();
							StringBuilder response = new StringBuilder();
							response.append("[");
							for(int i = 0; i < groupedExchange.size(); i++) {
								Exchange e = groupedExchange.get(i);
								response.append("{\"");
								response.append(e.getProperty("CaseId", String.class));
								response.append("\":\"");
								response.append( (e.getException() == null) ? "success" : "failure");
								response.append("\"}");
								if (i != groupedExchange.size() - 1) response.append(",");
							}
							response.append("]");
								ExchangeManager.retrieveExchange(groupedExchange.get(0).getProperty("OriginalExchangeId", String.class)).getOut().setBody(response);
						}
					})
				;		
			}
        });
        
        // start Camel and webserver
        camelContext.start();
        server.start();
        server.join();
        camelContext.stop();
    }

}
```

## 参考

1. [R.apex](https://propicsignifi.github.io/r-apex/)  Apex函数式开发框架
2. [apex-api-gateway](https://github.com/YoruNoHikage/apex-api-gateway)  开源api网关，用于管理构建Salesforce中的自定义RESTful API，并可以集成到AWS API网关服务中
3. [Workbench](https://workbench.developerforce.com/login.php)  Salesforce官方的自定义RESTful API调试工具
4. [HTTPCalloutFramework](https://github.com/rahulmalhotra/HTTPCalloutFramework)  Apex版本的`@RestTemplate`
5. [Camel-SpringBoot-Examples](https://github.com/apache/camel-spring-boot-examples)  Apache Camel集成Springboot的案例集
6. [Postman-Salesforce-Apis](https://github.com/forcedotcom/postman-salesforce-apis)  Postman测试Salesforce API的案例集