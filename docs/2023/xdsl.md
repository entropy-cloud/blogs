# XDSL: Generic Domain Specific Language Design

The Nop platform provides a language-oriented programming paradigm, that is, when we solve problems, we always tend to design a domain specific language (DSL) first, and then use the DSL to describe business logic concretely. Creating a custom DSL is greatly simplified in the Nop platform.

## One. Use XML or JSON syntax

The value of DSL is that it refines the domain-specific logical relations and defines the atomic semantic concepts specific to the domain. As for the specific grammatical form, it is not the key. After the program code is parsed by Lexer and Parser, the abstract syntax tree (AST) is obtained, and all the program semantics are carried by AST in principle. Both XML and JSON are tree structures that express the AST directly, thereby avoiding the need to write special Lexers and Parsers altogether.

> The Lisp language's approach is to express the AST directly using the generic S-Expr, making it easy to define custom DSLs using the macro mechanism. A similar effect can be achieved based on XML syntax. In particular, XML tags can represent template functions, dynamically generate new XML nodes, and play a role similar to Lisp macros (the structure of code and code generation results is XML nodes, which corresponds to [ What is called homography in Lisp ](https://zhuanlan.zhihu.com/p/34063805)).

We use the XDef metamodel definition language to constrain the syntactic structure of the DSL, for example [beans.xdef](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-xdefs/src/main/resources/_vfs/nop/schema/beans.xdef). Compared to XML Schema or JSON Schema, XDef definitions are simpler and more intuitive, and can express more complex constraints. For details on the XDef language, refer to [xdef.md](xdef.md)

>  All DSLs in the Nop platform are defined by XDef language, including workflow, report, IoC, ORM, etc. The definition files are stored in [ Nop-xdefs module ](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-xdefs/src/main/resources/_vfs/nop/schema).

![](xml-to-json.png)

XDef doesn't just define the syntax of a DSL in XML format, it also specifies a kind of bidirectional conversion rule between XML and JSON. Therefore, as long as the XDef metamodel is defined, the JSON representation can be automatically obtained, which can be directly used for the input and output of the foreground visual editor.

Without defining the XDef metamodel, the Nop platform also defines a compact convention transformation rule, which can realize the bidirectional transformation of XML and JSON without Schema constraints. See the XML representation of the front-end AMIS page for details: [amis.md](../xui/amis.md)

## Two. XDSL Common Syntax

After all DSLs are normalized to XML format, advanced mechanisms such as module decomposition, delta merging, and metaprogramming can be provided uniformly. Nop platform defines a unified XDSL extension syntax, which automatically adds reversible computation extension syntax for all DSL languages defined by XDef metamodel. The specific content of XDSL syntax is defined by [xdsl.xdef](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-xdefs/src/main/resources/_vfs/nop/schema/beans.xdef) this metamodel.

Examples of the main syntax elements of XDSL are as follows:


```xml
<orm x:schema="/nop/schema/orm/orm.xdef"
     x:extends="base.orm.xml" x:dump="true"
     xmlns:x="/nop/schema/xdsl.xdef" xmlns:xpl="/nop/schema/xpl.xdef">
    <x:gen-extends>
        <pdman:GenOrm src="test.pdma.json" xpl:lib="/nop/orm/xlib/pdman.xlib"
                      versionCol="REVISION"
                      createrCol="CREATED_BY" createTimeCol="CREATED_TIME"
                      updaterCol="UPDATED_BY" updateTimeCol="UPDATED_TIME"
                      tenantCol="TENANT_ID"
        />
    </x:gen-extends>

    <x:post-extends>
        <orm-gen:JsonComponentSupport xpl:lib="/nop/orm/xlib/orm-gen.xlib"/>
    </x:post-extends>

    <entities>
        <entity name="io.nop.app.SimsClassFee" x:override="remove"/>
    </entities>
</orm>  
```

1. All XDSL files require that the root node must use an `x:schema` attribute to specify the xdef definition file to be used.
2. The root node can be set `x:dump="true"` to print the intermediate results of the delta consolidation process and the final consolidated results. In the debugging mode of the Quarkus framework, the final merging result will be output to the _ dump directory of the current project.
3. The `x:extends` attribute is introduced into the inherited base model, and the current model and the base model are merged level by level according to the tree structure.
4.  `x:gen-extends` And `x:post-extends` provide built-in metaprogramming mechanisms for dynamically generating model objects that can then be merged with the current model.
5.  `x:override` Attributes allow you to control the details of merging two nodes, such as `x:override="remove"` deleting the corresponding node in the base model, or `x:override="replace"` completely covering the corresponding node in the base model by the current node. By default `x:override="merge"`, it indicates that the child nodes are merged step by step. Please refer to the document [ x-override.md ](x-override.md) for the detailed introduction of the consolidation rule

### Merge order of x-extend

The x-extends delta merging mechanism implements the technical pattern required by the reversible computation theory.

> App = Delta x-extends Generator<DSL>

Specifically, `x:gen-extends` and `x:post-extends` are Generators executed during compilation. They use the XPL template language to dynamically generate model nodes, allowing multiple nodes to be generated at one time and then merged in turn. The specific merging sequence is defined as follows:


```
<model x:extends="A,B">
    <x:gen-extends>
        <C/>
        <D/>
    </x:gen-extends>

    <x:post-extends>
        <E/>
        <F/>
    </x:post-extends>
</model>
```

The result of the merge is


```
F x-extends E x-extends model x-extends D x-extends C x-extends B x-extends A 
```

The current model overwrites `x:gen-extends` the results of and `x:extends`, and `x:post-extends` overwrites the current model.

With the help of X: extends and X: gen-extends, we can effectively implement the decomposition and composition of DSL.

### Significance of X: post-extends

If we have created an XDSL domain specific language and now want to introduce additional extensions for some special scenarios, but do not want to modify the underlying runtime engine, we can take advantage of `x:post-extends` the mechanism.

Based on the reversible computation theory, for the existing DSL, we can further decompose it by reversible computation to obtain a new DSLx.


```java
App = Delta x-extends Generator<DSL>
DSL = Delta x-extends Generator<DSLx>
```

We can use the DSLx extension syntax when describing the business, and then `x:post-extends` translate it into the existing DSL syntax. After the ** `x-extends` merge algorithm is executed, it will automatically delete all the attributes and child nodes ** of the X namespace, so the lowest level of parsing and running engine does not need to know the knowledge of these extended syntax at all, they only need to write ** All generic extension mechanisms are implemented at the XDSL syntax level by compile-time metaprogramming. ** for the original DSL semantic concepts.

Take a specific example. In the ORM engine, for a JSON text field, we want it to correspond to two entity properties, one is jsonText corresponding to the JSON text storage, and the other is JSON Compo nent corresponding to parsing the JSON text into an object structure. Modifying the object properties will eventually cause the jsonText storage text to be modified. We want to mark a field as JSON text by adding a JSON tag to it, and then automatically generate the corresponding component attribute for the field. This is a special convention that we don't want to build into the ORM engine, and we can use `x:post-extends` the mechanism to implement this abstraction.


```xml
<orm x:schema="/nop/schema/orm/orm.xdef"
     x:extends="base.orm.xml">
    <x:post-extends>
        <orm-gen:JsonComponentSupport xpl:lib="/nop/orm/xlib/orm-gen.xlib"/>
    </x:post-extends>
    <entities>
      <entity name="xxx.MyEntity">
            <columns>
                <column name="jsonExt" code="JSON_EXT" propId="101" tagSet="json"                           stdSqlType="VARCHAR"
                        precision="4000"/>
            </columns>
        <!-- 最终会自动生成component配置
           <components>
              <component name="jsonExtComponent"
                         class="io.nop.orm.support.JsonOrmComponent">
                 <prop name="jsonText" column="jsonExt" />
              </component>
           </components>
         -->
      </entity>
    </entities>
</orm>  
```

If we customize a lot of extensions, we can further encapsulate them into a base model, such as


```xml
<!-- std.orm.xml -->
<orm x:schema="/nop/schema/orm/orm.xdef"
     x:extends="base.orm.xml">

    <x:post-extends>
        <orm-gen:JsonComponentSupport xpl:lib="/nop/orm/xlib/orm-gen.xlib"/>
    </x:post-extends>
</orm>
<!-- my.orm.xml -->
<orm x:extends="std.orm.xml">
    ....
</orm>
```

Commonly used extensions can be encapsulated into a STD. POM. XML model, and then the corresponding extension support can be obtained only by inheriting the model.

> Multiple model paths separated by commas are `x:extends` supported, and multiple base models can be inherited at one time. The models are merged in order from the front to the back.

Furthermore, ** `x:post-extends` it paves the way ** for the implementation of customized visual designers. When the x-extends merging algorithm is executed, the merging phase can be specified. If it is only merged to the mergeBase phase, we will get the current model and `x:gen-extends` the merged result, but it has not been applied `x:post-extends` at this time. The visual designer can be tailored to the mergeBase artifacts, providing a wide range of business-specific configuration options without any changes to the underlying runtime engine.

In the Nop platform, the common countersign node of OA approval is implemented by the `x:post-extends` mechanism. The underlying workflow engine is designed for common scenarios. Because the function of countersigning can be implemented through a common step node + a Join merge node, there is no need to build in countersigning-related knowledge in the underlying engine. In the workflow designer, we provide countersign nodes and a large number of OA-related simplified operations, and then in the meta-programming phase `x:post-extends`, the mechanism is responsible for expanding these OA-related configurations into model nodes and attributes that can be recognized by the underlying engine.

### Executable semantics
Executable semantics are implemented in XDSL through the XLang language. As long as an attribute is marked as an EL expression in the xdef metamodel, or the content of a node is an XPL template language, the attribute will be automatically parsed as an IEvalAction executable function interface. Specific examples can be found in [wf.xdef](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-xdefs/src/main/resources/_vfs/nop/schema/wf/wf.xdef)

```xml
 <action name="!string" ...>
    <when xdef:value="xpl-predicate"/>

    <arg name="!var-name" xdef:ref="WfArgVarModel" xdef:unique-attr="name"/>

    <source xdef:value="xpl"/>
 </action>   
```

Nop platform provides the XLang language with document hints, autocompletion, syntax checking, breakpoint debugging and other functions through the nop-idea-plugin plug-in. See for [ idea-plugin.md ](https://gitee.com/canonical-entropy/nop-entropy/blob/master/docs/user-guide/idea/idea-plugin.md) details

![](https://gitee.com/canonical-entropy/nop-entropy/raw/master/docs/user-guide/idea/idea-executor.png)

![](https://gitee.com/canonical-entropy/nop-entropy/blob/master/docs/tutorial/xlang-debugger.png)

### Delta Customization Beyond Interfaces and Components

Based on the theory of reversible computation, XDSL of Nop platform has a built-in general Delta customization mechanism, which is simpler and more flexible than the traditional interface abstraction and component assembly.

All the XDSL model files are stored in the `src/resources/_vfs` directory, and they form a virtual file system. This virtual file system supports the concept of Delta layered overlay (similar to the overlay-fs layered file system in Docker technology), which has layers `/_delta/default` by default (more layers can be added through configuration). That is, if there is both a file `/_vfs/_delta/default/nop/app.orm.xml` and `/nop/app.orm.xml` a file, the version in the delta directory is actually used. In the delta customization file, you can use to `x:extends="raw:/nop/app.orm.xml"` inherit the specified base model, or use to `x:extends="super"` represent to inherit the base model of the previous level.

Delta customization is very flexible, and the granularity can be coarse or fine. Coarse enough to customize the entire model file. So detailed that you can customize individual attributes or nodes. And unlike interface customization, Delta customization can implement ** Delete ** functions, that is, to mark the deletion of a certain part of the model in the customization file, and it is a real deletion, not a null operation to simulate, which will not affect the runtime performance.

In contrast to the customization mechanisms provided by traditional programming languages, ** The rules of Delta customization are very general and intuitive, and are independent of the specific application implementation. **. Taking the customization of the database Dialect used by the ORM engine as an example, if we want to extend the built-in MySQLDialect of the Hibernate framework, we must have some knowledge of the Hibernate framework. Then we also need to know how Spring encapsulates Hibernate, and where to find Dialect and configure it to the current SessionFactory. In the Nop platform, we only need to add files `/_vfs/default/nop/dao/dialect/mysql.dialect.xml` to ensure that all places using the MySQL dialect are updated to use the new Dialect model.

Delta custom code is stored in a separate directory, which can be separated from the code of the main application. For example, the delta customization file is packaged into the nop-platform-delta module, and when this customization is needed, the corresponding module is imported. We can also introduce multiple delta directories at the same time and then control the order of the delta layers through the NOP. Core. VFS. Delta -layer-ids parameter. For example, the configuration NOP. Core. VFS. Delta -layer-ids = base, Hunan enables two delta layers, one for the base product and one above it for a specific deployment version. In this way, we can productize software at a very low cost: ** When a basic product with basically complete functions is implemented at various customers, the code of the basic product can not be modified at all, but only the Delta customization code can be added. **.

### Three. Antlr extension

The NOP platform also provides certain support for the DSL development of custom program syntax. The AST parser can be directly generated based on the G4 file definition of Antlr4 (Antlr only supports parsing to ParseTree, and you need to manually write the conversion code from ParseTree to AST). See for [antlr.md](antlr.md) details
