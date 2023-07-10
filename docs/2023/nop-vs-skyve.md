[Skyve](https://github.com/skyvers/skyve) Is an open source business software building platform written in the Java language. It supports rapid application development with no code and low code. Different database engines are supported: MySQL, SQL Server, and H2 database engine. The design of Skyve adopts a relatively traditional back-end low-code implementation scheme, which is also a popular low-code and no-code scheme at present. In this article, we will compare the design of Skyve with the design of Nop platform to help you understand the unique features of Nop platform.

## One. Multi-tenant customization

Skyve is a multi-tenant system, which provides an interesting feature: Customer Override, which simply means that each tenant can have its own customized configuration, so that each tenant can have its own unique function implementation.

Skyve does this by creating a model file in the/SRC/main/Java/customers/{ tenantId }/{ modelPath } directory, thus overwriting the corresponding file in the/SRC/main/Java/modules/{ moduleId } directory. Skyve's solution is similar to Docker's hierarchical file system design, in which each tenant is equivalent to a custom layer, with high-level files overlaying low-level files. Many low-code platforms essentially follow a similar customization scheme. However, if compared with the Delta customization mechanism implemented by Nop platform based on reversible computing theory, we can find that Skyve's scheme is only a very primitive AdHoc design, and does not really explore the ability of Delta customization.

1. Skyve's customizations are written specifically for each model file, rather than being based on a common differential file system concept. To add a new model file, Skyve needs to modify the FileSystemRepository implementation.
2. Loading model objects according to the file path is not abstracted into a unified ResourceLoader mechanism, and does not provide model parsing cache and resource dependency tracking (the model cache is automatically invalidated when the dependent file changes).
3. Instead of inheriting from the original file as the Nop platform does, the custom file overwrites the original file as a whole, including only the portion of the delta revision in the custom file.

In the Nop platform ** Load all model files in a unified way **

```
model = ResourceComponentManager.instance().loadComponentModel(resourcePath)
```

> The reference [ custom-model.md ](https://gitee.com/canonical-entropy/nop-entropy/blob/master/docs/dev-guide/model/custom-model.md) configures the model loader for the file type

For the file NopAuthUser. Xmeta/NOP/auth/model/NopAuthUser/, we can add a `/_delta/default/nop/auth/model/NopAutUser/NopAuthUser.xmeta` file, and when loading, the priority is to find the files in the _ delta directory. The default delta layer is enabled by default. We can explicitly specify the list of enabled delta layers through the NOP. Core. VFS. Delta -layer-ids parameter, that is ** Delta customization can be multi-layer. **, instead of Skyve's single-layer delta customization.

> Historically, we have used three layers of customization: platform -- customization and modification of platform-built features, product -- base product generic features, and app -- application-specific customization features.

In the customization file, we can use X: extends = "super" to indicate that the configuration of the previous layer is inherited, and we only need to add the delta description in this file.

```xml
<meta x:extends="super">
    <props>
      <!-- 删除基础模型中的字段 -->
      <prop name="fieldA" x:override="remove" />
      <prop name="fieldB">
         <!-- 为fieldB增加字典表配置 -->
         <schema dict="xxx/yyy" />
      </prop>
    </props>
</meta>
```

In addition to using X: extends = "super", we can explicitly specify the underlying model of the inheritance, for example

```xml
 <meta x:extends="/nop/app/base.xmeta">
 </meta>
```

** X: extends is a very efficient Tree decomposition mechanism that can also be applied to JSON files ** For example, for the front-end interface JSON, we can decompose a huge page into multiple subfiles in a similar way.

```json
{
  type: "page",
  body: {
     ...
     {
        type: 'action',
        dialog: {
           "x:extends" : "xxx/formA.page.json",
           "title" : "zzz", // 这里可以覆盖x:extends继承得到的属性
        }
     }
  }
}
```

## Two. Domain-specific model

Skyve's design goal is to define models through metadata rather than code as much as possible, so it provides several XML formats of domain models, such as Document, View, etc., so that we can use XML files to describe a large part of business logic without writing Java code.

Skyve uses XSD (XML Schema) language to standardize the format of XML model files, and then uses JAXB (Java Architecture for XML Binding) technology to implement XML parsing. Similarly, the Nop platform uses the meta-model definition language XDefinition to define the format of the model file, but its design idea is quite different from that of XSD:

### 1. Homomorphic design

XDef explicitly adopts the design idea of homomorphic mapping. The structure of XDef metamodel is consistent with the structure of the model itself, but some annotation information is added on the basis of the syntax structure of the model. For example [view.xdef](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-xdefs/src/main/resources/_vfs/nop/schema/xui/xview.xdef)

```xml
<!--
包含表单定义，表格定义，以及页面框架组织
-->
<view bizObjName="string" x:schema="/nop/schema/xdef.xdef" xdef:check-ns="auth"
      xmlns:x="/nop/schema/xdsl.xdef" xmlns:xdef="xdef">

    <grids xdef:key-attr="id" xdef:body-type="list">
        <grid id="!xml-name" xdef:ref="grid.xdef"/>
    </grids>
   ...
</view>
```

The model structure it describes is as follows:

```xml
<view x:schema="/nop/schema/xui/xview.xdef" bizObjName="NopAuthUser" 
      xmlns:x="/nop/schema/xdsl.xdef" xmlns:j="j">
    <grids>
        <grid id="list" >
            <cols>
                <!--用户名-->
                <col id="userName" mandatory="true" sortable="true"/>

                <!--昵称-->
                <col id="nickName" mandatory="true" sortable="true"/>
            </cols>
        </grid>
    </grids>
</view>  
```

Basically, you just need to use the original model file as a template and replace the specific values with the corresponding stdDomain definitions. For example, ID = "! Xml-name" indicates that the ID attribute is a non-empty attribute, and its format must meet the xml-name definition requirements, that is, it must conform to the XML name specification requirements.

> The StdDomain Registry. Register DomainHandler allows you to register a custom stdDomain

The XDef metamodel is so simple and intuitive that OpenAI's ChatGPT already understands its definition directly, see [ GPT-driven low-code platforms produce a validated strategy for complete applications ](https://zhuanlan.zhihu.com/p/614745000)

### 2. Executable type

In a schema definition language such as XSD or JSON Schema, only the basic data types are specified, but no code types with execution semantics are defined. In the XDef metamodel, we can specify a type such as stdDomain = expr/xpl to automatically parse the XML text into an expression object or an Xpl template object.

With the help of this mechanism, we can embed Turing-complete scripting language and template language into DSL. On the other hand, thanks to the compile-time macro processing capabilities in the Xpl templating language, we can seamlessly embed any domain-specific language in the templating language ** Seamless integration between general-purpose and DSL languages **.

The Nop platform provides an IDEA plug-in [ nop-idea-plugin ](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-idea-plugin). As long as the XDef metamodel definition is provided, the plug-in can automatically implement syntax hinting, syntax checking, link jumping, and in particular, it ** Breakpoint debugging capability is provided ** can step through DSL code. That is, we can easily develop a domain specific language (just by defining the XDef metamodel) and provide a set of development tool support for the domain specific language without special programming. See for [ plugin-dev.md ](https://gitee.com/canonical-entropy/nop-entropy/blob/master/docs/dev-guide/ide/plugin-dev.md) details

### 3. Domain coordinate system

In Skyve, XSD is only used as a kind of auxiliary information for XML serialization tools, and has no other function. On the Nop platform, the XDef metamodel definition does more than just define the domain model structure itself ** Provides a coordinate system for locating domain concepts in the domain model space. **!

In the Nop platform's domain model, each node corresponds to a unique path from the root node (that is, its unique location coordinates), such as `/view/grids[@id="list"]/cols/col[@id="fieldA"]/label` the label attribute of the column with ID fieldA of the table with ID list.

> The XPath syntax can also be used for positioning within a Tree structure, but an XPath may in principle match multiple nodes, attributes, and therefore is not a one-to-one description and cannot be used as a positioning coordinate.

In the definition of XDef, for each set element, we usually configure an additional xdef: key-attr attribute to indicate the unique identification of its child nodes. For example, in the above example, the XDef corresponding to the grids set element of view is defined as

```xml
    <grids xdef:key-attr="id" xdef:body-type="list">
        <grid id="!xml-name" xdef:ref="grid.xdef"/>
    </grids>
```

This is actually the same as the key attribute setting required by the virtual DOM Diff algorithm of the front-end framework Vue/React.

Based on the xdef: key-attr setting, if we want to add an attribute to an existing table column, we can use the following

```xml
<view x:extends="_NopAuthUser.view.xml">
    <grids>
      <grid id="list">
        <cols>
           <!-- 删除已有的列 -->
           <col id="fieldB" x:override="remove" />
           <col id="fieldA" width="增加新的配置">
           </col>
        </cols>
      </grid>
    </grids>
</view>
```

The ** general class inheritance mechanism cannot override the attributes of a specific element in a list in a base class! **

Based on the difference calculation of the domain model, many functions related to the architecture abstraction can be implemented by the platform without being built in a specific domain model. For example, the NopIoC dependency injection container uses a configuration syntax similar to Spring 1.0, and it can use the unified Delta customization mechanism to remove the bean definitions built into the system without building any bean exclusion handling code into the engine. Therefore, NopIoC can achieve dynamic configuration capabilities beyond SpringBoot in about 4000 lines of code.

```xml
<beans x:schema="/nop/schema/beans.xdef" xmlns:x="/nop/schema/xdsl.xdef"
       x:extends="super" x:dump="true">
    <bean id="nopDataSource" x:override="remove" />

    <bean id="nopHikariConfig" x:override="remove" />

    <alias name="dynamicDataSource" alias="nopDataSource" />
</beans>
```

The above example was customized [ dao-defaults.beans.xml ](https://gitee.com/canonical-entropy/nop-for-ruoyi/blob/master/ruoyi-admin/src/main/resources/_vfs/_delta/default/nop/dao/beans/dao-defaults.beans.xml) when the Nop platform was integrated with the Ruoyi framework based on SpringBoot. It removes the default data source definition provided by the Nop platform and sets an alias for the dynamicDataSource built into the Ruoyi framework, allowing the Nop platform to use the data source directly.

### 4. Meta programming

All model in Skyve are either hand-written or fixed at that time of the first code generation. If we find some structural patterns that often appear, it is difficult to abstract them. That is, ** Skyve does not provide a mechanism for further secondary abstraction based on the built-in model. **.

The theory of reversible computation States that software construction can follow the formula:

```
  App = Delta x-extends Generator<DSL>
```

Generator is a key part of reversible computation theory. The domain model of the Nop platform has built-in X: gen-extends and X: post-extends metaprogramming mechanisms to complete dynamic code generation during model parsing and loading. With this mechanism, a large number of common structural transformations can be stripped from the runtime engine and pushed forward to the compile-time execution, which can greatly simplify the design of the runtime engine and improve the overall performance of the system.

Taking the workflow as an example, we generally need to add some special processing logic in the engine when implementing the countersign function, but at the conceptual level, the countersign step is actually a redundant design: it can be disassembled into a common step + an implicit aggregation step. Therefore, in the design of NopWorkflow, To support the countersign function, you only need to add a `<wf:CounterSignSupport/>` call in the X: post-extends section, which is responsible for identifying the countersign step and automatically expanding it into two step nodes according to the attribute settings on the countersign step.

This metaprogramming mechanism is very powerful because it is similar to mathematical theorem derivation: you only need to consider how to change symbols to get the final result, and you don't need to consider complex runtime state dependencies at all.

In the NopORM engine, JSON object support and extension field support are also implemented through compile-time run-time technology, and the ORM engine itself has no built-in knowledge of this. See for [ orm-gen.xlib ](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-orm/src/main/resources/_vfs/nop/orm/xlib/orm-gen.xlib) details

### 5. Custom extension

The attributes of model objects in Skyve are fixed. We can only accept the design of Skyve unilaterally. We can't add custom extended attributes to model objects without modifying the core code of Skyve. The design philosophy of the Nop platform is that delta is ubiquitous, and the following pairing structure (base, delta) needs to be used in any design, so space must be reserved for extended attributes in the design of model objects. The general convention in the NOP platform is that, in addition to the attributes defined in the XDef metamodel, the attributes with namespaces are by default extension attributes. For example

```xml
<prop name="fieldA" ext:show="C">...</prop>
```

We do not define the ext: show attribute in the XDef metamodel, but because it has a namespace, it is saved directly to the model object as an extended attribute when parsed, and no validation failure exception is thrown.

> The (base, delta) pairing design is reflected in all aspects of the Nop platform. For example, all message structures passed in the Nop platform are (data, headers) pairings. In fact, in many cases, meta data can be regarded as some kind of delta supplementary information to data, and data and meta data can be transformed into each other in different usage scenarios. If the current processing logic does not need to involve certain information, they can be stored and transmitted as meta data, and when they need to be processed in the next stage, the original partial data can be converted into meta data, and the original partial meta data will be converted into data for processing. It is not entirely accurate to say that meta data is data that describes data. In practice, meta data can contain additional information that is not relevant to the current application but is also harmless (** Useless and harmless **).

### 6. Domain Language Workbench

Skyve's approach is more traditional in that it implements specific functionality for each model separately. The Nop platform tries to provide a Language Workbench to provide a series of technical support for the development of domain-specific languages, so that we can quickly develop a corresponding domain-specific language according to the needs of the domain. See [ XDSL: Generic Domain Specific Language Design ](https://zhuanlan.zhihu.com/p/612512300). Domain language workbench can be regarded as a Language Oriented Programming paradigm. JetBrains, the developer of IDEA, has released a product [MPS](https://www.jetbrains.com/mps/) specifically designed to implement LOP. The design goal of Nop platform is roughly the same as that of MPS, but it is based on the systematic reversible computing theory, and there are essential differences between Nop platform and MPS in the basic software construction principles and technical routes.

In the Nop platform, all domain models are defined using a unified metamodel mechanism, which conforms to the underlying XDSL syntax specification (defined by the metamodel [xdsl.xdef](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-xdefs/src/main/resources/_vfs/nop/schema/xdsl.xdef)). With the generic capabilities provided by XDSL, our self-defined DSLs automatically gain the capabilities of delta merging, metaprogramming, breakpoint debugging, visual design, and more. ** For example, for the workflow engine, we only need to write the most kernel process runtime, and we can get the visual process designer, process breakpoint debugging, differential customization, inheritance of existing process templates and other capabilities without additional work. **。

Based on XDSL, we also naturally achieve seamless embedding between multiple DSLs. For example, the rule engine is embedded in the process engine, and the process steps are triggered in the actions of the rule engine.

## Three. Specific model comparison

In addition to the above differences in general mechanisms, the Nop platform is also more refined than Skyve in the implementation of specific domain models, and is more abstract and easier to extend.

### 1. Data model

The Document model in Skyve describes the structure of object attributes and the relationships between objects. They are not only responsible for describing the interface structure between the front end and the back end, but also responsible for describing the persistent data structure of the data storage layer. In the Nop platform, we use XMeta model and ORM model to complete similar functions.

The bottom layer of Skyve is based on Hibernate framework technology, so it inherits the related shortcomings of Hibernate while gaining the powerful capabilities of Hibernate. NopOrm engine is a new generation of ORM engine based on the principle of reversible computation. Through theoretical analysis, it defines EQL as a minimal object-oriented extension of SQL syntax: EQL = SQL + AutoJoin, which overcomes some inherent defects of Hibernate in theory. At the same time, the native ability of SQL is retained to the maximum extent. The specific design can be found in [ What ORM engine is needed for low-code platforms (1) ](https://zhuanlan.zhihu.com/p/543252423).

Skyve does not distinguish between the structural model of the interface layer and the structural model of the storage layer. In fact, it is difficult to isolate the impact of different levels of demand for complex business scenarios, and it is also difficult to adapt to long-term structural evolution. In the storage layer, we want the data structure to reduce redundancy, while in the interface layer, we may need to return multiple derived data for the same data.

Nop platform ** GraphQL services can be automatically generated based on the data model **, which has a series of built-in functions common to business:

* Composite primary key support
* Field automatic encryption and decryption support
* Field generation mask such as card number
* Automatically generate the corresponding Label field for the field according to the dictionary table configuration (generated by using the metaprogramming mechanism in the XMeta configuration)
* Bulk loading optimization (solving Hibernate's common N + 1 problem)
* Logical deletion
* Optimistic lock
* Automatically record the modifier and modification time
* Automatically record the field values of the entity before and after modification
* Built-in MakerChecker approval mechanism. After being enabled, the modification operation needs to be approved before being submitted.
* Primary-sub table is submitted at one time
* Delete subtable data recursively
* Extended field support
* Sub-warehouse and sub-table
* Distributed transaction

The specific design can be found in [ What ORM engine is needed for low-code platforms (2) ](https://zhuanlan.zhihu.com/p/545063021)

### 2. Background service extension

Skyve implements backend logic extensions through Bizlets.

```java
class Bizlet{
      public void preSave(T bean) throws Exception {
    }

    public void preDelete(T bean) throws Exception {
    }

    public void postRender(T bean, WebContext webContext) {
    }
}
```

This design is obviously bound up with the logic of adding, deleting, changing and checking. And its design is not complete, we can not intercept the query operation in a simple way, add additional behavior before and after the query. The query is executed by directly calling the storage layer interface and is not processed by the Bizlet.

The NopGraphQL engine of the Nop platform is decomposed to the object level and corresponds to the BizModel model. It is a general service model and is not limited to the implementation of CRUD services. CrudBiz Model is simply a base class that provides default action definitions. With the help of metadata information contained in XMeta, CrudBizModel can automatically realize very complex parameter verification and functions such as saving and copying primary-sub table structures. NopGraphQL engine has a very flexible data permission filtering function built in, which can precisely control data access permissions on complex object graphs through simple description configuration. See video [ Nop platform how to configure list filtering conditions and how to add data permissions ](https://www.bilibili.com/video/BV1Ac411H7my/) for details

Another design point of concern is the emphasis ** Frame independence of business logic expression ** on the Nop platform. Traditional service implementation is dependent on a specific framework. For example, the background service of Skyve uses the WebContext object, which directly contains the HttpServletRequest object and the HttpServletResponse object, which makes it bound to the Web runtime environment. We write business code that is difficult to migrate to a non-Web environment. On the Nop platform, the GraphQL engine's entry parameters and return objects are POJO objects without any specific runtime environment dependencies. NopGraphQL can be regarded as a purely logical running engine, and its input can come from various channels, for example, it can read the request object from the batch file, and automatically convert the online service into a batch service (based on the NopOrm engine, it will automatically implement batch submission optimization). In addition, the Kafka message queue can be directly connected, and the GraphQL service can be directly converted into a message processing service (the return message can be sent to a Reply Topic).

![](https://gitee.com/canonical-entropy/nop-entropy/raw/master/docs/arch/BizModel.svg)

The design based on POJO also greatly reduces the difficulty of unit testing, and a single service function can be tested without integration with the server.

The specific design of Nop GraphQL can be found in [ GraphQL engine in a low-code platform ](https://zhuanlan.zhihu.com/p/589565334)

### 3. Display the model

Skyve describes the main structure of the interface through the View model, which can be regarded as a front-end framework with only a few fixed components.

```xml
<view xmlns="http://www.skyve.org/xml/view"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    name="_residentInfo" title="Resident Info"
    xsi:schemaLocation="http://www.skyve.org/xml/view ../../../../schemas/view.xsd">
    <form border="true" borderTitle="Resident Info">
        <column percentageWidth="30" responsiveWidth="4" />
        <column />
        <row>
            <item>
                <default binding="parent.residentName" />
            </item>
        </row>
    </form>
    <form border="true" borderTitle="Resident Photo">
        <column percentageWidth="30" responsiveWidth="4" />
        <column />
        <row>
            <item showLabel="false">
                <contentImage binding="parent.photo" />
            </item>
        </row>
    </form>
</view>
```

The positioning of the XView model in the Nop platform is similar to that of Skyve's View model, but it uses a more business-oriented abstraction to abstract the concepts of form, table, layout, action, page, etc. In particular, the NopLayout layout language can be used to isolate the form layout information from the specific control content information of the form. For example

```xml
<view>
    <forms>
        <form id="edit" size="lg">
            <layout>
                ========== intro[商品介绍] ================
                goodsSn[商品编号] name[商品名称]
                counterPrice[市场价格]
                isNew[是否新品首发] isHot[是否人气推荐]
                isOnSale[是否上架]
                picUrl[商品页面商品图片]
                gallery[商品宣传图片列表，采用JSON数组格式]
                unit[商品单位，例如件、盒]
                keywords[商品关键字，采用逗号间隔]
                categoryId[商品所属类目ID] brandId[Brandid]
                brief[商品简介]
                detail[商品详细介绍，是富文本格式]

                =========specs[商品规格]=======
                !specifications

                =========goodsProducts[商品库存]=======
                !products

                =========attrs[商品参数]========
                !attributes

            </layout>
            <cells>
                <cell id="specifications">
                    <gen-control>
                        <input-table addable="@:true" editable="@:true"
                                     removable="@:true" needConfirm="@:false">
                            <columns j:list="true">
                                <input-text name="specification" label="规格名" required="true"/>
                                <input-text name="value" label="规格值" required="true">
                                </input-text>
                                <input-text name="picUrl" label="图片" required="true"/>
                            </columns>
                        </input-table>
                    </gen-control>
                    <selection>id,specification,value,picUrl</selection>
                </cell>
              </cells>
      </form>
  </forms>
</view>
```

The NopLayout layout language can express complex interface layout rules in a very compact way. The display control of a single field will be automatically inferred according to the data type and data domain information defined in the data model, without expression. If the automatically assumed control does not meet the requirements, we can use the gen-control configuration of the cell to specify the presentation control for the field separately.

> Interestingly, NopLayout is a layout syntax that ChatGPT can easily understand and imitate. See [ How to overcome the input token limitation of GPT and generate complex DSL ](https://zhuanlan.zhihu.com/p/615685144)

For specific NopLayout syntax rules, see [ Form layout language in low-code platforms: NopLayout ](https://zhuanlan.zhihu.com/p/592131885)

Skyve's View model design also has a problem: what if the default interface model doesn't meet the requirements? Skyve's current answer is that there is nothing we can do. If we go beyond the original design of the model, we can only abandon the whole page and use other technologies to write it from scratch. In the Nop platform, we can implement partial inheritance by using the differential merging mechanism, and then supplement partial differential description information.

The front end of the NOP platform uses the Baidu AMIS framework, which is a very good and powerful front-end low-code framework. For an introduction to it, see [ Why is Baidu AMIS framework an excellent design? ](https://zhuanlan.zhihu.com/p/599773955). The page description used by our front end is the JSON description generated according to the XView model during compilation. On the basis of automatically generating JSON, we can customize a small amount of difference. Therefore, as long as the page is within the capability of AMIS, it can reuse the capability of the XView model through partial inheritance, instead of writing from scratch.

```yaml
# main.page.yaml页面文件缺省根据XView模型生成

x:gen-extends: |
  <web:GenPage view="NopAuthUser.view.xml" page="main" 
        xpl:lib="/nop/web/xlib/web.xlib" />
```

An interesting question is, what if AMIS is not powerful enough to describe the front-end page structure? First, the capabilities of AMIS can be complemented by custom components, because all front-end control structures can eventually be expressed as some abstract syntax tree (AST), which in turn can be serialized into some JSON structure, so the JSON form of AMIS is in principle complete. There are no situations that cannot be described (the most extreme case is that the entire page is displayed with a custom component that reads the body configuration and interprets it as specific interface control content). In addition, we can also use the code generation and metaprogramming mechanism to provide a set of interpreters for the XView model, and translate the XView model into Vue source code or React source code.

### 4. Code generation

Skyve provides a Maven plugin that automatically generates entity class code based on XML model configuration. Skyve's code generator is relatively simple to implement, which is to output through text splicing in Java code. XCodeGenerator, the code generator for the Nop platform, is a more systematic solution.

First of all, XCode Generator supports incremental code generation. The generated code and the manually modified enhanced code are stored separately and do not affect each other. They can be regenerated at any time according to the model without overwriting the manually modified part.

Second, XCode Generator is a data-driven code generator, which can control the judgment and loop in the code generation process through the template directory structure. For example { enabled } { entity Model. Name }.java indicates that a corresponding Java file will be generated for each entity only if the enabled property is set to true.

Third, XCodeGenerator, like other models of the Nop platform, supports Delta customization. That is, we can overwrite the built-in template file by adding the corresponding file in the delta directory without modifying the built-in generation template of the platform.

Fourth, the XCode Generator supports generation for custom models and can be used independently outside of the Nop platform. For example, in addition to the built-in data model and API model, we can define a domain model in Excel format for our business domain, and then add a imp. XML description file to automatically parse the Excel file as a domain model object, and then apply the user-defined code generation template to generate the target file.

For a detailed introduction to the XCode Generator, see [ Data-driven differentially quantized code generator ](https://zhuanlan.zhihu.com/p/540022264)

### 5. Reporting tool

Skyve uses JasperReport to generate reports, which is definitely not enough for complex Chinese-style reporting requirements. Nop platform provides a Chinese-style report engine NopReport with Excel as the visual designer, which implements the unique hierarchical coordinate expansion mechanism of Chinese-style report theory with about 3000 lines of code. For details, see [ NopReport: An Open Source Chinese Report Engine Using Excel as a Designer ](https://zhuanlan.zhihu.com/p/620250740). [演示视频](https://www.bilibili.com/video/BV1Sa4y1K7tD/)

For Word template export, the Nop platform also provides a Word template engine that uses Word as the visual designer. It takes advantage of the XPL template language built into the Nop platform to convert Word files into dynamically generated template files with only about 800 lines of code. For details, please refer to [ How to implement a poi-tl-like visual Word template with 800 lines of code ](https://zhuanlan.zhihu.com/p/537439335)

### 6. Automated testing

Skyve provides an interesting automated test support, which can automatically generate the corresponding WebDriver automated test cases based on the data model and View model.

```xml
<automation uxui="external" userAgentType="tablet" testStrategy="Assert" 
    xsi:schemaLocation="http://www.skyve.org/xml/sail ../../../skyve/schemas/sail.xsd" 
    xmlns="http://www.skyve.org/xml/sail" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <interaction name="Menu Document Numbers">
        <method>
            <navigateList document="DocumentNumber" module="admin"/>
            <listGridNew document="DocumentNumber" module="admin"/>
            <testDataEnter/>
            <save/>
            <testDataEnter/>
            <save/>
            <delete/>
        </method>
    </interaction>
```

Skyve has knowledge of the model, so the test cases it generates can automatically take advantage of the information already expressed in the model, such as testDataEnter, which means that a corresponding test value is randomly generated for each field present in the form.

Nop platform uses model information more deeply. In addition to applying model information at the input, we can make full use of the advantages of model-driven, capture all side effects in the system, and record them, so as to transform test cases that depend on complex States into pure logic test cases without side effects. See for [ Automated Testing in Low-Code Platforms ](https://zhuanlan.zhihu.com/p/569315603) details

## Four. The value of theory

The essential difference between Nop platform and all other low-code platforms is that it is based on a new software construction theory, reversible computing theory, which first establishes a minimum set of concepts, and then uses rigorous logical derivation to gradually build a huge technical system. Reversible computing theory overcomes the limitations of component theory at the theoretical level, and removes the theoretical obstacles for system-level, coarse-grained software reuse.Nop platform, as a reference implementation of reversible computing theory, provides a unified technical tool to solve the common problems in the modeling process of many fields.

The Nop platform solves problems in a way that is significantly different from other platforms. Taking Excel model parsing as an example, the general practice is to specify a model format for a specific business requirement, and then write a corresponding Excel parsing function. For different model files, we will write a number of different parsing functions. In the Nop platform, we specify a rule to map the Excel structure to the domain object structure, and then compile a unified Generic parser. If we use the terminology of category theory, we can say that the Excel model parser in Nop platform is a Functor mapping from the Excel category (containing infinitely many different Excel file formats) to the domain object category (containing infinitely many different domain object structures). If we define a reverse report derivation functor from the category of DSL model objects to the category of Excel, they can actually form a pair of Adjoint functors.

A so-called Functor is a "structure-preserving" mapping that defines every object in Domain A to an object in another Domain B. The main way to solve the problem of category theory is through the concept of functor. Specifically, if we want to solve a problem, we first expand it into a functor mapping problem, and solve a problem set containing all related problems at one time, so as to indirectly achieve the purpose of solving a specific problem. This solution to magnifying the problem is undoubtedly crazy. If it succeeds, the only possibility is that the field it applies to has a stable and reliable scientific law that can be clearly defined at the mathematical level, it is science.

Some people may not be interested in reversible theory, and feel that the theory is only a kind of rhetoric when the academic circles publish articles, which is out of touch with the practice of software engineering. But Vapnik, the father of statistical learning, famously said, nothing is more practical than a good theory. The theory of reversible computation is equivalent to expanding the solution space when we solve problems, revealing many unprecedented technological possibilities. Based on the guidance of reversible computing theory, Nop platform captures the unified construction law in the software structure space at a very small technical cost (the current amount of code is hundreds of thousands of lines), defines a feasible technical route to intelligent low-code development, and can clearly see where we are going and where we are at present. In the next few years, we will certainly see such terms as delta, Delta, reversible and generative frequently appear in various technical fields, and their comprehensive application will inevitably lead to reversible computing theory.

> An interesting thing is that the computational model of deep learning theory corresponds to `Y = Sigma( W*X + B) + Delta`. Considering the residual connection, the formula of deep learning is the same as the construction formula of reversible computing theory. When solving problems by reversible computing, it will inevitably involve the problem of deep nesting of multiple models, just like the multi-layer neural network in deep learning.

After setting up the Nop platform communication group, I often get feedback in the communication: Ah, it can still be like this. This is normal, one cannot understand what one does not yet understand, and a practical example of the development of the Nop platform may help us better understand the theory of reversible computation. Only when you need to customize the existing functions and mechanisms in the platform (or the basic products written by yourself) for specific business needs, can you realize the great difference between the Nop platform and all other open technologies.

Open source address of Nop platform:

- gitee: [ canonical-entropy/nop-entropy ](https://gitee.com/canonical-entropy/nop-entropy)
- github: [ entropy-cloud/nop-entropy ](https://github.com/entropy-cloud/nop-entropy)
- Development example: [ docs/tutorial/tutorial.md ](https://gitee.com/canonical-entropy/nop-entropy/blob/master/docs/tutorial/tutorial.md)
- [Introduction and Q & A of Reversible Computing Principle and Nop Platform _ bilibili _ bilibili] (https://www.bilibili.com/video/BV1u84y1w7kX/)