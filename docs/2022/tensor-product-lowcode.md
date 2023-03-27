# Design of LowCode Platform from a tensor product perspective 

## Contact me

* Blog -> <https://www.zhihu.com/column/reversible-computation>
* Email -> <canonical_entropy@163.com>
* GitHub -> [entropy-cloud@GitHub](https://github.com/entropy-cloud)

---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>


A fundamental problem in software design is the problem of extensibility. A basic strategy for dealing with scalability is to view the new element of change as a new dimension, and then examine the interaction between this dimension and the existing dimensions.

For example, an OrderProcess processing logic has been written for the Order object. If it is released as SAAS software, the tenant dimension needs to be added. In the simplest case, the tenant only introduces the filtering field at the database level, that is, the tenant dimension is relatively independent, and its introduction does not affect the specific business processing logic (the logic related to the tenant is independent of the specific business processing process, and can be uniformly defined and solved in the storage layer).

However, the more complex scalability requirement is that each tenant can have its own customized business logic. At this time, the tenant dimension cannot maintain independence and must interact with other business and technical dimensions. In this paper, we will introduce a heuristic point of view, which analogizes the general scalability problems such as tenant extension to the extension process of tensor space through tensor product, and combines with reversible computing theory to provide a unified technical solution for such scalability problems.

## One. Linear Systems and Vector Spaces

The simplest class of systems in mathematics is the linear system, which satisfies the law of linear superposition

$$
f(\lambda_1 v_1 + \lambda_2 v_2) = \lambda_1 f(v_1) + \lambda_2 f(v_2)
$$

We know that any vector can be decomposed into a linear combination of basis vectors.

$$
\mathbf v = \sum_i \lambda_i \mathbf e_i
$$

Therefore, the structure of a linear function acting on a vector space is very simple in nature, and it is completely determined by the values of the function on the basis vectors.

$$
f(\mathbf v) = \sum_i \lambda_i f(\mathbf e_i)
$$

Once we know the value $f (\ mathbf e _ I) $of the function f on all basis vectors, we can directly calculate the value of the function f at any vector in the vector space spanned by $\ mathbf e _ I $.

According to the spirit of mathematics, if a mathematical property is very good, we define the mathematical object (** A mathematical property defines a mathematical object, not that a mathematical object has some mathematical property. **) that we need to study on the premise of this property. So in the field of software framework design, ** If we actively require a frame design to satisfy the linear superposition law, ** what should its design look like?

First, we need to re-examine the meaning of linear systems from a less mathematical point of view.

1. $f(\mathbf v)$ It can be regarded as performing some operation on a parameter object with complex structure.

2. $\mathbf v = \sum_i 易变的参数\cdot 标识性的参数$。 Some parameters are relatively fixed parameters with special identification functions, while other parameters are volatile parameters that change with each request.

3. ** f first acts on the identifying parameter (the result of this action can be determined in advance) to obtain a calculation result, and then combines ** this calculation result with other parameters.

Take a specific example, such as submitting a request in the foreground, which needs to trigger an operation on a group of objects in the background.

$$
request = \{ obj1：data1, obj2: data2, ... \}
$$

Arrange in vector form

$$
request = data1* \mathbf {obj1} + data2* \mathbf {obj2} + ...
$$

As we explore ** All possible requests **, we will find that all requests form a vector space. ** Each objName corresponds to a basis vector in the vector space. **.

The processing logic of the back-end framework corresponds to the

$$
\begin{aligned}
process(request) &= data1* route(\mathbf {obj1}) + data2* route(\mathbf {obj2}) + ...\\
&= route(\mathbf {obj1}).handle(data1) + route(\mathbf {obj2}).handle(data2) + ...
\end{aligned}
$$

The framework bypasses the mutable parameter data, first acts on the object name parameter, routes to a handler according to the object name, and then calls the handler to pass in the data parameter.

> What we need to $\lambda_i f(\mathbf e_i)$ note here is that the combination of the $\langle \lambda_i, f(\mathbf e_i)\rangle $ parameter and $f (\ mathbf e _ I) $is not necessarily a simple numerical multiplication, but can be extended to the result of some inner product operation, which is embodied in the software code level as a function call.

## Two. Tensor Product and Tensor Space

In mathematics, a basic problem is how to automatically generate larger and more complex mathematical structures from some smaller and simpler mathematical constructions, and the concept of Tensor Product is a natural result of this automatic way of construction (the so-called naturalness here has a precise mathematical definition in category theory).

First, let's look at a generalization of linear functions: multilinear functions.

$$
f(\lambda_1 u_1+\lambda_2 u_2,v) = \lambda_1 f(u_1,v) + \lambda_2 f(u_2,v) \\
f(u,\beta_1 v_1+ \beta_2 v_2) = \beta_1 f(u,v_1) + \beta_2 f(u,v_2)
$$

A linear function acting on a vector space can be seen as a one-parameter function that receives a vector and produces a value. Similar to the generalization of single-parameter function to multi-parameter function, multi-linear function has multiple parameters, each of which corresponds to a vector space (which can be regarded as an independent variable dimension). When a certain parameter is fixed (for example, fixed parameter u, fixed parameter V or fixed parameter V, fixed parameter u), it satisfies the linear superposition law. Like a linear function, the value of a multilinear function is determined by its values on the basis vectors

$$
f(\sum_i \lambda_i \mathbf u_i,\sum_j \beta_j \mathbf v_j)= 
\sum_{ij} \lambda_i \beta_j f(\mathbf u_i,\mathbf v_j)
$$

$f(\mathbf u_i,\mathbf v_j)$ Is actually equivalent to passing in a tuple, which is

$$
f(\mathbf u_i, \mathbf v_j)\cong f(tuple(\mathbf u_i,\mathbf v_j)) \cong f(\mathbf u_i\otimes  \mathbf v_j )
$$

That is, we can forget that f is a multi-parameter function and think of it as a one-parameter function that receives a complex parameter form $\mathbf u_i \otimes \mathbf v_j$. Returning to the original multilinear function $f(\mathbf u,\mathbf v)$, we can now view it in a new perspective as a ** Linear Functions on a New Vector Space **

$$
f(\mathbf u\otimes \mathbf v)=\sum _{ij} \lambda_i \beta_j f(\mathbf u_i \otimes \mathbf v_j)
$$

$$
\mathbf u \otimes \mathbf v = (\sum_i \mathbf \lambda_i \mathbf u_i) 
\otimes (\sum_j \beta _j \mathbf v_j)  
= \sum _{ij} \lambda_i \beta_j \mathbf u_i \otimes \mathbf v_j
$$

> $f(\mathbf u,\mathbf v)$ And $f (\ mathbf u \ otimes \ mathbf V) $are actually not the same function, but have some equivalence, and their symbols are written as f here.



$\mathbf u \otimes \mathbf v$ Called the tensor product of the vector $\ mathbf u $and the vector $\ mathbf V $, it can be seen as vectors in a new vector space, the so-called tensor space, whose basis is $\mathbf u_i \otimes \mathbf v_j$.

If $\ mathbf U \ in U $is an m-dimensional vector space and $\ mathbf V \ in V $is an n-dimensional vector space, Then the tensor space $U \ otimes V $contains all vectors of the form $\ sum _ i T _ { IJ } \ mathbf u _ I \ otimes \ mathbf V _ J $, It corresponds to a vector space of dimension $m \ times n $ (which is also known as the tensor product space of $U $and $V $).



> $U\otimes V$ Is the space spanned by all tensor products of the form $\ mathbf u \ otimes \ mathbf V $, where the tensor is linearly spanned, that is, the set of all linear combinations of these vectors. There are more elements in this space than there are vectors of the form $\ mathbf u \ otimes \ mathbf V $, i.e., not all vectors in tensor space can be written in the form $\ matbf u \ otimes \ matbf V $. For example
> 
> $$
> \begin{aligned}
>  \mathbf u_1 \otimes \mathbf v_1 + 4 \mathbf u_1 \otimes \mathbf v_2
>  + 3 \mathbf u_2 \otimes \mathbf v_1
>  + 6 \mathbf u_2 \otimes \mathbf v_2
>  &= (2\mathbf u_1 + 3 \mathbf u_2)\otimes (\mathbf v_1
>  + 2 \mathbf v_2) \\
>  &= 
>  \mathbf u \otimes \mathbf v
>  \end{aligned}
> $$
> 
> However, $2 \mathbf u_1 \otimes \mathbf v_1 + 3 \mathbf u_2 \otimes \mathbf v_2$ it cannot be decomposed into the form of $\ mathbf u \ otimes \ mathbf V $, but can only be kept in the form of linear combination.
> 
> Physically, this corresponds to the so-called quantum entangled state.



Tensor product is a Free strategy for constructing complex structures from simple structures. Free here (which has a strict mathematical meaning in category theory) means that the construction process does not add any new operation rules, but takes one from each set to form a pair and puts it there.



> In essence, $\ mathbf u \ otimes \ mathbf V $中$ \ mathbf u $and $\ mathbf V $do not have any direct interaction. The effect of $\ math BF V $on $\ math BF u $is shown only when the external function $f $is applied to $\ math BF u \ otimes \ math BF V $. That is to say $f(\mathbf u \otimes \mathbf v) \ne f(\mathbf u)$, we will find that the existence of $\ mathbf V $will affect the result of f acting on $\ mathbf u $.



With the help of the concept of tensor product, it can be considered that a multilinear function is equivalent to an ordinary linear function on a tensor space. Of course, this statement is very loose. A slightly stricter statement is:

For any (every) multilinear function $\phi: U\times V\times W ...\rightarrow X$, there is the ** There exists a unique ** tensor space $U \ otimes V \ otimes W.. A linear function on $such that

Or any product $U \ times V \ times W.. Acting on a vector space. Multilinear functions on $can be decomposed into a two-step mapping process, that is, first mapping to tensor products, and then applying linear functions on tensor spaces.



In the previous section, we introduced the concepts of linear systems and vector spaces, and pointed out that the software framework can simulate the action process of linear systems. Combined with the concept of tensor product introduced in this section, we can easily get a general scalability design scheme: from receiving vector parameters to receiving tensor parameters ** The increasing need for variability can be absorbed by the tensor product **. For example,

$$
process(request) = data * route(\mathbf {objName} \otimes \mathbf {tenantId})
$$

Adding the tenant concept may change the processing logic of all business objects in the system, but at the framework level, we only need to enhance the route function to allow it to receive the tensor product composed of objName and tenantId, and then dynamically load the corresponding processing function.



If we think about the processing logic here, we will find that if we implement the software framework as a linear system, its core is actually a tensor product as a parameter ** Loader function **.



In software systems, the concept of Loader function is ubiquitous, but its role has not been fully recognized. Looking back at the case of NodeJs, all library functions that are called are formally loaded through the require (path) function, that is, when we call the function f (a), we essentially execute require ( "f").call (null, a). If we enhance the require function to allow it to be dynamically loaded based on more identifying parameters, it is clear that we can achieve an extensible design at the function level. The hot update mechanism of HMR module used in Webpack and Vite can be understood as a Reactive Loader, which monitors the changes of dependent files, and then repackages, loads and replaces the function pointers currently in use.



The theory of reversible computation provides a new theoretical interpretation of Loader function ** It brings a unified and common technical implementation scheme. **. In the following pages, I will present an overview of the technical solution used in Nop Platform 2.0, an open source implementation of reversible computing, and its rationale.

## Three. Everything is Loader

> The programmer asks the function: Where do you come from and where do you want to go?
> 
> The function answers: Born in Loader, attributed to data

The Maxim of functional programming is that everything is a function, everything is function. However, considering the extensibility, this function can not be constantly changing. In different scenarios, we will eventually apply different functions. If the basic structure of the program is f (data), we can reconstruct it in a systematic way as

loader("f")(data)。 The design of many frameworks and plug-ins can be examined from this perspective.

* Ioc container:
  
  buildBeanContainer(beansFile).getBean(beanName, beanScope).methodA(data)
  
  $$
  Loader(beansFile\otimes beanName\otimes beanScope \otimes methodName)
  $$

* Plug-in system
  
  serviceLoader(extensionPoint).methodA(data)
  
  $$
  Loader(extensionPoint \otimes methodName)
  $$

* Workflow:
  
    getWorkflow(wfName).getStep(stepName).getAction(actionName).invoke(data)
  
  $$
  Loader(wfName\otimes stepName \otimes actionName)
  $$

When we identify similar Loader structures at all levels of the system, an interesting question is: how consistent are these Loaders? Can the code be reused between them? Workflow engine, IoC engine, report engine, ORM engine.., all of these engines need to load their own specific models. At present, most of them are fighting separately. Can one ** System-level, unified Loader ** of them be abstracted to load the model? If so, what specific common logic can be implemented in this unified Loader?



The design goal of a low-code platform is to model the logic of the code, and when the model is saved in serialized form, it forms a model file. The input and output of visual design are model files, so in fact, visualization is only a incidental benefit of modeling. The most basic work of a unified low-code platform should be to manage all models in a unified way and realize the resource utilization of all models. The Loader mechanism must be a core component in such a low-code platform.



Let's look at a common function in daily development.


```java
JsonUtils.readJsonObject(String classPath, Class beanClass)
```

This is a general Java configuration object loading function. It reads a JSON file under classpath and converts it into a Java object of a specified type through the JSON deserialization mechanism. Then we can use this object directly in programming. If the configuration file format is wrong, for example, the field name is written wrong, or the data format is wrong, it can be detected in the type conversion stage. If some validator annotations such as @ Max and @ NotEmpty are configured, we can even perform some business-related verification during deserialization. Obviously, the loading and parsing of all kinds of model files can be seen as a variant of this function. Taking the loading of workflow model as an example,


```java
workflowModel = workflowLoader.getWorkflow(wfName);
```

Loaders for workflow models typically have the following enhancements over the more primitive JSON parsing:

1. May be loaded from the database, not limited to a file under class path

2. The model file format may be in XML format, not limited to JSON format

3. Executable script code can be configured in the model file, not limited to a few primitive types of data items such as string/Boolean/number.

4. The format verification of the model file is more strict, such as checking that the attribute value is within the range of the enumeration item and that the attribute value meets the specific format requirements.

Nop Platform 2.0 is an open source implementation of the theory of reversible computation, which can be seen as a low-code platform that supports the development of domain-specific languages (DSLs). In the Nop platform, a unified model loader is defined.


```java
interface IResourceComponentManager{
    IComponentModel loadComponent(String componentPath);
}
```

1. The model type can be identified by the suffix name of the model file, so there is no need to pass in the type information of componentClass.

2. In the model file, the schema definition file that the model needs to meet is introduced through the X: schema = "XXX. Xdef", so as to implement format and semantic verification that is stricter than Java type constraints. ".

3. By adding field types such as expr, executable code blocks can be directly defined in the model file and automatically parsed into executable function objects.

4. Through the virtual file system, multiple storage modes of the model file are supported. For example, you can specify a path format that points to the model files stored in the database.

5. The loader automatically collects the dependencies in the model parsing process and automatically updates the model parsing cache based on the dependencies.

6. If equipped with a FileWatcher, it can actively push the updated model when the model dependency changes.

7. DeltaMerger and XDslExtender are used to realize the decomposition and assembly of the model. This is covered in more detail in Section 5 (and is a significant difference between the Nop platform and other platform technologies).

In the Nop platform, all model files are loaded through the unified model loader. ** All model objects are also automatically generated through the Meta Model definition. **. In this case, review the workflow model processing above


```java
getWorkflow(wfName).getStep(stepName).getAction(actionName).invoke(data)
```

GetWorkflow is implemented by a unified component model loader and does not need to be specially written. At the same time, getStep/getAction and other methods are automatically generated by the meta-model definition and do not need to be specially written. Therefore, the implementation of the whole Loader can be said to be completely automated.

$$
Loader(wfName\otimes stepName \otimes actionName)
$$



From another point of view, the parameter of Loader can be regarded as a multi-dimensional coordinate (** All the information available for unique positioning is coordinates. **): each wfName corresponds to a virtual file path path, and path is the coordinate parameter required for locating in the virtual file system. At the same time, stepName/actionName and so on are coordinate parameters required for unique positioning within the model file. Loader takes a coordinate and returns a value, so it can also be thought of as defining a coordinate system.



In a sense, reversible computation theory is to establish and maintain such a coordinate system, and to study the evolution and development of model objects in this coordinate system.


## Four. Loader as Multiple Dispatch

The function represents some kind of static computation (the code itself is deterministic), while the Loader provides a computation mechanism, and the result of its computation is the returned function, so the Loader is a higher-order function. If the Loader does not simply locate an existing code block according to the parameters, but can dynamically generate the corresponding function content according to the incoming parameters, then the Loader can be used as an entry point for the metaprogramming mechanism.



In programming language theory, there is a language built-in meta-programming mechanism called Multiple Dispatch, which has been widely used in Julia language. There are many similarities between multiple dispatch and the Loader mechanism defined here. In fact, Loader can be seen as an extension of multiple dispatch beyond the type system.



Consider a function call f (a, B). If it is implemented in an object-oriented language, we will choose to implement the first parameter a as an object of type A, while the function f is a member function defined on type A, and B is a parameter passed to the function f. The object-oriented call form a. F (B) is so-called single dispatch, that is, according to the type of the first parameter a (this pointer) of the function, the virtual function table of type a is dynamically queried to determine the specific function to be called. That is,


```text
在面向对象的观点下　f::A->(B->C)
```

> The a. F (B) corresponds at the implementation level to a function f (a, B), where a is the implicitly passed this pointer

The so-called multiple dispatch means that when a function is called, the "most suitable" implementation function is selected to be called according to ** All parameters ** the runtime type, that is,


```text
在多重派发的观点下　f:: A x B -> C， AxB为A和B构成的元组
```

> The Julia language can dynamically generate a specialized version of the code at compile time based on the type of parameters given when calling a function, thereby optimizing program performance. For example, f (int, int) and f (int, double) may generate two different binary code versions in the Julia language.

If we take the point of view of vector space, we can think of different types as different basis vectors, for example, 3 actually corresponds to 3 int, and "a" actually corresponds to "a" string (analogy $\lambda_i \mathbf e_i$), and the values of different types are in principle separated from each other. Relationships are not allowed when types do not match (regardless of automatic type conversion), just as different basis vectors are independent of each other. In this sense, the multiple distribution f (3, "a") can be understood as



Type information is descriptive information that is appended to data at compile time and is not inherently special. In this sense, Loader can be regarded as a more general kind of multiple distribution acting on the tensor product of arbitrary basis vectors.

## Five. Loader as Generator

A generic model loader can be thought of as having the following type definitions:


```
    Loader :: Path -> Model
```

For a universal design, we need to realize that the so-called coding is not only to deal with the immediate needs, but also to take into account the future changes in requirements and the evolution of the system in time and space. In other words, programming is not about the present, the only world, but ** All possible worlds. **. Formally, we can introduce a Possible operator to describe this matter.


```
    Loader :: Possible Path -> Possible Model
    Possible Path = stdPath + deltaPath
```

StdPath refers to the standard path for the model file, and deltaPath refers to the delta custom path used when customizing an existing model file. For example, we have a business process main. WF. XML built into the base product, and we need to use a different process when we customize it for Client A. But we don't want to change the code in the base product. At this point, we can add a delta model file `/_delta/a/main.wf.xml`, which represents the main. WF. XML customized for customer A. Loader will automatically identify the existence of this file and automatically use this file, and all existing business codes do not need to be modified.

If we just want to fine-tune the original model, rather than completely replace it, we can use the X: extends inheritance mechanism to inherit the original model.


```java
Loader<Possible Path> = Loader<stdPath + deltaPath> 
                      = Loader<deltaPath> x-extends Loader<stdPath>
                      = DeltaModel x-extends Model
                      = Possible Model
```

In the Nop platform, the model loader is actually implemented in two steps.


```java
interface IResource{
    String getStdPath(); // 文件的标准路径
    String getPath(); // 实际文件路径
}

interface IVirtualFileSystem{
    IResource getResource(Strig stdPath);
}


interface IResourceParser{
    IComponentModel parseFromResource(IResource resource);
}
```

IVirtualFile System provides a differential file system similar to the overlayfs used by Docker containers, while IResourceParser is responsible for parsing a specific model file.

The theory of reversible computation suggests a general software construction formula.


```java
App = Delta x-extends Generator<DSL>
```

Based on this theory, we can regard Loader as a special case of Generator and Path as a minimizing DSL. After loading a model object according to the path, we can continue to apply the formula of reversible calculation to convert and modify the model object, and finally get the model object we need. As an example,

In the Nop platform, we define a definition file orm. XML for ORM entity objects. Its function is similar to the HBM file in Hibernate. The approximate format is as follows:


```xml
<orm x:schema="/nop/schema/orm/orm.xdef" xmlns:x="/nop/schema/xdsl.xdef">
  <entities>
    <entity name="xxx" tableName="xxx">
       <column name="yyy" code="yyy" stdSqlType="VARCHAR" .../>
       ...
    </entity>
  </entities>
</orm>
```

Now we need to provide a visual designer for this model file. What do we need to do? In the Nop platform, we only need to add the following description:


```xml
<orm>
   <x:gen-extends>
      <orm-gen:GenFromExcel path="my.xlsx" xpl:lib="/nop/orm/xlib/orm-gen.xlib" />
   </x:gen-extends>
    ...
</orm>
```

The X: gen-extends is a metaprogramming mechanism built into the XLang language, which is a code generator that executes at compile time to dynamically generate base classes for the model. `<orm-gen:GenFromExcel>` Is a user-defined tag function, which is used to read and parse the Excel model, and then generate the orm definition file according to the requirements of the orm. XML format. The format of the Excel file is shown in the following figure:

[excel-orm](excel-orm.png)

The format of the Excel model file is actually very close to the format of the requirements document we use in our daily life (the Excel file format in the example itself is copied and pasted from the requirements document). Only needs to edit the Excel document to be possible to realize to the ORM entity model visualization design, moreover this kind of design revision is ** Effective immediately **! Thanks to the dependency tracking capabilities of the IResource Component Manager, the orm model is recompiled whenever the Excel model file is modified.



Some people may not be satisfied with the way Excel is edited and want to adopt a graphical designer like Power Designer. No Problem！ Just swap out the metaprogramming generator, and it's really a matter of words.


```xml
<orm>
   <x:gen-extends>
      <orm-gen:GenFromPdm path="my.pdm" xpl:lib="/nop/orm/xlib/orm-gen.xlib" />
   </x:gen-extends>
    ...
</orm>
```

Now we can happily design the mockup in Power Designer.



The above example epitomizes the concept of Representation Transformation in ** Representation transformation ** the theory of reversible computation. What really matters is the core ORM model object, and visual design is just using some representation of this model object, which can be reversibly transformed between different representations. ** Appearance is not the only one! ** Moreover, we need to note that the representation transformation does not need to involve the runtime at all (that is, the designer does not need to know any relevant information about the ORM engine), and it is entirely a matter of the formal level (similar to some formal transformation at the mathematical level). Many designers for low-code platforms today can't exist without specific runtime support, which is actually an unnecessary limitation.



Now there's an interesting question. To support `<orm-gen:GenFromExcel>`, do we need to write a parser for a specific Excel model file that parses the Excel document with the sample format? In the NOP platform, the answer is: ** No need **.



An orm model is essentially an object of a Tree structure, and the constraints that this Tree structure needs to satisfy are defined in the orm. Xdef file. Excel model is a visual representation of orm model, and it can also be mapped to a Tree structure. If this mapping can be described by some deterministic rules, we can use a unified Excel parser to complete the model parsing.


```java
interface ExcelModelParser{
    XNode parseExcelModel(ExcelWorkbook wk, XDefinition xdefModel);
}
```

So, the reality is, ** As long as the xdef meta-model file is defined, we can use Excel to design the model file. **. When the xdef metamodel is defined, the parsing, decomposition, merging, delta customization, IDE hints, and breakpoint debugger of the model are automatically obtained without additional programming.

In the Nop platform, the basic technical strategy is that xdef is the origin of the world, and as long as you have the xdef metamodel, you automatically have everything on the front and back end. If you are not satisfied, differential customization will help you fine-tune and improve.



> In the sample Excel model file, the formatting is relatively free. You can add and remove rows and columns at will, as long as it can be converted to a Tree structure in some ** Natural ** way. If we use the terminology of category theory of high lattice, we can say that Excel ModelParser is not a conversion function from a single Excel model object to a single Tree model object, but a conversion function that acts on the whole Excel category. It is mapped to a functor of Tree category (the functor acts on every object in the category and maps them to an object in the target category). The way category theory solves problems is so exaggerated that it ** By solving every problem in the category and then claiming that a specific problem is solved. **. There is only one reason for such a crazy scheme to succeed: It's science.



Finally, the key point of reversible computation is re-emphasized:

1. Total quantity is a special case of differential quantity, so the original configuration file itself is a legal description of differential quantity, and the transformation of reversible calculation can completely eliminate the need to modify the existing configuration file. Taking Baidu's amis framework as an example, adding reversible calculation support for amis's JSON file in Nop platform only changes the loading interface from JsonPageLoader to IResource Component Manager, and does not need to change the original configuration file in principle. There is no need to change any logic at the application level.

2. Before entering the world of strong types, there is a unified layer of weak types. Reversible computation can be applied to any Tree structure (including but not limited to JSON, yaml, XML, vue). Reversible computation is essentially a formal transformation problem, which can be completely independent of any runtime framework and can be an upstream part of multi-stage compilation. Reversible computing provides a series of infrastructure support for the construction, compilation and transformation of domain-specific languages and domain-specific models. As long as the built-in merging operation and dynamic generation operation of reversible computing are used, the decomposition, merging and abstraction of domain models can be realized in a general way. This mechanism can be used for Workflow and BizRules on the back end, as well as for front-end pages. Similarly, it can be applied to AI models, distributed computing models and so on. The only requirement is that these models need to be expressed in some structured Tree form. For example, applying this technology to k8s is essentially the same as kustomize, which k8s is currently pushing. [https://zhuanlan.zhihu.com/p/64153956](https://zhuanlan.zhihu.com/p/64153956)

3. Any interface that loads data, objects, and structures by name, such as loader, resolver, require, and other functions, can become an entry point for reversible computation. On the surface, path name is the simplest atomic concept without internal structure, but reversible computation points out that any quantity is the result of differential computation and has internal evolutionary power. Instead of thinking of the pathname as a symbol that points to a static object, we can think of it as a symbol that points to the result of a computation, a possible future world. Path -> Possible Path -> Possible Model

## Brief summary

Briefly summarize what has been covered in this article.

1. The linear system is good

2. Multilinear systems can be reduced to linear systems.

3. The heart of a linear system is Loader: Path- > Model

4. Loader can be extended to Possible Path-> Possible Model, Load = Composition

5. The theory of reversible computation provides a more profound theoretical explanation.



For more on the theory of reversible computation, see my previous article.

[Reversible computation] (https://zhuanlan.zhihu.com/p/64004026)

[Technical Realization of Reversible Computation](https://zhuanlan.zhihu.com/p/163852896)
