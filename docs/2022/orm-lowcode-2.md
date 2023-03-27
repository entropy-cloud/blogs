# 

# What ORM engine is required for low code platforms? (2)

[书接上回](https://zhuanlan.zhihu.com/p/543252423)。 In the last article, I made a preliminary theoretical analysis of the design of ORM, and proposed the minimal extension of SQL language: EQL object query language, and then implemented a variety of user-customizable dynamic storage structures based on EQL language. In this article, I'll first look at some of the functional trade-offs made in the NopOrm engine and how common ORM performance issues can be addressed with this functional trade-off. Then I'll show you how to implement a customizable Dialect, how to implement SQL management like MyBatis in 200 lines of code, how to do GraphQL integration and visualization integration, and more.

## Four. Less is More

There is a persistent myth that Hibernate is easy to get started with, but hard to master. But what technology is not like this? The problem with Hibernate is that it seems to offer too many choices and always forces us to keep choosing. For example, to map the associated table to a collection object, which of the many options like Set/Bag/List/Collection/Map is the best? Do delete/update/insert operations cascade to associated entities? Does removing from the collection mean removing from the database? Should association objects be loaded eager or lazy? This freedom of choice can make people with obsessive-compulsive disorder and difficulty in choosing very entangled. What if I choose the wrong one? What if changing the selection affects someone else's code? What if I regret it?

If we are always making choices, but each choice has irreversible consequences, then the rich choices bring us not happy life, but deep regret.

The NopOrm engine slashes the programmer's decision points. ** The encapsulation that the application layer can complete by itself is excluded from the engine kernel. **. For example, why do we need to map the associated table to a List, map the index field to the subscript of the list element, and add a bunch of HQL special query syntax related to List?

### 4.1 ORM automatic mapping

The first design decision for the NopOrm engine was: ** The physical model of the database design is automatically mapped to the Java entity model without requiring any additional design decisions to be made. **. This is not based on the logical model of database design, because the path from the logical model to the physical model is uncertain, and additional information must be added, while starting from the physical model, it can be completed automatically without making any choice.

> The physical model itself is already the result of the final combination of various design decisions, and will exist steadily in the future. If the ORM mapping is based on the logical model, it is equivalent to repeating the expression selection process.

Specifically, NopOrm maps each database field to a Java property on the entity (the same as MyBatis), and each foreign key association to a Lazy-loaded entity object. That is, the same field may be mapped to multiple attributes, an atomic field attribute plus one (or more) associated object attributes, which are automatically synchronized. If you update the atomic field property, the association object is automatically set to null and looked up from the session the next time the association object is accessed.

If the foreign key association explicitly marks a one-to-many collection attribute for generation, an attribute of type Set is automatically generated (without providing a choice of different collection types). According to the basic principle of ORM, object pointers in the same Session remain unique, so it can be inferred that they naturally form a Set set, and if other types of attribute mapping are used, additional assumptions must be added. Because of pointer equality, we don't need to override the equals method of the entity object either.

Only when we clearly need to use Component/Computed/Alias, we will add the corresponding configuration, and these configurations are expressed incrementally, that is, whether they exist or not will not affect all the previous field and association mappings, and will not affect the database structure definition itself. ** Because NopOrm's implementation follows the principles of reversible computation, the configuration of these deltas can be expressed in delta files without modifying the original model design files. **。

### 4.2 Goodbye, POJO

The second important design decision for NopOrm was: ** Discard the assumption of POJO **. POJO (Plain Old Java Object) was very important for Hibernate at that time, because it helped Hibernate get rid of the container environment of EJB (Enterprise Java Bean), and eventually destroyed the EJB ecosystem. But POJOs are not enough to do the job, Hibernate has to enhance the Java Entity Object with AOP (Aspect Oriented Progamming) technology to add additional functionality to it, while maintaining an EntityEntryMap in memory. Used to manage additional state data.

In the context of low-code applications, entity classes themselves are code-generated, and AOP is essentially a means of code generation (generally using the run-time bytecode generation mechanism). That being the case, ** Is it not enough to generate the final code at one time? Is it necessary to split it into two different generation stages? **

With the development of technology, the hidden cost of POJO is still increasing, leading to the continued weakening of the reasons for using it.

1. AOP's bytecode generation is slow and not easy to debug.

2. The use of POJO requires the use of reflection mechanism, which has a large loss in performance, and native Java technologies such as GraalVM need to avoid the use of reflection mechanism as far as possible.

3. POJO objects cannot maintain the relatively complex persistent state of entities, which leads to the failure of effective optimization. For example, Hibernate cannot identify whether an entity has been modified through a simple dirty flag, so it is forced to maintain a copy of the object data in memory. Every time the session flushes, it needs to traverse the object and compare the consistency between the object attribute data and the copy data one by one. This affects the performance of Hiber Nate and consumes more memory.

4. In order to implement some necessary business functions, we often need to choose entity classes to inherit from a public base class, which actually breaks the POJO assumption of objects. For example, adding dynamic attribute mapping to an entity and automatically recording the field data of the entity before and after modification all require the base class of the entity to provide a series of member variables and methods.

5. The implementation of collection properties is not performance friendly and is prone to misuse. When an object is initialized, the collection attribute is generally of HashSet type. After the object is associated with the session, the collection attribute will be automatically replaced by the PersistSet implementation class inside the ORM engine, which is equivalent to creating a new collection to replace the original POJO collection. At the same time, according to the implementation principle of ORM, in order to support lazy loading, the collection object must be bound to an entity, so it is not allowed to directly assign the collection attribute of one entity to the collection attribute of another entity. However, POJOs implement get/set methods, which can be easily misused. For example, the otherEntity. SetChildren (myEntity. GetChildren ()) is an error, and the collection returned by the myEntity. GetChildren () is bound to mgyEntity. It can no longer be a property of otherEntity.

All entity classes in NopOrm require the implementation of the IOrmEntity interface and provide a default implementation of OrmEntity

[IOrmEntity](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/IOrmEntity.java)

Each column model has a unique propId attribute, and the attribute data can be accessed through the IOrmEntity. Orm _ propValue (int propId) method instead of the reflection mechanism.

All collection properties are of type OrmEntitySet, which implements the IOrmEntitySet interface. When code is generated, the entity's collection attribute will only generate get methods, not set methods, thus eliminating the possibility of misuse.

[IOrmEntitySet](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/IOrmEntitySet.java)

When the code is automatically generated, two Java classes will be generated for each entity, such as SimsExam and _ SimsExam. The _ SimsExam class will be automatically overwritten each time, while the SimsExamination class will keep its original content if it already exists, so the manually adjusted code can be written in the SimsExamination class. See

[_SimsExam](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/java/io/nop/app/_gen/_SimsExam.java)

[SimsExam](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/java/io/nop/app/SimsExam.java)

### 4.3 Lazy and Cascade all

In ** All associated entities and associated collections are lazy-loaded and do not support class inheritance ** NopOrm. This design decision greatly simplifies the internal implementation of the ORM engine and lays the foundation for uniform bulk loading optimization.

The lazy setting can be added to the column definition of the entity model to specify the delay loading of the data column. By default, only all eager attributes are loaded when the entity is loaded for the first time, and lazy attributes are loaded only when they are used. At the same time, the engine provides a bulk preloading mechanism that explicitly specifies which data columns are to be loaded at once, thereby avoiding multiple database accesses.

Support for eager fetch syntax is not provided in the EQL syntax because using eager fetch results in SQL statements that are not as consistent as you might expect. For example, if the main table is loaded and the sub-table record information is loaded through the join statement at the same time, the number of entries in the returned result set will increase, and a large amount of redundant data will be returned, which is not friendly to performance. In NopOrm, the batch loading queue provided by Batch Load Queue is used to achieve performance optimization.

Hibernate cascade is triggered by action. For example, when calling the session. Save, cascade will execute the save action of the associated property. The original purpose of its design may be performance optimization, for example, some attributes do not need cascade and will be automatically skipped. However, action-based triggering can lead to unexpected results. For example, if the entity is modified again after save, two SQL statements may be generated: one insert and one update. Originally, only one insert statement needs to be generated.

Hibernate's FlushMode setting is also prone to confusing results. FlushMode is set to auto by default, and hibernate decides whether to actively refresh the database. As a result, some subtle and equivalent logic adjustments at the Java code level will mislead hibernate to mistakenly judge that the database needs to be refreshed, thus issuing a large number of SQL calls, resulting in serious performance problems.

NopOrm has been designed ** Completely lazy ** so that it eliminates the FlushMode concept, only flushes the database when the session. Flush () is explicitly called, and combines the OrmTemplate pattern, The template method is responsible for calling the session. Flush () before the transaction is committed, thus improving the predictability of the execution results of the ORM engine at the conceptual level.

NopOrm adopts a state-driven cascade design, that is, no cascade is executed in each operation, and only one cascade operation is executed for all entities in the session. Flush. At the same time, the dirty flag is used to implement pruning optimization. If all entities of a certain type are not modified, the dirty flag corresponding to this type is false, and all instances of this type will automatically skip the flush operation. If all entities in the entire session are not modified, the global dirty flag is false, and the entire session. Flush () operation will be skipped.

Another side effect of Hibernate's action-based cascade is that the execution order of specific SQL statements is difficult to control precisely. In NopOrm, the actions generated by flush will be cached in the actionQueue queue, and then uniformly sorted according to the topological dependency order of the database tables before execution, so as to ensure that the database is always modified according to the determined table order, which can avoid the occurrence of database deadlock to a certain extent.

> Deadlock is usually caused by thread A modifying table A first and then modifying table B, while thread B modifying table B first and then modifying table A. Executing the database update statements in the specified order acts as a sort of lock.

### Five. More is Better

NopOrm abandons a large number of features in Hibernate, but at the same time it provides many features that Hiber Nate lacks, which are very common in general business development and often require a lot of development. The difference is that these features are all optional, and whether they are enabled or not has no effect on the other functionality that is already implemented.

### 5.1 Good parts of Hibernate

NopOrm inherits some very good designs from Hibernate and Spring frameworks:

1. By default, the cache size ** L2 Cache and Query Cache ** is limited to avoid memory overflow.

2. It is difficult to completely avoid compound primary keys ** Composite primary key support ** in business system development. NopOrm has a built-in composite primary key class OrmCompositePk, and automatically generates a Builder auxiliary function to automatically realize the conversion between String and OrmCompositePk, thereby simplifying the use of the composite primary key.

3. Primary keys can be automatically generated in Java code ** Primary Key Generator ** as long as the seq tag is marked on the column model. If the primary key has been set on the entity, the value set by the user shall prevail. Different from Hibernate, NopOrm uses a globally uniform SequenceGenerator. Generate (entityName) to generate primary keys, which is convenient for dynamically adjusting the primary key generation strategy at runtime. NopOrm abandons the support of database self-increasing primary key, because this feature is not supported by many databases and has problems in distributed environments.

4. ** JDBC Batch **: Automatically merge database update statements to reduce database interactions. In the debugging mode, the specific parameter values of the SQL statement before merging will be printed, which is convenient for diagnosing problems when errors occur.

5. ** Optimistic lock **: Update the database with update XXX set version = version + 1 where version =: curVersion to avoid concurrent modification conflicts

6. ** Template method pattern **: Improved the coordination of JdbcTemplate/Transaction Manager/OrmTemplate, reduced redundant encapsulation conversion, and added asynchronous processing support to use OrmSession in an asynchronous environment.

7. ** Interceptor **: You can intercept operations on a single entity inside the ORM engine through methods such as preSave/preUpdate/preDelete of OrmInterceptor, so as to implement functions similar to triggers in the database.

8. Unify the paging mechanism of different databases ** Paging ** with Dialect. It also adds support for a MySQL-like offset/limit syntax to EQL syntax.

9. SQL syntax compatibility translation across databases ** SQL compatibility ** with Dialect, including translation of syntax formats and SQL functions.

### 5.2 ORM with better understanding of requirements

Some common business requirements can be easily implemented with the ORM engine, so NopOrm provides support for them out of the box without the need to install additional plug-ins.

1. ** Multi-tenancy **: Add a tenantId filter condition for tenant-enabled tables and prohibit cross-tenant access to data

2. ** Sub-warehouse and sub-table **: Dynamic selection of sub-database and sub-table through IShardSelector

3. ** Logical deletion **: Convert the delete operation into the modification operation of setting delFlag = 1, and automatically add the filter condition of adding delFlag = 0 in the general query statement.

4. ** Timestamp **: Automatically record operation history information such as modifier and modification time

5. ** Modify the log **: The field value information of the entity before and after modification can be obtained by intercepting the entity modification operation through OrmInterceptor, and recorded in a separate modification log table.

6. ** History table support **: Add revType/beginVer/endVer fields to the record table, allocate a starting version number and an ending version number to each record, convert the modified record into a newly added record, and set the ending version number of the previous record as the starting version number of the latest record. Filter conditions are automatically added to the general query statement to find only the latest version of the record.

7. ** Field encryption **: Adding an enc tag to the column model indicates that the field needs to be encrypted for storage. At this time, the system will use the customized IDataParameterBinder to read the database field value, so that it can be saved in the database in encrypted form and stored in the Java attribute in decrypted form. The EQL parser can know the parameter type through syntax analysis, so that the encode binder is used transparently to encrypt and decrypt the parameters of the SQL statement.

8. ** Sensitive Data Mask **: a mask tag can be added to sensitive information fields such as the user's card number and ID number, so that the field value can be automatically masked when the log is printed within the system to avoid disclosure to the log file.

9. ** Component logic reuse ** A set of related fields may form a reusable component, and these logics can be reused through the OrmComponent mechanism. For example, the precision of the Decimal type in the database must be specified in advance, but the customer requires that it must be displayed and calculated according to the precision specified during input, which requires us to add a VALUE _ SCALE field in the record table to retain the precision information. But when we fetch the value from the database, we want to directly get a BigDecimal whose scale has been set to the specified value. NopOrm provides a FloatingScaleDecimal component to do this. Fields with complex association logic, such as attachments and attachment lists, can be encapsulated in a similar manner.
   
   [FloatingScaleDecimal](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/support/FloatingScaleDecimal.java)

In combination with the peripheral framework, the Nop platform also has more commonly used solutions built in. For example

1. ** General Query **: There is no need to write code at the front and back ends. As long as the form submits a query request in a certain format, the background can perform format verification and permission verification according to the meta metadata configuration. After passing the verification, the query is automatically executed and the result is returned in the GraphQL result format.

2. ** Modification confirmation and approval **: Combined with CRUD service and API call service, the user does not directly modify the database or send an API call after submitting a request, but automatically generates an approval application. The approver can see the contents before and after modification on the approval interface, and then execute the subsequent actions after approval. Through this scheme, any form interface can be transformed into an application submission page and an approval confirmation page.

3. ** Copy New ** A complex business object can be created by copying an existing object. The fields that need to be copied can be specified in a GraphQL-like query syntax.

4. ** Dictionary Table Translation **: When displaying the front end, you need to translate fields such as statusId into the corresponding display text through the dictionary table, and you need to select the corresponding multi-language version according to the locale setting of the current login user. The Nop platform will automatically discover all the fields configured with the dict attribute in the metaprogramming phase, and automatically add an associated display text field for their corresponding GraphQL descriptions, for example, add a statusId _ text field according to the statusId. When the foreground GraphQL requests the statusId _ text field, the result of the dictionary translation can be obtained, and the original value of the field can still be obtained through the statusId field.

5. ** Batch import and export **: The data can be imported by uploading a CSV file or an Excel file. The logic executed during the import is completely consistent with the manual submission through the interface, and the data permission will be automatically verified. Files can be exported in CSV or Excel format.

6. ** Distributed transaction **: Automatically integrates with the TCC distributed transaction engine.

NopOrm follows the principle of reversible computation, so its underlying models are customizable. Users can add custom attributes to the model at any time according to their own needs, and then use this information through meta-programming, code generators and other mechanisms. A large number of function implementations described above are actually implemented using similar mechanisms, many of which are not functions completed by the engine kernel, but are introduced by the customization mechanism.

### 5.3 Embrace the new world of asynchrony

Traditionally, JDBC access interfaces are all synchronous, so the encapsulation style of JdbcTemplate and HibernateTemplate is also synchronous. However, with the spread of the idea of asynchronous high concurrency programming, the responsive programming style has gradually begun to enter the mainstream framework. Spring is currently proposed [R2DBC标准](https://r2dbc.io/), but [vertx框架](https://vertx.io/) also built-in support for MySQL, PostgreSQL and other mainstream databases [异步连接器](https://vertx.io/docs/vertx-pg-client/java/). On the other hand, if the ORM engine is used as a data fusion access engine, its underlying storage may be NoSQL data sources such as Redis, ElasticSearch and MongoDB that support asynchronous access, and the ORM needs to cooperate with the GraphQL asynchronous execution engine. With these conditions in mind, NopOrm's OrmTemplate wrapper also adds an asynchronous invocation pattern.


```java
 public interface IOrmTemplate extends ISqlExecutor {
    <T> CompletionStage<T> runInSessionAsync(
          Function<IOrmSession, CompletionStage<T>> callback);
 }
```

OrmSession is thread-unsafe by design, allowing access by only one thread at a time. In order to implement multi-thread access to the unsafe data structure of the same thread, a basic design scheme is to adopt an Actor-like task queue pattern.


```java
class Context{
    ContextTaskQueue taskQueue;

    public void runOnContext(Runnable task) {
        if (!taskQueue.enqueue(task)) {
            taskQueue.flush();
        }
    }
}
```

Context is a context object that is passed across threads and has a corresponding task queue. Only one thread will execute a task registered in the task queue at any one time. The runOnContext function registers the task in the task queue, and if it finds that no other thread is executing the task queue, the current thread is responsible for execution.

> For recursive calls, taskQueue actually serves a similar [ trampoline function ](https://zhuanlan.zhihu.com/p/142241289) purpose.

If the concept of asynchronous Context is introduced, we can also improve the timeout support for remote service calls. After the remote service call times out, the client will throw an exception or initiate a retry, but the server does not know that it has timed out and continues to execute. Generally, the service function will access the database for many times. If the traffic caused by retry is superimposed at this time, the actual pressure on the database will be much greater than that in the scenario without timeout. An improvement strategy is to add a timeout attribute on the Context.


```java
class Context{
    long callExpireTime;
}
```

When cross-system calls are made, a timeout timeout interval can be passed through the RPC message header. After the server receives the timeout, the current time is added to get the callExpireTime (callExpireTime = currentTime + timeout). Then in the JdbcTemplate, before each database request is issued, it will check whether the callExpireTime has been reached, so as to find out that the server has timed out in time. If the API of a third-party system is to be called on the server side, the timeout = callExpireTime-currentTime is recalculated to obtain the remaining timeout interval, which is passed to the third-party system.

### 5.4 Differential Quantization Customization for Dialect

NopOrm uses the Dialect model to encapsulate the differences between different databases.

[default dialect](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-dao/src/main/resources/_vfs/nop/dao/dialect/default.dialect.xml)

[mysql dialect](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-dao/src/main/resources/_vfs/nop/dao/dialect/mysql.dialect.xml)

[postgresql dialect](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-dao/src/main/resources/_vfs/nop/dao/dialect/postgresql.dialect.xml)

Referring to the example above, both the MySQL. Dialect. XML and PostgreSQL. Dialect. XML are inherited from the default. Dialect. XML. Compared with Hibernate's construction of Dialect objects by programming, the use of dialect model file has higher information density and more intuitive expression. More importantly, the configuration relative to the default. Dialect. XML ** Addition, modification and reduction ** can be clearly identified in the PostgreSQL. Dialect. XML.

Because the bottom layer of the entire Nop platform is built based on the principle of reversible computation, the parsing and verification of the dialect model file can be completed by the general DslModelParser, and at the same time, Delta customization is automatically supported. That is ** Without modifying the default. Dialect. XML file and without modifying all references to the default. Dialect. XML films ** (for example, there is no need to modify the X: extends attribute in the PostgreSQL. Dialect. XML), we can add a default. Dialect. XML file in the/_ delta directory to customize the built-in model file of the system.


```xml
<!-- /_delta/myapp/nop/dao/dialect/default.dialect.xml -->
<dialect x:extends="raw:/nop/dao/dialect/default.dialect.xml">
  这里只需要描述差量变化的部分
</dialect>
```

Delta customization is similar to overlay FS differential file system ** Allow overlay of multiple Delta layers ** in Docker technology. Unlike Docker, Delta customization occurs not only at the file level, but also extends to the differential structure operations within the file. ** With the help of the xdef metamodel definition, all model files in the Nop platform automatically support Delta quantization customization. **。

### 5.5 Visual integration

The hibernate HBM definition file and JPA annotations are designed for database structure mapping and are not suitable for visual model design. Adding a visual designer to Hibernate is a relatively complex matter.

NopOrm uses orm. XML model files to define solid models. First of all, it is a complete structure definition model, which can generate database building scripts according to the information in the model, and automatically compare the differences with the current database structure and automatically migrate data.

[app.orm.xml](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/resources/_vfs/nop/test/orm/app.orm.xml)

Adding a visual designer to NopOrm is as simple as adding a metaprogramming tag call


```xml
<orm ... >
    <x:gen-extends>
        <pdman:GenOrm src="test.pdma.json" xpl:lib="/nop/orm/xlib/pdman.xlib"
                      versionCol="REVISION"
                      createrCol="CREATED_BY" createTimeCol="CREATED_TIME"
                      updaterCol="UPDATED_BY" updateTimeCol="UPDATED_TIME"
                      tenantCol="TENANT_ID"
        />
          ...
    </x:gen-extends>
</orm>  
```

[Pdman](http://www.pdman.cn/) Is an open source database modeling tool that saves model information in the JSON file format. `<pdman:GenOrm>` Is an XPL template language tag that runs during the compile-time metaprogramming phase and automatically generates orm model files based on pdman's JSON model. This generation is immediate, that is, as long as the test. Pdma. JSON file is modified, the parsing cache of OrmModel will be invalid, and the new model object will be re-parsed when it is accessed again.

According to the theory of reversible computation, the so-called visual design interface is just a graphical Representation of the domain model, and the model file text can be regarded as the text Representation of the domain model. Reversible computing theory points out that a model can have multiple representations, and visual editing only means that there is a reversible transformation between the graphical representation and the text representation. Reasoning along this direction, we can draw an inference that the visual presentation form of a model is not unique, and there can be a variety of different forms of visual designers to design the same model object.

Corresponding to the orm model, in addition to pdman, we can also choose to use the power designer design tool to design, the same

A similar `<pdm:GenOrm>` tag allows you to convert PDM model files into the model format required by orm.

In the Nop platform, we also support the definition of entity data models through the Excel file format.

[test.orm.xlsx](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/resources/_vfs/nop/test/orm/test.orm.xlsx)

Similarly, we just need to introduce a tag call `<orm-gen:GenFromExcel>`, and then we can happily design ORM models in Excel.


```xml
<orm ...>
  <x:gen-extends>
     <orm-gen:GenFromExcel path="test.orm.xlsx" />
  </x:gen-extends>
</orm>
```

It is worth mentioning that the Excel model file parsing in Nop platform is also designed based on reversible computing theory. The theory of reversible computation regards the parsing of Excel model files as a functor mapping from the category of Excel to the category of DSL AST (Abstract Syntax Tree) (also an equivalent representation transformation), so a general Excel model parser can be implemented. The Excel model can be parsed only by inputting the structural information defined by the orm meta-model file without any special coding. This mechanism is completely universal, that is, for any model file defined in the Nop platform, we can get its corresponding Excel visual editing model for free. At the same time, the format of these Excel files is relatively free, and we can adjust the position, style and order of cells at will. As long as they can be recognized as Tree structures according to some deterministic rule.

For further information on model transformations, refer to the following article

Low Code Platform Design from Tensor Product (https://zhuanlan.zhihu.com/p/531474176)

The model information in the Nop platform can also be exported in the form of a general Word template. For specific technical solutions, please refer to

How to implement a Word template like poi-tl in 800 lines of code (https://zhuanlan.zhihu.com/p/537439335)

All business functions supported by the Nop platform are implemented in a model-driven manner, so it is possible to derive a large amount of useful information by analyzing model information. For example, we have exported database model documents, data dictionary documents, API interface documents, unit test documents and so on according to the internal model.

### Six. The long-standing N + 1 problem

Ever since the birth of Hibernate, the so-called N + 1 problem has been a dark cloud hanging over the ORM engine. Suppose we have such a model.


```java
class Customer{
    Set<Order> orders;
}

class Order{
   Set<OrderDetail> details;   
}
```

If we want to process the order details of a customer, we need to traverse the orders set.


```
Customer customer = ... // 假设已经获取到customer
Set<Order> orders = customer.getOrders();
for(Order order: orders){
    process(order.getDetails());
}
```

Loading the orders collection from the customer requires issuing an SQL statement, traversing the orders collection, and issuing another SQL statement for each order to obtain its details collection, resulting in N + 1 query statements throughout the process.

The N + 1 problem is notorious because the amount of data in the development phase is small, performance problems are often overlooked, and we have no ** Remedy by local adjustment ** means to find problems when we go online. All changes often require rewriting the code, or even completely changing the design of the program.

This problem plagued Hibernate until many years later when the JPA (Java Persistence API) standard introduced the concept of an Entity Graph.


```java
@NamedEntityGraph(
   name = "customer-with-orders-and-details",
   attributeNodes = {
       @NamedAttributeNode(value = "orders", subgraph = "order-details"),
   },
   subgraphs = {@NamedSubgraph(
       name = "order-details",
       attributeNodes = {
           @NamedAttributeNode("deails")
       }
   )}
)
@Entity       
class Customer{
    ...
}
```

Add the NamedEntity Graph annotation on the entity class, and declare that the orders collection and details collection should be loaded at one time when loading the object. Then specify which Entity Graph configuration to use when calling the find method.


```java
EntityGraph entityGraph = entityManager.getEntityGraph("customer-with-orders-and-detail");
Map<String,Object> hints = new HashMap<>();
hints.put("javax.persistence.fetchgraph", entityGraph);
Customer customer = entityManager.find(Customer.class, customerId, hints);
```

In addition to being declared using annotations, EntityGraphs can also be constructed in code.


```java
EntityGraph graph = entityManager.createEntityGraph(Customer.class);
Subgraph detailGraph = graph.addSubgraph("order-details");
detailGraph.addAttributeNodes("details");
```

A SQL statement similar to the following is actually generated


```sql
select customer0.*,
       order1.*,
       detail2.*
from 
     customer customer0 
       left join order order1 on ...
       left join order_detail detail2 on ...
 where customer0.id = ?   
```

Hibernate uses a single SQL statement to get all the data out, at the cost of requiring multiple tables to be associated and returning a lot of redundant data.

Are there other solutions to this problem? From the structure of the data model itself, the nested structure of Customer- > orders- > details is very intuitive and natural, and there is no problem. However ** The problem is that we can only get the data in the way that the object structure is defined, and we can only iterate through the object structure one by one **, it results in a large number of data query statements. ** If we can bypass the object structure, get the object data directly in some way, and organize them in memory according to the required object structure, won't this problem be solved? **


```java
Customer customer = ...
// 插入一条神秘的数据获取指令
fetchAndAssembleDataInAMagicalWay(customer);
// 数据已经在内存中存在，可以安全的遍历并使用，不再产生数据加载动作
Set<Order> orders = customer.getOrders();
for(Order order: orders){
    process(order.getDetails());
}
```

NopOrm provides an interface to load attributes in bulk via OrmTemplate.


```java
ormTemplate.batchLoadProps(Arrays.asList(customer), Arrays.asList("orders.details"));
// 数据已经在内存中存在，可以安全的遍历并使用，不再产生数据加载动作
Set<Order> orders = customer.getOrders();
```

OrmTemplate does this internally by loading the queue with IBatchLoadQueue


```java
IBatchLoadQueue queue = session.getBatchLoadQueue();
queue.enqueue(entity);
queue.enqueueManyProps(collection,propNames);
queue.enqueueSelection(collection,fieldSelection);
queue.flush();
```

The internal implementation principle of BatchLoadQueue is actually similar to the DataLoader mechanism of GraphQL, which is to collect the entity or entity collection objects to be loaded first, then use one `select xxx from ref__entity where ownerId in:idList` to obtain data in batches, and then split them into different objects and collections according to ownerId. Because the BatchLoadQueue has all the information of the entity model and has a unified loader, its internal implementation is more optimized than the DataLoader. At the same time, in terms of external interfaces, the amount of information that needs to be expressed is less. For example, a orders. Details means that you need to load orders first, then the details collection, and get all the eager properties of the OrderDetail object. If you are using a GraphQL description, you need to explicitly specify which properties on the OrderDetail object to get, and the description is a bit more complex.

> BatchLoadQueue is not GraphQL-inspired. GraphQL was open sourced in 2015, and we were already using BatchLoadQueue before that.

If the number of entity objects that need to be loaded is very large and the hierarchy is very deep, batch fetching by ID also has some impact on performance. To this end, NopOrm maintains a super backdoor for experts only,


```java
 session.assembleAllCollectionInMemory(collectionName);
 或者
 session.assembleCollectionInMemory(entitySet);
```

** AssembleAllCollectionInMemory assumes that all the entity objects involved have been loaded into memory **, so it no longer accesses the database, but ** The collection elements are determined directly by filtering the data in memory **. As for how to load all relevant entities into memory, there are many ways. For example


```java
orm().findAll(new SQL("select o from Order o"));
orm().findAll(new SQL("select o from OrderDetail o"));
session.assembleAllCollectionInMemory("test.Customer@orders");
session.assembleAllCollectionInMemory("test.Order@details");
```

> This approach is dangerous because if you don't load all the associated entities into memory before calling the assemble function, the assembled collection object is wrong.

If we recall the SQL statement generated by the EntityGraph earlier, it actually corresponds to the following EQL query


```sql
select c, o, d
from Customer c left join c.orders o left join o.details
where c.id = ?
```

According to the basic principle of ORM, although the query statement returns many duplicate Customer and Order objects, because their primary keys are the same, only one instance will be retained when they are constructed as objects in memory. Even if a Customer or Order object has been loaded before, its data will be based on the results loaded before, and the data obtained from this query will be automatically ignored.

>  That is, ORM provides a database-like [ Repeatable Read Transaction Isolation Level ](https://zhuanlan.zhihu.com/p/150107974) effect. When repeated reads only read a lonely, ORM engine will only keep the results of the first read. For the same reason, in the case of Load X, Update X, Load X, the data read by the second load is automatically discarded, so that what we observe is always the result of the first load and the subsequent modifications we make to the entity, which is equivalent to implementation [ Read your writes such causal consistency ](https://zhuanlan.zhihu.com/p/59119088).

Based on the above knowledge, the execution process of Entity Graph is equivalent to the following call


```sql
orm().findAll(new SQL("select c,o,d from Customer c left join ..."));
session.assembleSelectionInMemory(c, FieldSelectionBean.fromProp("orders.details"));
// assembleSelection的执行过程等价于如下调用
session.assembleCollectionInMemory(c.getOrders());
for(Order o: c.getOrders()){
    session.assembleCollectionInMemory(o.getDetails());
}
```

## Seven. QueryBuilder is important, but has nothing to do with ORM

Some people think [ The role of the ORM is largely in the QueryBuilder ](https://www.zhihu.com/question/23244681/answer/2426095608), and I think this is a misunderstanding. ** Query Builder is useful only because Query objects need to be modeled. **。 In the Nop platform, we provide the QueryBean model object, which supports the following functions

1. QueryBean corresponds to QueryForm and QueryBuilder controls in the foreground, and complex query conditions can be directly constructed by these control

2. The filter condition corresponding to the background data permission filtering can be directly inserted into the QueryBean. Compared with SQL splicing, the structure is clear and SQL injection attacks will not occur. queryBean.appendFilter(filter)

3. QueryBean supports user-defined query operators and query fields, which can be converted into built-in query operators through queryBean. Transform Filter (FN) in the background. For example, we can define a virtual field myField, then query the state data in memory and the data of other associated tables, and convert it into a subquery condition ** Under the single-table query framework, the effect of multi-table joint query can actually be realized. **.

4. The DaoQuery Helper. QueryToSelectObjectSql (query) translates query conditions into SQL statement

5. The QueryBeanHelper. ToPredicate (filter) can convert the filter condition to the Predicate interface, so as to filter directly in Java.

6. Through the and, EQ and other operators defined in FilterBeans, combined with the attribute name constant automatically generated during code generation, we can achieve the following compile-time safe construction.
   
   filter = and(eq(PROP_NAME_myFld,"a"), gt(PROP_NAME_otherFld,3))

Query Builder is essentially ORM-agnostic, because we can still use the Query model completely outside of relational databases and SQL statements. For example, in a business rule configuration


```xml
<decisionTree>
    <children>
      <rule>
        <filter>
          <eq name="message.type" vaule="@:1" />
          <match name="message.desc" value="a.*" />
        </filter>
        <output name="channel" value="A" />
      </rule>
      <rule>
        ...
      </rule>
    </children>
</decisionTree>
```

The foreground Query Builder can be directly reused to realize the visual configuration of the background decision rules.

Constructing SQL statements in Java code through something called QueryDsl is not inherently advantageous. If a model-driven approach is adopted, it is better to directly use the QueryBean passed by the foreground, and to supplement a few query conditions, static composition functions such as and/or/EQ defined in FilterBeans can be used. If it is a very complex SQL construction, it is undoubtedly a better choice to directly adopt a scheme similar to MyBatis and unify the management in an independent external file. In sql-lib, we can achieve intuitiveness, flexibility, and extensibility that QueryDsl cannot (more on that later).

[QueryBean](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-api-core/src/main/java/io/nop/api/core/beans/query/QueryBean.java)

## Eight. Can OLAP analysis use ORM?

There has been a saying that ORM is only suitable for OLTP applications, and it is powerless for the complex query statements required for OLAP data analysis. But some people want to go up in the face of difficulties, that is, to use ORM, but also to use faster, higher and stronger!

> To be honest, is it easy to write summary analysis statements with SQL? Too many associations and subqueries are simply used to organize data along a certain dimension. Would it be easier to split it into multiple queries and then assemble them together in the program?

[润乾报表](http://www.raqsoft.com.cn/about#aboutme) It is a very unique company. Its founder, Jiang Buxing, is a legendary figure in Chinese history (the first Chinese gold medal winner of the International Mathematical Olympiad, from Shihezi, Xinjiang, see [顾险峰教授的回忆](https://blog.sciencenet.cn/blog-2472277-1160241.html)). He invented the theory related to the Chinese-style report model and led the technological trend of a whole generation of report software. Although due to various reasons, the final development of Rungan Company is not satisfactory, it has expressed many unique views on design theory.

Run dry open source one [前端BI系统](http://www.raqsoft.com.cn/r/os-bi), although its face value is a little low, but in the technical level has put forward a unique DQL (Dimentinal Query Language) language. The specific introduction can refer to the article of Cadre College.

[Say Goodbye to Broad Watch, Achieve a New Generation of BI with DQL-Dry College] (http://c.raqsoft.com.cn/article/1653901344139?p=1&m=0)

Rungan's point of view is that it is difficult for end users to understand complex SQL JOINs. In order to facilitate multi-dimensional analysis, only large-width tables can be used, which brings a series of difficulties to data preparation. DQL, on the other hand, simplifies the mental model of JOIN operations for end users and has a performance advantage over SQL.

Take how to find ** American employees of Chinese managers ** as an example


```sql
-- SQL
SELECT A.*
FROM  员工表 A
JOIN 部门表  ON A.部门 = 部门表.编号
JOIN  员工表 C ON  部门表.经理 = C.编号
WHERE A.国籍 = '美国'  AND C.国籍 = '中国'

-- DQL
SELECT *
FROM 员工表
WHERE 国籍='美国' AND 部门.经理.国籍='中国'
```

The key point here is called foreign key attribution, which means that the fields of the table pointed to by the foreign key can be directly referenced by subattributes, and multiple and recursive references are also allowed.

Another similar example is to query the shipping city name, province name and region name of the order according to the order table (orders) and area table (area).


```sql
-- DQL
SELECT
    send_city.name city,
    send_city.pid.name province,
    send_city.pid.pid.name region
FROM 
    orders
```

The second key idea of DQL is ** Same Dimension Table Equal Assimilation ** that tables that are one-to-one related can consider their fields to be shared without explicitly writing related query conditions. For example, the employee table and the manager table are one-to-one, and we need to query ** Income of all employees **


```sql
-- SQL
SELECT 员工表.姓名, 员工表.工资 + 经理表.津贴
FROM 员工表
LEFT JOIN 经理表 ON 员工表.编码 = 经理表.编号

-- DQL
SELECT 姓名,工资+津贴
FROM 员工表
```

The third key idea of DQL is ** Subtable collectivization ** that, for example, an order detail table can be thought of as a collection field of an order table. If you want to calculate the total amount for each order,


```sql
-- SQL
SELECT T1.订单编号,T1.客户,SUM(T2.价格)  
FROM 订单表T1  
JOIN 订单明细表T2 ON T1.订单编号=T2.订单编号  
GROUP BY T1.订单编号,T1.客户

-- DQL
SELECT 订单编号,客户,订单明细表.SUM(价格)  
FROM 订单表
```

"If there are multiple sub-tables, SQL needs to do GROUP first, and then JOIN with the main table together, which will be written in the form of sub-query, but DQL is still very simple, directly add fields after SELECT.".

The fourth key idea of DQL is: ** Natural alignment of data by dimension **. We don't need to specify the association conditions. The reason why the final data can be displayed in the same table is not because there is any prior association between them, but because they share the leftmost dimensional coordinates. For example, we hope ** Statistics of contract amount, recovery amount and inventory amount by date **. We need to take data from the three tables separately, align them according to the date, and summarize them into the result data set.


```sql
-- SQL
SELECT T1.日期,T1.金额,T2.金额, T3.金额
FROM (SELECT  日期, SUM(金额) 金额  FROM  合同表  GROUP  BY  日期）T1
LEFT JOIN (SELECT  日期, SUM(金额) 金额  FROM  回款表  GROUP  BY  日期）T2
ON T1.日期 = T2.日期
LEFT JOIN (SELECT  日期, SUM(金额) 金额  FROM  库存表  GROUP  BY  日期 ) T3
ON T2.日期 = T3.日期

-- DQL
SELECT 合同表.SUM(金额),回款表.SUM(金额),库存表.SUM(金额) ON 日期
FROM 合同表 BY 日期
LEFT JOIN 回款表 BY 日期
LEFT JOIN 库存表 BY 日期
```

In DQL, dimension alignment can be combined with foreign key attribution, for example


```sql
-- DQL
SELECT 销售员.count(1),合同表.sum(金额) ON 地区
FROM 销售员 BY 地区
JOIN 合同表 BY 客户表.地区
SELECT 销售员.count(1),合同表.sum(金额) ON 地区
FROM 销售员 BY 地区
JOIN 合同表 BY 客户表.地区
```

If we look at the design of DQL from the perspective of NopOrm, it is clear that DQL is essentially an ORM design.

1. DQL requires the designer to define primary and foreign key associations and assign explicit names on the interface to each field, exactly as it does for ORM model design.

2. The foreign key attribution, the same dimension assimilation and the sub-table aggregation of DQL are essentially the object attribute association syntax in EQL syntax, but it directly uses the association field of the database as the association object name. This approach is relatively simple, but the disadvantage is that it is not easy to handle the case of composite primary key associations.

3. Dimension alignment of DQL is an interesting idea. Its specific implementation should be divided into multiple SQL statements to load data, and then implement association in memory through Hash Join, which is very fast. Especially in the case of paging query, we can only perform paging query on the main table, and then other sub-tables can only take the records involved in the data of this page through the in condition. In the case of a large table, it is possible to speed up a lot.

It is relatively simple to implement the functions of DQL based on EQL language. After reading Rungan's article, I probably spent the weekend implementing an MdxQueryExecutor that performs dimensionally aligned queries. Because EQL has built-in support for object attribute association, it only needs to implement splitting, fragment execution, data collocation and fusion of QueryBean objects.

## Nine. SQL template management, you deserve it

When we need to construct more complex SQL or EQL statements, it is undoubtedly of great value to manage them through an external model file. MyBatis provides such a mechanism for modeling SQL statements, but there are still many people who prefer to splice SQL dynamically in Java code through schemes like QueryDsl. This is actually an illustration ** The function implementation of MyBatis is relatively weak, and it can not give full play to the advantages of modeling. **.

In NopOrm, we manage all complex SQL/EQL/DQL statements uniformly through the sql-lib model. With the existing infrastructure of the Nop platform, it only takes about 200 lines of code to implement this SQL statement management mechanism similar to MyBatis. For the specific implementation code, refer to

[SqlLibManager](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/sql_lib/SqlLibManager.java)

[SqlItemModel](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/sql_lib/SqlItemModel.java)

[SqlLibInvoker](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/sql_lib/proxy/SqlLibInvoker.java)

For the test sql-lib file, see

[test.sql-lib.xml](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/resources/_vfs/nop/test/sql/test.sql-lib.xml)

Sql-lib provides the following features

### 9.1 Unified Management SQL/EQL/DQL

There are three kinds of nodes in the sql-lib file. The SQL/eql/query corresponds to the SQL statement, the EQL statement, and the DQL query model described in the previous section. They can be managed in a unified way.


```xml
<sql-lib>
  <sqls>
     <sql name="xxx" > ... </sql>
     <eql name="yyy" > ... </eql>
     <query name="zz" > ... </query>
  </sqls>
</sql-lib>
```

The first benefit of modeling is the Delta customization mechanism built into the Nop platform. Assuming that we have developed a Base product and need to optimize SQL according to the customer's data when deploying it at the customer's place, we ** There is no need to modify the code of any Base product ** only need to add a sql-lib differential model file to customize any SQL statement. For example


```xml
<sql-lib x:extends="raw:/original.sql-lib.xml">
   <sqls>
      <!-- 同名的sql语句会覆盖基类文件中的定义 -->
      <eql name="yyy"> ...</eql>
   </sqls>
</sql-lib>
```

Another common use of Delta customization is in conjunction with metaprogramming mechanisms. Assuming that our system is a system with a very complete domain model and there are a large number of similar SQL statements, we can automatically generate these SQL statements during compilation through the metaprogramming mechanism, and then improve them through Delta customization. For example


```xml
<sql-lib>
   <x:gen-extends>
       <app:GenDefaultSqls ... />
   </x:gen-extends>

  <sqls>
     <!-- 在这里可以对自动生成SQL进行定制 -->
     <eql name=”yyy“>...</eql> 
  </sqls>
</sql-lib>
```

### 9.XPL Template Component Abstraction Capabilities

MyBatis only provides a few fixed tags such as Foreach/if/include, which can be said to be powerless when it comes to writing highly complex dynamic SQL statements. Many people find it troublesome to splice SQL in XML, which boils down to the fact that MyBatis provides an imperfect solution that ** Lack of a mechanism for secondary abstraction **. In Java programs, we can always reuse a certain section of SQL splicing logic through function encapsulation. In contrast, MyBatis only has three built-in axes, and basically does not provide any auxiliary reuse capability.

NopOrm directly uses the XPL template language in XLang as the underlying generation engine, so it automatically inherits the tag abstraction capabilities of the XPL template language.

> XLang is a programming language designed for reversible computation theory, which includes XDefinition/XScript/Xpl/XTransform and so on. Its core design idea is the generation, transformation and difference combination of the abstract syntax tree AST. It can be considered as a programming language designed for Tree grammar.


```xml
<sql name="xxx">
  <source>
   select <my:MyFields />
       <my:WhenAdmin>
         ,<my:AdmninFields />
       </my:WhenAdmin>
   from MyEntity o
   where <my:AuthFilter/>
  </source>
</sql>
```

The Xpl template language not only has `<c:for>` built-in syntax elements required by Turing-complete languages such as?, `<c:if>` but also allows the introduction of new tag abstractions through a custom tag mechanism (analogous to the vue component encapsulation on the front end).

Some template languages require that all functions that can be used in the template be registered in advance, while the Xpl template language can call Java directly.


```xml
<sql>
  <source>
    <c:script>
       import test.MyService;

       let service = new MyService();
       let bean = inject("MyBean"); // 直接获取IoC容器中注册的bean
    </c:script>
  </source>
</sql>
```

### 9.3 Metaprogramming capabilities for Macro tags

MyBatis splices dynamic SQL in a clumsy way, so some MyBatis-like frameworks provide some specially designed simplified syntax at the SQL template level. For example, some frameworks introduce an implicit condition determination mechanism.


```sql
select xxx
from my_entity
where id = :id
[and name=:name]
```

By automatically analyzing the variable definition in the brackets, an implicit conditional judgment is automatically added, and the corresponding SQL fragment is output only when the attribute value of name is not null.

In NopOrm, we can implement a similar ** Local Syntactic Structure Transformation **


```xml
<sql>
  <source>
    select o from MyEntity o
    where 1=1
     <sql:filter> and o.classId = :myVar</sql:filter>
  </source>
</sql>
```

 `<sql:filter>` Is a macro tag, which is executed at compile time and is equivalent to transforming the structure of the source code, equivalent to the following handwritten code


```xml
<c:if test="${!_.isEmpty(myVar)}">
   and o.classId = ${myVar}
</c:if>
```

For the implementation of specific tags, refer to

[sql.xlib](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/resources/_vfs/nop/orm/xlib/sql.xlib)

Essentially, this concept is equivalent to a Lisp macro, and in particular, like Lisp macros, it can be used in any part of the program code (that is, any node of the AST can be replaced with a macro node). However, it takes the form of XML, which is more humane than Lisp's mathematical notation style.

The LINQ (Language Integrated Query) syntax of Microsoft C # language is implemented by obtaining the abstract syntax tree object of the expression at compile time, and then submitting it to the application code for structural transformation, which is essentially a macro transformation technology at compile time. In the XLang language, you can use XScript's macro functions to convert between SQL syntax and object syntax, in addition to the macro tags provided by the Xpl template. For example


```xml
<c:script>
function f(x,y){
    return x + y;
} 
let obj = ...
let {a,b} = linq `
  select sum(x + y) as a , sum(x * y) as b
  from obj
  where f(x,y) > 2 and sin(x) > cos(y)
`
</c:script>
```

XScript's template expressions automatically recognize macro functions and execute them automatically at compile time. So we can define a macro function LINQ that parses the template string into an SQL syntax tree at compile time, and then transforms it into a normal JavaScript AST, which is equivalent to embedding a DSL of SQL-like syntax in an object-oriented XScript syntax (a scripting language like Type Script). You can do something similar to LinQ, but the implementation is much simpler and the form is closer to the original form of SQL.

> The above is only a conceptual example. Currently, the Nop platform only provides macro functions such as XPath/jpath/xpl, and does not provide built-in LINQ macro functions.

### 9.4 SQL output mode of template language

The design bias of template language is to regard the side effect of Output as the first class concept compared with ordinary programming language. When we do not do any special decoration, it means external output, and if we want to express the execution of other logic, we need to use expressions, labels and other forms to clearly isolate it. As a Generic template language, Xpl template language strengthens the concept of output and adds the design of multi-mode output.

The Xpl template language supports multiple Output modes.

* Text: Output of plain text without additional escaping

* XML: Output of XML-formatted text, automatically escaped according to the XML specification

* Node: The output of the structured AST, which preserves the source location

* SQL: Support the output of SQL objects to prevent SQL injection attacks

The SQL mode makes special treatment for the SQL output, and mainly adds the following rules

1. If the object is output, it is replaced by? And the object is collected in the parameter collection. For example ID = \ ${ ID } will actually generate ID =? With a List to hold the parameter values.

2. If you output a collection object, it is automatically expanded to multiple parameters. For example, ID in (\ ${ ids }) corresponds to ID in (?,?,?).

If you really want to output the SQL text directly, spliced into the SQL statement, you can use the raw function to wrap it.


```
from MyEntity_${raw(postfix)} o
```

In addition, NopOrm has a simple wrapper model for parameterized SQL objects themselves.


```
SQL = Text + Params
```

Via SQL = SQL. Begin ().sql ( "o. ID =?"? ", name).end () This form can construct a SQL statement object with parameters.". The SQL output mode of the Xpl template automatically recognizes SQL objects and automatically processes text and parameter collections separately.

### 9.5 Automatic verification

One drawback of managing SQL templates in external files is that it cannot rely on the type system for validation and can only expect runtime tests to check for proper SQL syntax. If the data model changes, it may not be immediately obvious which SQL statements are affected. There are some relatively simple solutions to this problem. After all, now that SQL statements have been managed as a structured model, the means by which we can manipulate them have become extremely rich. NopOrm has a built-in mechanism similar to Contract Based Programming: the model for each EQL statement supports a validate-input configuration where we can prepare some test data. When loading sql-lib, the ORM engine will automatically run validate-input to get the test data, execute the SQL template based on the test data to generate EQL statements, and then submit them to the EQL parser to analyze their validity. O that the consistency of the ORM model and the EQL statement is checked in a quasi-static analysis mode.

### 9.6 Debugging support

Unlike MyBatis's built-in homemade simple template language, NopOrm uses the Xpl template language to generate SQL statements, so it's natural to use the XLang language debugger to debug. The Nop platform provides IDEA development plug-ins that support DSL syntax hinting and breakpoint debugging. It can automatically read the SQL-lib. Xdef metamodel definition file, automatically verify the syntax correctness of the SQL-lib file according to the metamodel, provide syntax prompt function, support adding breakpoints in the source section, and perform single step debugging.

All DSLs in the Nop platform are built based on the principle of reversible computation, and they are described by a unified metamodel definition language XDefinition, so it is not necessary to separately develop IDE plug-ins and breakpoint debuggers for each DSL. To add IDE support for the user-defined sql-lib model, the only thing required is to add the attribute X: schema = "/NOP/schema/orm/sql- lib. Xdef" "on the root node of the model and introduce the xdef metamodel.

The XLang language also has some built-in debugging features that make it easy to diagnose problems during the metaprogramming phase.

1. The AST node generated in the Output Mode = node output mode will automatically retain the line number of the source file, so when the generated code is compiled and reports an error, we directly correspond to the code location of the source file.

2. The xpl: dump attribute can be added to the Xpl template language node to print out the AST syntax tree obtained after the current node is dynamically compiled

3. Any expression can be appended to call the extension function \ $, which automatically prints the text, line number, and result of the current expression, and returns the result value of the expression. For example


```
x = a.f().$(prefix) 实际对应于
x = DebugHelper.v(location,prefix, "a.f()",a.f())
```

## Ten. GraphQL over ORM

If we look at it from a more abstract point of view, the way of interaction between the foreground and the background is nothing more than: ** Request the business method M on the background business object O, pass it the parameter X, and return the result Y **. If you write this as a URL, you get a similar result


```
view?bizObj=MyObj&bizAction=myMethod&arg=X
```

Specifically, bizObj can correspond to a background Controller object, bizAction corresponds to a business method defined on the Controller, and view represents the result information presented to the caller. Its data source is the data obtained by the business method. For normal AJAX requests, the format of the returned JSON data is uniquely determined by the business method, so it can be written as a fixed JSON. For general RESTful services, the choice of view can be more flexible. For example, you can decide whether to return JSON format or XML format according to the contentType header of Http. If the view is uniquely determined by the business object and method of the request, we say that the Web request is in push mode, and if the client can select the returned view, we say that the corresponding Web request is in pull mode. Based on this knowledge, we can think of GraphQL as ** Composable Pull-mode Web Request **.

The most significant difference between GraphQL and a normal REST or RPC request is that its request mode corresponds to


```
selection?bizObj=MyObj&bizField=myField&arg=X
```

GraphQL is a pull mode request that specifies the result data to be returned. However, this designation is not completely new, but ** Selection and partial reorganization (renaming) based on existing data structures **. It is precisely because the selection information is highly structured that it can be parsed ahead of time and become a blueprint to guide the execution of the business method. Also because it is highly structured, business requests for multiple business objects can be grouped together in an orderly fashion.

In a sense, the logical structure of a Web framework is virtually unique. In order to achieve effective logical splitting, we must distinguish different business objects in the background, and in order to achieve flexible organization, we must specify the returned view. The corollary is that the URL should be of the form

> Many years ago, I wrote an article that analyzed the design principles of the Web MVC framework: [ The past and present of WebMVC ](http://www.blogjava.net/canonical/archive/2008/02/18/180551.html). The article's analysis is still valid today.

Based on the above knowledge, the combination of GraphQL and ORM can be very simple. In the Nop platform, GraphQL services can be directly mapped to the underlying ORM entity objects through deterministic mapping rules, and the GraphQL services can be run without programming. On the basis of this automatic mapping rule, we can gradually supplement other business rules, such as permission filtering, business process, data structure adjustment and so on. Specifically, the code generator automatically generates the following code for each database table as an alternative business object:


```
/entity/_MyObj.java
       /MyObj.java
/model/_MyObj.xmeta
      /MyObj.xmeta
      /MyObj.xbiz
/biz/MyObjBizModel.java
```

* The MyObj. Java is an entity class automatically generated according to the ORM model definition, and we can directly add auxiliary attributes and functions to the entity class.

* The MyObj. Xmeta is an externally visible business entity data structure, which is used by the system to generate the Schema definition of GraphQL objects.

* Custom GraphQL service response functions and data loaders are defined in the MyObjBizModel. Java.

* MyObj. Xbiz involves more complex concepts of business aspects, which will not be repeated in this article.

GraphQL and ORM essentially provide different levels of information structure. GraphQL is for external perspectives, while ORM is more focused on internal usage within the application, so by necessity they don't share the same Schema definitions. However, in general business applications, they are obviously similar and have great commonality. ** Reversible computing provides a standardized solution for dealing with information structures that are similar but not identical **。

In view of the above situation, the design of Nop platform is that `_MyObj.java` and `_MyObj.xmeta` are generated directly according to the ORM model, and the information between them is completely synchronized. The MyObj. Java inherits from `_MyObj.java`, to which you can add additional properties and methods that are visible within the application. In the MyObj. Xmeta, the X: extends variance merging mechanism `_MyObj.xmeta` is used to customize and support ** Addition, modification and deletion ** the definition of object attributes and methods. At the same time, we can also specify auth permission checking rules in xmeta and rename attributes. For example


```xml
<meta>
  <props>
    <prop name="propA" x:override="remove" />
    <prop name="propB" mapTo="internalProp">
      <auth roles="admin" />
      <schema dict="/app/my.dict.yaml" />
    </prop>
  </props>
</meta>
```

In the above example, the propA attribute will be removed, so it cannot be accessed by GraphQL queries. At the same time, the internalProp attribute is renamed to propB, that is, when GraphQL queries propB, the internalProp attribute is actually loaded. PropB is configured with auth roles = admin, which means that only administrators have access to this attribute. The dict configuration in schema indicates that its value is limited to the range of dictionary table my. Dict. Yaml. In Section 5.2, we introduced the dictionary table translation mechanism in NopOrm: In the metaprogramming phase, the underlying engine finds the dict setting and automatically generates a propB _ text field that returns the internationalized text translated from the dictionary table.

For the topmost GraphQL object, the Nop platform automatically generates the following structure definition:


```graphql
extend type Query{
    MyObj__get(id:String): MyObj
    MyObj__findPage(query:String): PageBean_MyObj
    ...
}
```

In addition to the default operations such as get/findPage, we can define extension properties and methods in MyObjBizModel.


```java
@BizModel("MyEntity")
public class MyEntityBizModel {

    @BizLoader("children")
    @BizObjName("MyChild")
    public List<MyChild> getChildren(@ContextSource MyEntity entity) {
        ...
    }

    @BizQuery("get")
    @BizObjName("MyEntity")
    public MyEntity getEntity(@ReflectionName("id") String id, IEvalScope scope,
                              IServiceContext context, FieldSelectionBean selection)     {
       ...
    }

    @BizQuery
    @BizObjName("MyEntity")
    public PageBean<MyEntity> findPage(@ReflectionName("query") QueryBean query) {
        ...
    }
}

@BizModel("MyChild")
public class MyChildBizModel {

    /**
     * 批量加载属性
     */
    @BizLoader("name")
    public List<String> getNames(@ContextSource List<MyChild> list) {
        List<String> ret = new ArrayList<>(list.size());
        for (MyChild child : list) {
            ret.add(child.getName() + "_batch");
        }
        return ret;
    }
}
```

In BizModel, the GraphQL Query and Mutation operations are defined through @ BizQuery and @ BizMutation respectively. The format of the GraphQL operation name is `{bizObj}__{bizAction}`. Meanwhile, we can add the fetcher definition of GraphQL through @ BizLoader, introduce the parent object instance of GraphQL through @ ContextSource, mark the argument through @ ReflectionName, and automatically convert the type during parameter mapping.

> If functions such as get/findPage are also defined in the BizModel, the implementation of the default functions such as MyObj _ _ get will be overridden.

There are only business objects, business methods and business parameters in the design space of BizModel, and it is completely decoupled from GraphQL, so we can easily provide REST service bindings or other RPC calling interface standard bindings for BizModel. In our specific implementation, we even provide a batch file binding for it, that is, the background batch task runs regularly, parses the batch file to get the request object, then calls BizModel to execute the business logic, and writes the returned object as a result into the result file. The key to the design is to realize the optimization of batch processing, that is, the batch processing task processes 100 records in each batch, and the database should be updated at one time after the whole batch is completely processed, instead of updating the database immediately after processing each business request. Thanks to the ORM engine's session mechanism, this batch optimization is completely free.

## Conclusion

The theory of reversible computation is not a clever design pattern, nor is it a set of lessons learned based on best practices. It is an innovative technical idea about large-scale software structure construction, which is rooted in the real physical laws of our world and based on strict logical derivation from the first principle.

Under the guidance of reversible computing theory, NopOrm's technical solution reflects the completeness and consistency of the underlying logical structure, which enables it to solve a series of difficult technical problems in a straightforward and simple way. (In many cases, the complexity of the system is not very high, but the structural barriers and conceptual conflicts that arise when multiple components are configured with each other lead to a lot of accidental complexity.)

NopOrm is not an ORM engine dedicated to low code, it supports a smooth transition from LowCode to ProCode. It not only supports code generation in the development period, but also supports the dynamic addition of fields in the running period, and provides a complete solution for user-defined storage.

It is very lightweight, and with the main features of Hibernate + MyBatis + SpringData + JDBC + GraphQL, the effective amount of hand-written code is less than 20000 lines (and a lot of code is automatically generated. Because the Nop platform is trying to develop all of its components in a low-code way. It is not only suitable for the development of small single projects, but also suitable for the development of distributed, high-performance, high-complexity large-scale business systems. It also provides some syntax support for BI systems, and supports the compilation of native applications through GraalVM.

NopOrm follows the principle of reversible computing, and can customize and enhance the underlying model through Delta customization and metaprogramming. Users can continuously accumulate reusable domain models in their own business domains, and even develop their own proprietary DSLs.

What's more, NopOrm is open source! (It is still in the code consolidation stage and will be released with Nop Platform 2.0.)

Finally, for the students who can insist on seeing here, it is really not easy, and I sincerely praise your studious spirit!