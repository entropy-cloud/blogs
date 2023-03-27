# What ORM engine is required for low code platforms? (1)

Low-code platforms try to minimize the amount of hand-written code, and the core tools they can rely on must be all kinds of explicitly established information models, such as data models, form models, process models, report models and so on. Among them, the data model is undoubtedly the most important. As an ORM (Object Relational Mapping) engine based on a data model, what value does it bring to low-code platforms?

To answer this question, we need to go back to the basic concept of ORM: What is ORM? Why can ORM simplify the coding of the data access layer? What common business semantics can be expressed in the ORM layer? In the context of a low-code platform, the data structure needs to support user-defined adjustments, and the logical path from the front-end presentation interface to the back-end data storage needs to be compressed as much as possible. What support can the ORM engine provide for this? If we are not satisfied with some low-code application scenarios defined in advance, but want to achieve a smooth upgrade path from Low Code to ProCode, what requirements will we put forward for the ORM engine?

Based on the reversible computing theory, this paper makes a preliminary theoretical analysis of the design and implementation of ORM engine, and introduces the implementation scheme of NopOrm engine used in Nop Platform 2.0. NopOrm roughly contains the main functions of Hibernate + MyBatis + Spring Data JDBC + GraphQL, but because it uses a large number of innovative designs and makes certain trade-offs on functional features based on theoretical analysis, Therefore, the actual amount of effective code that needs to be written by hand is not large (about 20,000 lines or less). On the basis of relatively simplified code implementation, NopOrm actually provides more feature support for business development. At the same time, based on the general underlying solution of reversible computing theory, NopOrm provides flexibility and scalability that other ORM engines can not achieve for free.

## One. What is ORM?

What is ORM? On hibernate's website, there is an authoritative explanation of this problem for many years: [ What is Object/Relational Mapping ](https://hibernate.org/orm/what-is-an-orm/). Hibernate's explanation is that ORM solves the problem of the so-called object-relational impedance (Object-Relational Impedance Mismatch), that is, there is a mismatch between the relational paradigm and the object paradigm, and a framework is needed to solve the adaptation problem. Specifically, there are five mismatches:

1. Granularity: Relational databases manage data at the granularity of tables and fields, while object models can adopt richer management structures. For example, the address information in the user table may be split into multiple fields, which can be mapped to the Address component class.

2. Inheritance: In object-oriented programming languages, class inheritance is widely used to reuse existing concepts and implement specific functions, while in relational databases, there is a lack of similar means to achieve reuse.

3. Identity: The relational model distinguishes different objects by primary keys, while different objects in the object model correspond to different object pointers. There is conceptual inconsistency between the two.

4. Associations: The relational model uses foreign keys to express associations between records, while the object model uses object attributes to express associations.

5. Data Navigation: In the object model, we can traverse the entire object graph in the form of property access through a. B. C, while in the relational model, we need to explicitly specify the associated tables, associated fields, and how they are associated with each other.

Do the five aspects proposed by Hibernate reflect the essence of ORM? Here I want to analyze it from a different angle.

First of all, if a technology has essential advantages, it must be ** Makes better use of certain information than other alternatives **, not just to solve some form of adaptation problem. What information does a better ORM make better use of than a mediocre one? This involves the biggest secret in relational database theory: ** There are no relationships in a relational database **! Although the relational database is always talking about the word "relationship", ** The truth is that relational databases store unrelated, independent atomic data after the relationship has been decomposed! **

If we set up the table model strictly according to the third normal form in the relational database theory, modifying the value of any field will not affect the value of other fields in principle!

> Baidu Encyclopedia: Third Normal Form (3rd NF) ** All data elements in the table must not only be uniquely identified by the primary key, but also be independent of each other, and there is no other functional relationship between them. ** refers to. That is to say, for a data structure that satisfies 2nd NF, there may be some data elements in the table that depend on other non-key data elements, which must be eliminated

On the surface, primary and foreign key fields appear to be defined in the database, but they do not play a role in true logical expression, except for some integrity checking. The reason why using SQL language to access complex associated data is verbose is ** Every time we access data, we need to specify which fields of which tables need to be associated according to what conditions. ** that the associated information is explicitly injected into the system through code when we access the data, and does not belong to the built-in knowledge of the system. In fact, many developers of large software systems have even passed on an old secret: Don't set up foreign key associations on large tables; they affect program performance. ** Foreign key associations are not useful for application development at all, or if they are, they are negative! **

** The relational model uses a symmetric access pattern. ** That is, all tables and all fields are equally weighted, and there is essentially no difference between them. Through the join statement, we can establish an association between any field of any table, and is not restricted by the concepts of primary key and foreign key. The reason why we tend to use the primary key to read records is essentially because there is a primary key index on the primary key, which can speed up access. We can also create unique indexes on other fields, and the primary key does not have any exclusive specificity.

When we gradually move from a business-independent and general storage model to an application model that is convenient for business processing, we will inevitably find that some fields have special and important meanings in business (symmetry is broken), and the relationship between them is relatively stable and frequently used, so there is no need to repeat the expression every time.

> The conceptual uniformity and universality of the relational model are often considered to be the beauty of the theory. But the real world is complex, and the direction of development is to gradually identify the differences and find natural forms of expression to express these differences.
> 
> The uniform relational model is the simplest model with the highest symmetry. In the face of physical constraints, it implicitly assumes that little interaction occurs between collections, with single tables (the mapping between forms to data tables) and master-detail tables being the broadest cases. Try to imagine a relational model, in which we generally only see two data tables, and when considering multiple tables, because there is no clear distinguishability between these tables, their image is modular.
> Pasty. Only when we are clearly aware of the different concepts of primary key, foreign key, primary table, subordinate table, dictionary table, fact table and latitude table, when the symmetry is broken, can the model in our thinking be enriched.
> 
> The star schema and snowflake schema established in the data warehouse theory emphasize the relational decomposition that allows partial redundancy for the subject domain. It actually emphasizes the inequivalence between tables, not all of them.
> The tables are all in the same position. The difference between Fact Table and Dimension Table is recognized and explicitly handled. From complete decomposition to partial decomposition, we can form a model level column. At different levels of complexity, we can choose specific implementation models according to the guidance of theory.

** The special value of ORM is that it recognizes the particularity of primary key and foreign key, and realizes the intrinsic expression and full use of pairwise relationship. **。

First, the primary key has a special meaning in ORM, and it becomes the key of the object cache. Through object caching, ORM can ensure that objects with consistent primary keys actually correspond to the same object pointer, thus automatically maintaining the identity relationship of a. B. C. A = = a, that is, traversing the object graph through different attribute paths can ensure that the same object node is reached.

Second, the foreign key association information is solidified and reused in the ORM. Consider the following SQL statement


```sql
select * from a, b
where a.fldA = b.fldB
and a.fldC = 1 and b.fldD = 2
```

A. FldA = B. FldB may be referred to as a correlation condition, and a. FldC = 1 and B. FldD = 2 may be referred to as coordinate conditions. A large part of the complexity of SQL comes from the frequent need to specify exactly the same association conditions everywhere without being able to abstract them into reusable components. In the object space provided by ORM, the pairwise association between objects can play a role in various operations such as addition, deletion, modification and query as long as it is specified once. Especially in the object query statement, the association relationship between multi-entities can be automatically derived through the pairwise association, that is, a. B. C. D = 3 can be automatically derived as


```sql
select ... 
from A a join B b on a.xx = b.id join C c on b.yy = c.id
where c.d = 3
```

With the help of automatic attribute association, the scope of application of the single table model has been greatly expanded: any single table automatically becomes a subject table, and any field on the associated table automatically becomes a field that can be directly accessed on the subject table. For example, if we place a query field a. B. C in the foreground, it can be exactly the same as placing a query field d in the background query processing pipeline. If the domain driven design (DDD) is adopted, it is easy to implement the so-called Aggregate Root pattern based on the subject table entity object.

Based on the above theoretical analysis, we can find that among the five aspects proposed by Hibernate, granularity and inheritance are relatively minor concepts, and we do not necessarily need to invest a lot of energy in the core of the engine! On the other hand, a data access engine like MyBatis lacks a query language that can exploit object associations, and it is certainly not a complete ORM.

> In the actual development process, we can use composition instead of inheritance (the development of object-oriented technology in recent years has been advocating that composition is superior to inheritance). In fact, in the Java language, the combination of inheritance and lazy loading is itself a conceptual contradiction. A proxy object may need to be created before the entity is actually loaded, but its type is undetermined at this time, and after the proxy is loaded lazily, the ORM engine cannot convert it to a specific object type without guaranteeing the uniqueness of the object pointer.

## Two. EQL = SQL + AutoJoin

A long-standing criticism of ORM engines is that the object query syntax is very limited, especially the support for multi-table union queries with non-primary foreign key associations is poor, and arbitrary associations between arbitrary tables are not supported. Query statements such as select * from (select XXX) with subquery as the data source are also not supported. However, is this a problem in the nature of the ORM engine, or is it a problem in the concrete implementation of Hibernate?

According to the theoretical analysis in the previous section, the object query language that can make full use of object association is one of the essential values of ORM engine, so what should realize this essential value ** Minimized Object Query Language **? The EQL (Entity Query Language) in the NopOrm engine is ** Superset of the SQL language ** defined as, which is based on the SQL query syntax (in theory, it can support all SQL syntaxes) and adds a minimal extension of object-related attributes. EQL abandons all the object-specific query syntax introduced by Hibernate, and only adds the processing of attribute association syntax such as a. B. C, so it is very similar to the traditional SQL language in use, and can naturally support the following query statements:


```sql
with a as (
  select o.u ...
)
select a.*, b.d
from a, (select c.xx, c.d from C c where c.d.e > 3) b 
where a.u = b.xx
limit 3 offset 2
```

In NopOrm, the execution of SQL and EQL is abstracted into a unified interface ISqlExecutor, and the results they return are encapsulated into the IDataSet interface (a replacement for JDBC's ResultSet). The only difference at the usage level is that the result fields returned by EQL may be objects or collections of objects, not just atomic data types. For the specific definition of the interface, refer to

[ISqlExecutor](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-dao/src/main/java/io/nop/dao/api/ISqlExecutor.java)

The extension of EQL to the SQL language involves only two areas:

1. Like the from MyEntity o left join o. RelField, the association method is actively specified in the from statement

2. Similar to the o. A. B. C, the associated properties of the object are accessed as qulified name

> There is a special handling rule here: normally o. A. B are translated as inner join associations between tables, but for order by o. A. B and o. A. B not used elsewhere, a left join association is preferred. Avoid using inner join when the o. A is null to affect the number of entries in the result set.

Because the translation of the object-property association syntax is basically orthogonal to the rest of the SQL syntax and can be wrapped into a separate AST Transformer, adding new syntax support to the SQL language does not affect the translation of EQL syntax. If we use the AST automatic parsing technology described in the following article, we can even automatically realize the conversion from EQL to SQL by modifying the G4 syntax definition of antlr. EQL compatibility with all SQL syntax becomes a relatively simple task.

[How Antlr4 automatically parses to get AST instead of ParseTree] (https://zhuanlan.zhihu.com/p/534178264)

At present, many low-level frameworks need to parse SQL statements to obtain data structure information, such as

1. Ali [ Druid Database Connection Pool ](https://github.com/alibaba/druid/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98) needs to parse SQL statements to prevent SQL injection attacks and implement SQL auditing.

2. [ Apache ShardingSphere ](https://shardingsphere.apache.org/index_zh.html) Need to parse SQL statements to achieve sub-database sub-table and data encryption and other functions.

3. Ali [Seata](https://seata.io/zh-cn/) needs to parse SQL statements to implement distributed transactions in AT mode.

Now that the ORM engine has implemented EQL parsing (a superset of the SQL language), similar functionality can be implemented at a fraction of the cost. Even if the frameworks themselves are well layered and isolated, it should be possible to integrate them directly into the execution engine of EQL.

Another long-standing myth about the ORM object query language is that the object query language translation process is opaque and the resulting SQL statements look "ugly.". For example


```sql
select
author0_.id as id1_0_0_,
book2_.title as title3_1_1_,
books1_.bookId as bookId1_2_0__,
from
Author author0_
left outer join
Book book2_
on books1_.bookId=book2_.id
left outer join
Publisher publisher3_
on book2_.publisherid=publisher3_.id
where
author0_.id=100
```

If all the object-specific query syntax introduced by HQL is abandoned and only the object attribute associations are retained, the EQL syntax is no more complex and obscure than the ordinary SQL syntax. In fact, its correspondence to SQL syntax is so straightforward that you can access all entity data using just the SQL syntax (which is the legal EQL syntax). For example, we can use


```sql
select s.statusName
from MyUser o, MyStatus s
where o.statusId = s.id

或者
select o.status.statusName
from MyUser o
```

As for why a large number of automatically generated aliases were introduced into the SQL statements translated by HQL, which made the SQL statements less friendly, the Hibernate team gave an explanation in the April 2022 release [ Hibernate 6.0 ](https://www.infoq.com/news/2022/04/red-hat-releases-hibernate-6/): Hibernate always reads data from a ResultSet by column name, so you need a unique alias for each column. In Hibernate 6.0, it has been changed to read data by column subscripts, so there is no need to generate aliases anymore! Through this modification, the performance of Hibernate 6.0 has been further improved.

To be honest, this explanation sounds a little embarrassing. Why does this happen? Perhaps this is a legacy of Gavin King's unfamiliarity with the JDBC API when he implemented the first version of Hibernate in 2003.

## Three. Dynamic ORM mapping

In low-code platforms or general SAAS applications, there is a need for user-defined data storage. Because different users need to design different storage structures according to their own needs, we must provide a dynamic ORM mapping mechanism that can be customized at runtime.

In [从实现原理看低代码](https://zhuanlan.zhihu.com/p/451340998) this article, Wu Duoyi introduced several common user-defined storage schemes with low back-end code.

1. Direct Mapping of Relational Databases via Dynamic Entities

2. Use a document database

3. Use rows instead of columns, that is, table horizontal to table vertical

4. Use meta information + wide table to reserve a large number of fields

5. Use a single file

In the Nop platform, the above five solutions can be directly implemented with the help of the NopOrm engine, and these five methods can coexist in the same OrmSession, that is to say, we can save part of the entity data in a common database table, and save part of the data in a vertical table. Part of the data is saved in the Redis cache or ElasticSearch document database, while the other data is saved in the data file. At the use level, they are all ordinary Java objects and form a unified object graph. At the application level, it is impossible to identify which storage mechanism is used at the bottom. When appropriate, we can even switch the data storage mode. For example, in order to avoid modifying the database at the beginning, we use the vertical table to store the extended data. With the growth of data volume and the gradual stabilization of business logic, we can switch to the storage in the form of ordinary data table or wide table. The original object structure can be kept unchanged in the application layer without any change.

### 3.1Use a relational database directly

NopORM supports dynamic attribute configuration. When an attribute defined in an entity model is not defined in a Java entity class, it will be stored as a dynamic attribute and converted to a data type according to the type specified during definition. There is no difference between NopORM and ordinary Java attribute fields in all application levels.


```xml
<entity name="io.nop.app.SimsExam" className="实现类，一般与entityName相同">
    <columns>
        ...
        <column name="examScoreScale" propId="20" code="EXAM_SCORE_SCALE" stdSqlType="TINYINT"/>
        <!-- 不生成java实体代码 -->
        <column name="extField" propId="21" code="EXT_FIELD" 
            stdSqlType="INTEGER" notGenCode="true"/>
    </columns>
</entity> 
```

In the above configuration, if the examScoreScale and extField attributes exist on the SimsExam entity class, the corresponding get/set methods will be used to access the attributes. Property values are placed in the dynamicValues property collection of the base DynamicOrmEntity class.

The notGenCode = true is specified in the model definition of extField, which means that when the Java entity code is generated according to the orm. XML model definition, the get/set method will not be generated for this field, so it will always be accessed as a dynamic property.

If we do not need to generate code, we can access all fields as dynamic properties by specifying the implementation class as the io. NOP. Orm. Support. DynamicOrmEntity for the entity via the className attribute.

The Nop platform has built-in support for a MethodMissing mechanism similar to the Ruby language, allowing properties to be added to objects dynamically. In Java code, we can get the value of a dynamic property through a BeanTool. Get Property ( "extField") or a entity. Prop _ get ( "extField").

Nop platform's built-in scripting language XScript recognizes the IPropGetMissingHook and IPropSetMissingHook extension interfaces, so accessing dynamic entity properties in the script code or expression language is the same as accessing ordinary properties.


```java
entity.extField = 3;
let x = entity.examScoreScale;
```

### 3.2 Using a Documented Database

In NopOrm's entity model definition, you can specify a different persist driver for each entity type, such as persistDriver = "elasticS earch" "to indicate that the ElasticS earchEntityPersist Driver will be used to access the entity. It corresponds to the IEntityPersistDriver interface in the ORM engine and supports bulk and asynchronous entity data access.

[IEntityPersistDriver](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/main/java/io/nop/orm/driver/IEntityPersistDriver.java)

At the same time, for the data query of single entity, NopOrm makes a unified encapsulation through the IEntityDao. FindPage (QueryBean) function. If the PersistDriver implements the IEntity DaoExtension interface, the application layer can use the complex query capabilities provided by the underlying Driver through the IEntityDao interface.

[IEntityDao](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-dao/src/main/java/io/nop/dao/api/IEntityDao.java)

Another extension is to use a text field in a relational database to store a JSON string, and then parse the JSON string into a Map when you use it. For example


```xml
<entity name="io.nop.app.SimsClass">
    <columns>
      ...
      <column name="jsonExt"  code="JSON_EXT" propId="101" 
           stdSqlType="VARCHAR" precision="4000" />
    </columns>

    <components>
       <component name="jsonExtComponent" needFlush="true" className="io.nop.orm.support.JsonOrmComponent">
         <prop name="_jsonText" column="jsonExt"/>
       </component>
    </components>
 </entity>
```

In the above example, we used the Component mechanism in the NopOrm engine to parse the jsonText field into a Map object, and we can access the corresponding properties in the program as follows


```java
BeanTool.getProperty(entity,"jsonExtComponent.fld1")
```

If you find Component configuration a bit tedious, you can take advantage of the built-in metaprogramming capabilities of the Nop platform to simplify it. For example, the following configuration may be substituted


```xml
<orm x:schema="/nop/schema/orm/orm.xdef"
     xmlns:x="/nop/schema/xdsl.xdef" xmlns:xpl="/nop/schema/xpl.xdef">

    <x:post-extends>
        <orm-gen:JsonComponentSupport xpl:lib="/nop/orm/xlib/orm-gen.xlib" />
    </x:post-extends>

    <entities>
        <entity name="io.nop.app.SimsClass">
            <columns>
                <column name="collegeId" propId="100" lazy="true"/>
                <column name="jsonExt"  code="JSON_EXT" propId="101" tagSet="json" stdSqlType="VARCHAR" precision="4000" />
            </columns>
            <aliases>
                <alias name="extFld1" propPath="jsonExtComponent.fld1" type="String"/>
            </aliases>
        </entity>
   </entities>
</orm>
```

 `<orm-gen:JsonComponentSupport>` The tag will recognize the tag on `tagSet="json"` the field and automatically generate the corresponding JsonComponent configuration for that field. At the same time, we can take advantage of the alias configuration to simplify the attribute names used by the application layer. With the above configuration, we have the following equivalence between the XScript script and the EQL query language


```
entity.extFld1 == entity.jsonExtComponent.fld1
```

Alias can provide a short property name for a complex property path, thus masking the underlying concrete storage structure.

The design idea of Hibernate is to deduce the storage structure of relational database based on object paradigm. NopOrm's design idea is contrary to this, its processing strategy is a positive design based on database design, following the relational system paradigm, from simple to complex, first mapping all atomic fields of the database table through column, and then gradually constructing more complex ComponentProperty. Intertwined object structures such as Computed Property, Entity Reference Property, and EntitySetProperty.

Hibernate, which is based on object paradigm, has essential difficulties in dealing with complex data relationships. For example, if you have multiple components and associated properties that all map to the same database field, they can cause data conflicts. In this case, which component sets the value of the attribute we are going to use? The secret of solving data conflicts in relational databases is that when all data structures are decomposed into atomic data types, all conflicts disappear automatically. A large part of the complexity of the Hibernate implementation code is the need to maintain a verbose two-way mapping from tangled object structures to clean, independent database fields.

The disadvantage of using Json Componet to implement extended storage is that it does not support queries and sorting very well. If the underlying database supports JSON data types, you can do a local transformation in the EQL AST Transformer to translate entity attribute access in EQL syntax, such as the entity. JSON ExtComponent. Fld1, into the JSON attribute access supported by the database. For example `json_extract(entity, "$.fld1")`

### 3.3 Use rows instead of columns

Rows and columns in a relational database are asymmetric, so it is easy to add rows and the number is not limited, but the number of columns is generally very limited, and the operation of adding/deleting columns is a very expensive operation (this may change with the popularity of columnar databases). If we want to get a model with symmetrical rows and columns, we can use the so-called vertical table scheme.


```
rowId colId value
```

We can set up an extended table with only three fields, rowId and colId can be regarded as a symmetrical coordinate system, corresponding to row coordinates and column coordinates respectively, and value is the value at a given position in the coordinate system.

The data structure may be more complicated in the specific implementation, such as adding a fieldType column to mark the actual data type corresponding to value, adding multiple value fields to facilitate the implementation of correct sorting, and facilitating the use of built-in date operation functions in the database.


```
class OrmKeyValueTable{
    String entityId;
    String fieldName;
    byte fieldType;
    String stringValue;
    Integer intValue;
    BigDecimal decimalValue;
    DateTime dateTimeValue;
}
```

How do I convert rows to columns? At the object level, this is equivalent to how a record in a list is converted to an extended property of the object. In the theory of reversible computation, this is actually a standard structural transformation operation: ** For any collection structure, we can specify a keyProp attribute for the collection element to convert it into an object attribute structure. **.

For example, if keyProp = name is set, the entity. ExtFields. MyKey may be translated to

> The existence of a keyProp is the key to defining a stable system of domain coordinates. For example, in the virtual DOM Diff algorithm of the foreground, in order to stably and quickly identify the changed components, we need to specify `v-key` attributes for the components.

For specific configuration examples, please refer to

[app.orm.xml](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-orm/src/test/resources/_vfs/nop/test/app.orm.xml)


```xml
<entity name="io.nop.app.SimsExam">
    ...

    <aliases>
        <alias name="extFldA" propPath="ext.fldA.string" type="String"/>
        <alias name="extFldB" propPath="ext.fldB.boolean" type="Boolean" notGenCode="true"/>
    </aliases>

    <relations>
        <to-many name="ext" refEntityName="io.nop.app.SimsExtField" keyProp="fieldName">
            <join>
                <on leftProp="_id" rightProp="entityId"/>
                <on leftValue="SimsExam" rightProp="entityName"/>
            </join>
        </to-many>

        <to-many name="examExt" refEntityName="io.nop.app.SimsExamExtField" keyProp="fieldName">
            <join>
                <on leftProp="examId" rightProp="examId"/>
            </join>
        </to-many>
    </relations>
</entity>
```

The above example demonstrates two vertical table designs. One is the global extension table, which supports storing the extension fields of all entity tables into one table and distinguishes different entities through the entityName field. The other is a dedicated extension table. A separate extension table can be created for each specific entity table. See SimsExamExtField.

If you combine the horizontal and vertical transformations with the alias attribute aliasing mechanism described in the previous section, you can further simplify the extension field. For example, extFlda in the above example actually corresponds to the ext. FldA. String. In the EQL query language,


```sql
select xxx from SimsExam o where o.extFldA = 'a'
-- 将被转换为
select xxx 
from SimsExam o left join SimeExtField f
   on f.entityId = and f.entityName = 'SimsExam'
where f.fieldName = 'fldA' and f.stringValue = 'a'   
```

Because row-column conversion is a built-in mechanism in the EQL AST Transformer, we can actually query and sort the vertical table fields, but the performance is lower.

The above row-column conversion capability is general in nature and is not limited to the conversion of KVTable. ** Any one-to-many child table can be converted to the association property of the primary table by specifying the keyProp property **。 For example

 `entity.orders.odr333.orderDate` Indicates to get the orderDate attribute of the order with the number odr333.

### 3.4 yuan Information + Wide Table

Because the ORM engine itself has a large amount of meta information, the meta information + wide table mode is actually supported by the general ORM engine. For example


```xml
<entity name="xxx.MyEntity" tableName="GLOBAL_STORE_TABLE">
   <columns>
      <column name="id" code="ID" stdSqlType="BIGINT" />
      <column name="entityName" code="ENTITY_NAME" 
          stdSqlType="VARCHAR" precision="100" fixedValue="MyEntity" />
      <column name="name" code="VALUE1" stdSqlType="VARCHAR" 
              precision="100" />
      <column name="amount" code="VALUE2" stdSqlType="VARCHAR" 
          precision="100" stdDataType="int" />
   </columns>
</entity>
```

In the above example, all entity data are stored in a unified GLOBAL _ STORE _ TABLE table. In order to store the data of the MyEntity entity, the value of the entityName column is set as a fixed string `"MyEntity"`. Also, value1 and value2 are renamed to name and amount. The type of the VALUE2 attribute in the database is VARCHAR, and the type in Java is Integer. By specifying the stdDataType attribute, we can clearly distinguish the data types of these two levels, and automatically implement the conversion between them. Based on the above definition, we can use the EQL syntax to query as if we were accessing a normal database table


```sql
select * from MyEntity o where o.name = 'a' and o.amount > 3
-- 会被翻译为
select * from GLOBAL_STORE_TABLE o
where o.ENTITY_NAME = 'MyEntity'
   and o.VALUE1 =  'a' and o.VALUE2 > '3'
```

With the help of the aliasing mechanism mentioned in the previous section, we can concatenate multiple one-to-one or one-to-many tables into a logical large-width table. For example


```xml
<entity name="xxx.MyEntityFacade">
   ...
   <aliases>
     <alias name="fldA" propPath="myOneToOneRel.fldA" type="String" />
     <alias name="fldB" propPath="myManyToOneRel.fldB" type="Integer" />
     <alias name="fldC" propPath="myOneToManyRel.myKey.fldC"
           type="Double" />
   </aliases>
</entity>
```

### 3.5 Use a single file

The NopOrm engine supports specifying a dedicated persistDriver for each entity. Therefore, in principle, data can be saved to a data file as long as the IEntity PersistDriver interface is implemented. If the IEntity DaoExtension interface is further implemented, it can support the composite query and sorting of the records in the data file.

In the Nop platform, the composite query condition for a single table or entity is abstracted as a QueryBean message object, which can be automatically converted into an executable query filter.


```java
Predicate<Object> filter = QueryBeanHelper.toPredicate(
            queryBean.getFilter(), evalScope);
```

So implementing a simple single entity storage model based on JSON or CSV files is not a very complicated thing.

With the development of data lake technology, a single data file has gradually developed into a substitute for a single database table with built-in indexes that can support operator pushdown. In the near future, integrating a feature-rich data file store like iceberg may become a simple matter.

## To be continued

There should be few students who can insist on seeing here. I decided to stop here for the first half of this article in order to avoid reading zero. In the second half of this article, I will continue to discuss the solution of performance-related N + 1 problems, as well as Dialect customization, GraphQL integration, visualization integration and other related technical solutions.

If you are not familiar with reversible computation theory, you can refer to my previous article.

[Reversible Computing: Next Generation Software Construction Theory] (https://zhuanlan.zhihu.com/p/64004026)

Technical Realization of Reversible Computation (https://zhuanlan.zhihu.com/p/163852896)

Low Code Platform Design from Tensor Product (https://zhuanlan.zhihu.com/p/531474176)