# Decoupling is much more than dependency injection.

What is coupling? How to decouple? Today, with the prevalence of object-oriented technology, the so-called interaction is expressed as the correlation between objects, and the external manifestation of coupling is the pointer holding related objects, so the decoupling problem seems to be how to minimize the amount of information required for object assembly, and the final solution is the technical concept of dependency injection. See invalid's answer for details.

[What is decoupling] (https://www.zhihu.com/question/20821697/answer/2608624207)

> The Depend</b> <b> ency Inversion Principle (DIP) States that when designing a code structure, higher-level modules should <b> not depend on </b> lower-level modules. Both should <b> rely on </b> their abstraction.

But the idea that dependency injection is the key to decoupling is just a popular understanding of the past. On the one hand, it does provide a practical means of decoupling; ** It does not depend on the object as a whole, but only on the minimal abstract interface. Different objects can implement the same interface. Assembly selection is delayed by a descriptive assembly container. ** however, on the other hand, it is not the whole means of decoupling in software, and we can even say that it should not be the main means of decoupling.

## One. Dependency Injection What? Homomorphic image

The so-called dependence on abstraction rather than on concrete details essentially means that we ** Does not depend on the object as a whole, but only on one of its homomorphic images **.

In mathematics, a homomorphic mapping refers to a transformation in the object space, which maps the objects a and B in the original space O to P (a) and P (B) in the image space, and at the same time, this mapping preserves some operational relations f in the original space, that is, the functions defined in the initial space can be naturally transformed into functions in the image space.


```java
 P(f(a,b)) = f(P(a), P(b))
```

An object implements an interface. An object is an instance of an interface, obj is an instance of interfaceA. In the design, we need to maintain the conceptual "is a" relationship between objects and base classes, as well as between objects and interfaces, in order to ensure that the code written for interfaces can act on derived objects, that is,


```java
f(interfaceA,interfaceB) == f(objA, objB)
```

There can be many functions on an object. Generally, only a small number of functions are functional interface functions necessary for executing business functions, and a large number of other functions are auxiliary functions provided to support dynamic configuration and maintain the life cycle. If we insist that objects only depend on functional interfaces, it is equivalent to homomorphically mapping the problem to a smaller semantic space, thus simplifying the problem.

The descriptive dependency injection container knows the dependencies between all objects, so it can globally sort their construction order, and it can implement lazy assembly through configuration files, lazy loading semantics through caching, etc., which is equivalent to ** Decoupled dependencies between unnecessary object creation orders (global optimization can only be achieved with global knowledge) **. Of course, if the structure of our system is relatively simple and can be extracted into several explicit life cycle stages, the role of the dependency injection container in this respect will be weakened.

An interface can be seen as a local Representation of an object. In order to maximize the effect of this representation, ** A necessary requirement is that the operations in the representation space have some completeness. ** the main business function can be completed through the function operation defined in the representation space. The functions defined on the interface should be justified at the business level, otherwise there will be the problem of so-called abstraction leakage.

## Two. From Homomorphic Mapping to Representation Transformation

In retrospect, the concepts of coupling and decoupling actually come from the fields of mathematics and physics, and the means of decoupling in mathematics and physics are much richer and more powerful.

Recall the most important Fourier transform theory in mathematical physics. Multiple signals with different frequencies are superimposed together. In the time domain, they are completely mixed together. In fact, there are multiple signals at each time point. But through Fourier transform, we can get multiple signals which are completely separated in the frequency domain! So ** In order to achieve decoupling, the most essential and powerful means should be representation transformation, and interface is only the simplest form of representation transformation. **..

One criterion for good or bad decoupling is so-called high cohesion and low coupling. The limit of high cohesion is inseparability (atomicity), while the limit of low coupling is irrelevance. In the case of linear systems, there are always infinitely many optimal solutions for decoupling: ** Simply find an arbitrary set of unit orthogonal vectors. ** (an orthogonal coordinate system rotated by any angle is still a legal orthogonal coordinate system), and then all quantities in the whole space can be obtained by linear combinations of actions on basis vectors.

Representation transformation, simply speaking, can be regarded as coordinate system transformation. The essence of things has not changed, but when we examine them in different coordinate systems, we will mistakenly think that their complexity and the relationship between various parts are changing dramatically. If we think about it in reverse, when we design a system, we can choose the linear system with orthogonal dimensions as our target model, and try to maintain the linearity of all levels of the system, so as to simplify the system structure. See

Low Code Platform Design from Tensor Product (https://zhuanlan.zhihu.com/p/531474176)

If DSL (Domain Specific Language) is regarded as a global representation, the representation transformation can be `F(DSL)` expressed as an interpreter or code generator for DSL. But generally speaking, DSL is a summary of known domain requirements, and its capabilities are always limited, so we must add additional information Delta to produce the target system. From this we get


```java
App = F(DSL) + Delta
```

That is to say, we need to combine the difference information provided by the outside with the information obtained by the representation transformation. But merging means mixing, and mixing generally leads to an increase in entropy (an increase in the degree of chaos in the system). Entropy increases essentially because of the loss of information. ** If the entropy increase can be controlled, the integrity of information must be maintained to ensure that information is not lost, which means that the evolution of the system is reversible in thermodynamics. **.

** Reversible operation means the decoupling of the variance and the ontology. **。 The variable quantity can exist independently from the noumenon, and it has its own meaning.


```java
Delta = App - F(DSL)
```

> In the theory of reversible computing, Docker image can be regarded as a differential structure, and the application image can be stored, transmitted and managed separately from the underlying operating system image.

After introducing the idea of reversible computation, some interesting phenomena will appear. In the general concept, decoupling is to reduce the interaction, which should first reduce the number of objects involved in the interaction. But if there is a reversible computation, decoupling can be achieved by adding a new object Delta. For example, if object A and object B conflict with the same functional design, they can design according to the situation that is most suitable for their own internal structure, without considering the coupling problem caused by avoiding the conflict. When they are used together, the Delta difference can be supplemented to resolve the conceptual conflict between them.


```java
 App = A + B + (-C - D + E) = A + B + Delta
```

Delta is a completely external entity to a and B.

Object-oriented and component theory are essentially trying to draw experience from the production activities of human material world, but with the deepening of software production activities, we ** It is necessary to re-recognize the abstract nature of software. **. Software is an information product that exists in an abstract logical world, and information is not material. ** The structure and production laws of the abstract world are essentially different from those of the material world. **。 The production of material products always has a cost, while the marginal cost of copying software can be zero. To remove a table from a room, you have to go through a door or window in the physical world, but in the abstract information space, you just need to change the coordinates of the table from X to -x. The operational relationship between abstract elements is not limited by many physical constraints, so the most effective mode of production in the information space is not to assemble, but to master and formulate operational rules.



For a complete theoretical exposition of reversible computation, see my article.

[Reversible Computing: Next Generation Software Construction Theory] (https://zhuanlan.zhihu.com/p/64004026)



Some specific implementations can be referred to.

Technical Realization of Reversible Computation (https://zhuanlan.zhihu.com/p/163852896)

What kind of ORM engine is needed for low-code platforms? (1)](https://zhuanlan.zhihu.com/p/543252423)

[What ORM engine is needed for low code platforms? (2)] (https://zhuanlan.zhihu.com/p/545063021)
