# A Heuristic Derivation of Y Combinator

Y Combinator is a key concept in the theory of Lambda calculus, through which we can implement anonymous recursive calls to functions. As for its explanation, it is generally called "all who understand", in other words, people who do not understand probably do not understand after reading it. In this article, I want to provide a heuristic derivation that makes the construction of the Y combinator as intuitive as possible. For the convenience of students who are not familiar with lambda calculus, I have also added a brief introduction to the rules of lambda calculus in the appendix section.

## One. Y Combinator

First, let's look at the basic form of a recursive function:


```javascript
let f = x => 函数体中用f指代自身，实现递归调用
// 例如阶乘函数
let fact = n => n < 2 ? 1 : n * fact(n-1)
```

> The above recursive function is a so-called first-order recursive function, that is, a function whose definition refers only to itself to cause recursion

We see that the implementation of the recursive function refers to itself by the function name f. If we want to eliminate this reference to itself, then ** You must convert it to a parameter. **. Thereby obtaining


```javascript
let g = f => x => 函数体中用f表示原递归函数
```

The function G is equivalent to adding a layer on the basis of f, making it a higher-order function. Because f is some arbitrary recursive function, the only thing we know about function G is that it acts on function f.


```javascript
g(f) = x => 函数体中的f通过闭包变量引用了参数f
```

Obviously, the return value of G (f) is the target recursive function we need, from which we obtain the so-called fixed point equation.


```javascript
g(f) = f
```

The function G acts on the argument f and returns a result that is also equal to f, in which case ** F is called the fixed point of the function G **.

Now we can formulate a standard procedure for constructing anonymous recursive functions:

1. Define the helper function G ** ** in terms of the named function f

2. ** Find the fixed point ** of the function G

Suppose ** There is a standard solution to get the fixed point of G. ** we write this solution as Y,


```javascript
f = Y g  ==>  Y g = g (Y g)
```

If the solution to Y exists, what does it look like? In order to solve the fixed point equation, a common method is the iterative method: we repeatedly apply the original equation, and then examine the results of the evolution of the system.


```java
f = g(f) = g(g(g...))
```

If completely expanded, then f corresponds to an infinitely long sequence. ** Suppose this infinitely long sequence can be squared. **


```java
f = g(g(g...)) = G(G) = g(f) = g(G(G))
```

If such a function G exists, what is its definition? Luckily, ** G (G) = G (G (G)) itself can be regarded as the definition of the function G. **


```java
G(G) = g(G(G)) ==> G = λG. g(G(G)) = λx. g(x x)
```

The last equal sign in the above equation corresponds to the parameter name renaming of the function, the so-called alpha-transformation in the lambda calculus.

In the case that G is known, the definition of Y is obvious.


```
Y g = f = G(G) = (λx.g (x x)) (λx.g (x x))   (1)
Y = λg. (λx.g (x x)) (λx.g (x x))            (2)
```

In the above formula, (1) is directly substituted into the definition of G. And (2) is to regard Y G as the definition of Y


```
 Y g = expr ==> Y = λg. expr
```

We can go ahead and perform the alpha-transformation, changing the parameter names so that the definition of the Y combinator is as usual in the general literature.


```
Y = λf. (λx.f (x x)) (λx.f (x x))
```

It can be checked that Y does satisfy the fixed point equation

<img title="" src="Y-combinator.png" alt="Y-combinator.png" width="457">

The Y in the above diagram is actually Y f, the result of the action of Y on f, so it veriﬁes Y f = f (Y f).

## Two. The essence of recursion G (G)

In the derivation of the previous section, the most crucial step is ** f = G(G) ** that we decompose the square root of the recursive function into the result of the G function acting on itself. In the following, we will deepen our understanding of the necessity of this structure through a concrete example.

** There must be two kinds of structures inside the so-called recursive function: recursive structure and (non-recursive) computational structure. **。 For example


```javascript
let fact = n => n < 2 ? 1 : n * fact(n-1)
// 或者写成函数声明的形式
function fact(n){
    return n < 2 ? 1: n*fact(n-1)
}
```

The fact function not only completes the calculation of this step, but also implements recursion by referring to itself through the fact name.

If we try ** Separation of recursive structures from computational structures **, we can define the following pure calculation structure (all the variables used in the calculation are passed by parameters, and there is no recursion caused by self-reference).


```javascript
function fact0(fact, n){
    return n < 2 ? 1 : n * fact(n-1)
}
// 或者使用lambda表达式形式
const fact0 = fact => n => n < 2 ? 1 : n * fact(n-1)
```

If the recursive structure can be abstracted into an operator Y, we can expect to increase the recursive structure for fact0 by calling the form as follows.


```javascript
Y(fact0)(n) = fact(n), fact = Y(fact0)
```

That is to say, the operator Y can be regarded as a processor, which eats the fact0 without recursion and spits out the fact with recursion function. Fact0 is a two-argument function, while the resulting fact is a one-argument function.

Next, we note that fact0 is something that we think already exists, and fact is something that we need to construct, which is unknown. The definition of fact0 contains both known and unknown quantities, so we need to ** Rewrite the definition of fact0 to be constructed entirely based on known quantities **


```javascript
const fact1 = fact1 => n => n < 2 ? 1: n * fact1(fact1)(n-1)
```

In the above expression, fact1 in the function body actually points to the parameter name fact1, not the function name. If the name is confusing, we can rewrite it again as


```javascript
const fact1 = self => n => n < 2 ? 1 : n * self(self)(n-1)
```

We can verify that fact (n) is equivalent to fact1 (fact1) (n)


```javascript
Y(fact0)(n) = fact(n) = fact1(fact1)(n)
```

Comparing the two sides of the equation, we can see that


```javascript
Y(fact0) = fact1(fact1), 也就是上一节中的 Y g = G(G)
```

Through the above analysis, we have noticed a basic structure `x(x)` that must appear in recursive functions.

If we define the following function


```javascript
function w(x){
   return x(x)
}
// 或者写成lambda表达式形式
const w = x => x(x)
```

Clearly w (G) = G (G), and w (w) provides the simplest and most essential form of an infinite loop. In the theory of lambda calculus, the above w function is said to be an Omega combinator,

$$
\omega := \lambda x. x\ x \\
$$

$$
\Omega := \omega \omega = (\lambda x. x \ x)(\lambda x. x \ x)\\
$$

$$
(\lambda x.x x)(\lambda x.x x) \mapsto (\lambda x.x x)[x\mapsto (\lambda x.x x)]
\mapsto (\lambda x.x x)(\lambda x.x x)\\


$$

That is to say

$$
\omega \omega = \omega \omega\\
$$

When w acts on itself, it can continuously generate itself, so X (X) can encode an infinite loop, so the ability to introduce an infinite loop for any function F can take the form


```javascript
const wF = x => F(x(x))
```

Note that the w function returns `x(x)` directly, while wF takes the `x(x)` function F as an argument. Referring to the analysis of the fact0 function above, wF is equivalent to replacing the fact function call in fact0 with `fact1(fact1)`.

The wF function corresponds to

$$
\omega_F := \lambda x. F(x\ x)\\
$$

The definition of the Y combinator corresponds to

$$
Y_F := \omega_F\omega_F = (\lambda x. F(x\ x))(\lambda x. F(x\ x))\\
$$

Written as a JavaScript function of the form


```javascript
const Y = f => {
    const g = x => f(x(x));
    return g(g);
}
```

### Three. Z Combinator

If we actually implement the Y combinator in the form of the previous section, we will find that `Y(fact0)(3)` we throw a stack overflow error and do not get a result `fact(3)`.

This is because JavaScript uses the call-by-value calling convention (CBV), where the values of its arguments must be computed before they are passed into the function, and the f (X (X)) call creates an endless loop.

In order to solve this problem, to implement the Y combinator in JavaScript, we need to make the parameter lazy to evaluate. In short, we need to make the direct parameter lazy to load. This step corresponds to the so-called eta-reduction in lambda calculus.


```diff
- Y = λf.(λx.f (x x)) (λx.f (x x))
+ Z = λf.(λx.f (λy. (x x) y)) (λx. f (λy. (x x) y))
```

What we get in this way is the so-called Z combinator. Corresponding to JavaScript


```diff
function Z(f) {
    const g = x => {
-       return f(x(x));
+       return f(y => x(x)(y));
    };
    return g(g);
}

fact(n) == Z(fact0)(n)
```

## Four. Turing Combinator

Y Combinator is not the only fixed point combinator of a function, in fact, it is infinite. We recall the definition of fixed point.

$$
f_* = f(f_*)\\
f_*  = \Theta f\\
\Theta f = f(\Theta f) \\
$$

Following the recursive expansion in the first section, we can get a new recursive equation.

$$
\Theta f = f(f(\Theta f)) = f(f(... f)) = \Theta' \Theta' f = 
f(\Theta' \Theta' f)
$$

$$
\Theta \equiv \Theta'\Theta' \\
$$

The definition of Theta 'can be read directly from the above formula

$$
\begin{aligned}
\Theta' \Theta' f &= \Theta'(\Theta')(f)\\
 &= (\lambda \Theta'\lambda f.f(\Theta'\Theta'f))(\Theta')(f)\\
\end{aligned}
$$

$$
\Theta' := \lambda t.\lambda f. f( t\ t \ f)
$$

The Theta obtained by this decomposition is the so-called Turing combinator.

## Appendix: Lambda calculus

The lambda calculus, which claims to be the simplest and smallest formal system, contains three syntax rules.


```text
x | λx.expr | expr expr
```

Where X denotes the variable definition and λx.expr denotes the function definition. Expr expr represents a function call,

The above three cases respectively correspond to the


```text
x
x => expr
expr(expr)
例如
expr0 = 2
expr1 = x => x + 1
expr1(expr0) = 2 + 1 
```

> The amazing thing about lambda calculus is that we can do all the general purpose calculations by just relying on the above three rules, and it is the simplest form of a general purpose programming language!

Agreement:

1. The function is left associative. This means that f X y ≡ (f (X) y)
2. The lambda operator is bound to the entire expression that follows it so that many parentheses can be omitted, for example ( (λx. (X X)) (λy.y)) can be abbreviated to (λx.x X) λy.y

Three axioms:

1. Alpha-transform: The name of the function parameter is an arbitrary λx. (λx.x) X is equivalent to λy. (λx.x) y.

2. Beta-reduction: The function application (operation process) is the replacement process of the symbol template.
   
   (λx. M) N means that all symbols with the name X in the function body M are replaced by N. For example (λf.f 3) (λx.x + 2) = (λx.x + 2) 3 = 3 + 2 = 5

There is an implicit inherent complexity here: there is no general way to decide whether two lambda expressions are equivalent. Church proposed the lambda calculus in order to prove this theorem and thus disprove the Hilbert decision problem.

3. Eta-reduction: If the operation results are always equal, then the expression of λ is equivalent to f ≡ λx.f X

Church-Rosser theorem: When applying reduction rules to lambda terms and lambda calculus, the order of reduction chosen will not affect the final result

## Reference

[Y Combinator Details (The Y Combinator)] (https://blog.csdn.net/universsky2015/article/details/100891288)

Lambda Calculus Written by Cognitive Scientists to Xiaobai (https://zhuanlan.zhihu.com/p/30510749)

[Derive Y Combinator] (https://zhuanlan.zhihu.com/p/137588168)

[Reinventing the Y Combinator for JavaScript (ES6)] (https://zhuanlan.zhihu.com/p/20616683)

[Fixed-Point Combinators in JavaScript](https://benestudio.co/fixed-point-combinators-in-javascript/)

[Lambda calculus encodings; Recursion](https://groups.seas.harvard.edu/courses/cs152/2015sp/lectures/lec07-encodings.pdf)

[On Recursive Functions](https://deniskyashif.com/2019/05/15/on-recursive-functions/)
