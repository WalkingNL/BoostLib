###In a nutshell
The **Boost Lambda Library** (BLL in the sequel) is a C++ template library, which implements a form of lambda abstractions for C++. The term originates from functional programming and lambda calculus, where a lambda abstraction defines an unnamed function. The primary motivation for the BLL is to provide flexible and convenient means to define unnamed function objects for STL algorithms. In explaining what the library is about, a line of code says more than a thousand words; the following line outputs the elements of some STL container a separated by spaces:

Boost Lambda Library（后续简称BLL）作为一个C++模板库，是在C++语言中引入的一种Lambda抽象形式。Lambda抽象(Lambda abstraction)这个术语源自函数式编程以及Lambda算子(Lambda Calculus)。一个Lambda 抽象对应于一个匿名函数(即没有函数名的函数)。对于BLL而言，起初的动机是为了能在STL算法(STL Algorithm)中，更加简便灵活的定义匿名函数对象(unnamed function objects)而设立的。那么这个库到底为何物，我们用一行代码来解释，见以下，其含义是把某个STL容器中的元素输出，以空白符作为间隔。

	for_each(a.begin(), a.end(), std::cout << _1 << ' ');

-- 

The expression `std::cout << _1 << ' '` defines a unary function object. The variable **\_1** is the parameter of this function, a placeholder for the actual argument. Within each iteration of for_each, the function is called with an element of a as the actual argument. This actual argument is substituted for the placeholder, and the “body” of the function is evaluated.

表达式`std::cout << _1 << ' '`定义了一个一元函数对象，变量 **\_1**作为该函数的参数，是实参的占位符(placeholder)。for_each的每一次迭代，这个函数都会以a的元素作为实参来被调用。这个实参替代了占位符(placeholder)，并且函数体被评估(注：这个评估稍微有点迷惑，会在后面作解释)。

-- 

The essence of BLL is letting you define small unnamed function objects, such as the one above, directly on the call site of an STL algorithm.

总而言之，BLL的本质就是让你在STL算法(STL algorithm)的调用位置(call site)上，直接定义类似上面代码所示的，轻量级的函数对象。


## 引进Lambda函数
# 动机
标准模板库STL[[STL94](http://www.hpl.hp.com/techreports ) 是现在C++标准库[C++98]的一部分]是一种通用容器和算法库。通常STL算法通过函数对象操作容器元素。这些函数对象被当做参数传递给这些算法。

任何可以通过使用这种函数调用语法调用的C++构造都是函数对象。STL包含一些常见情况下(例如plus，less 以及 not1)的预定义函数对象。比如对于标准的plus模板来说，一种可能的实现方式是：

	template <class T> 
	struct plus : public binary_function<T, T, T> {
  		T operator()(const T& i, const T& j) const {
    		return i + j; 
  		}
	};
这个基类`binary_function<T, T, T>`含有对入参以及返回类型的typedefs定义（注意：这里的typedef牵涉C++当中的一个关键字typedef，所以不能简单的译成定义），需要这些来构建这个函数对象的适配器(adaptable)


另外，对于基础函数对象类，例如上面的这一个，这个STL包含binder模板，为的是从一个可以通过设置一个参数作为常量值的二元函数对象构建一个一元函数对象。举例来说，不必显式的写一个函数对象，类似以下代码：

	class plus_1 {
  		int _i;
	public:
  		plus_1(const int& i) : _i(i) {}
  		int operator()(const int& j) { return _i + j; }
	};
使用plus模板及binder模板之一就可以创建与上面代码等价的功能。例如，下面的两个表达式创建具有相同功能的函数对象；一旦被调用，两个都返回让这个函数对象的参数增加1这样的结果：

	plus_1(1)
	bind1st(plus<int>(), 1)
下面这一行的子表达式是一个二元函数对象，用以计算两个整数的和，然后bind1st调用这个函数对象将置第一个参数为1。例如使用上面的函数对象，下面代码加1到某个容器a的每一个元素，然后输出结果到标准的深入流couture中去。

	transform(a.begin(), a.end(), ostream_iterator<int>(cout), bind1st(plus<int>(), 1));
为了让构建的binder模板更具可用性，STL包含适用于创建指向函数的指针或引用的适配器，以及创建适用于指向成员函数的指针。最后，一些STL的实现包含作为标准[[SGI02](http://www.sgi.com/tech/stl/)]扩展的函数组合操作。

所有这些工具旨在实现同一个目标：在一个STL算法的调用中，尽可能的指定匿名函数。换句话说，传递代码片段作为一个参数到一个函数中。然而，这个目标只能部分的实现。上面这个简单的例子表明带有标准工具的匿名函数的定义是非常冗杂的。复杂的表达式涉及仿函数(functors)，适配器（adapters），绑定器(binders)，函数组合操作(function composition operations)会使代码理解变得非常困难。除此之外，在使用标准工具的时候还有一些不可忽视的局限性，影响这种功能的使用。例如标准的绑定器(binders)只允许一个二元函数绑定一个参数，没有其他的情况，可以绑定2个、3个参数的三元函数、四元函数。

基于这些不足，Boost Lambda库(Lambda Library)提供对以上问题的解决方案：
匿名函数(unnamed functions)可以非常容易的使用直觉中的语法形式被创建。这样一来，以上的例子便可以写为

	transform(a.begin(), a.end(), ostream_iterator<int>(cout), 1 + _1);
或者更直观的写为:

	for_each(a.begin(), a.end(), cout << (1 + _1));
这样大多数数的绑定参数的限制就不存在了，实际上，任何C++函数的任意参数都可以被绑定。单独支持函数组合操作是没有必要的，因为这种操作其实已被隐式地支持。


# Lambda表达式简介
Lambda表达式在支持函数式编程语言中已被普遍使用，它们的语法形式在不同的语言之间（以及在Lambda算子(Lambda calculus)）有很大差异，但是一个Lambda表达式的最基本的形式是：

	lambda x1 ... xn.e
可以看出，一个Lambda表达式定义了一个匿名函数以及由包含下面两个要点的组合：
#####这个函数的入参：x1 ... xn
#####表达式e，用于根据这些入参 x1 ... xn 计算出该函数的值

下面是一个简单的lambda表达式

	lambda x y.x+y
应用Lambda表达式就意味着用实参替代形参，如下：

	(lambda x y.x+y) 2 3 = 2 + 3 = 5 
Lambda表达式在C++中，这部分lambda x1 ... xn 这部分是隐藏起来的，并且形参已经预定义好了名称。在这个库目前的版本中，有三个已经预定义好的形参，称作占位符(plcaceholders)，分别是：_1, _2 以及 _3，分别指定是这个lambda表达式所定义的第一个，第二个以及第三个参数。例如，上面的例子

	lambda x y.x+y
就是

	_1 + _2
因此，C++lambda表达式没有句法关键字。占位符作为一个操作的用法暗含着这个操作就是一个lambda表达式。然而，这仅能适用于去调用它。没有诸如函数调用、控制结构、转化等Lambda表达式应当还具备的语法形式。故需要特定的句法结构，还有最为重要的是函数的调用必须封装在一个绑定函数(a bind function)中。举例来说，考虑下面的lambda表达式：

	lambda x y.foo(x,y)
而不是这样的形式foo(\_1,_2)，对于C++同行而言，这个表达之应该这样写：

	bind(foo, _1, _2)
我们将这种类型的C++表达式称作绑定表达式。

一个lambda表达式定义了一个C++函数对象，因此函数应用语法就类似调用任何其它的函数对象一样，就如：(_1 + _2)(i, j).

# 偏函数应用
一个绑定函数实际上是一个偏函数应用。在偏函数的应用中，给一个函数的一些参数设定固定的一些值。结果就是另一个函数，可能会有更少的参数，当用未绑定参数调用时，这个新的函数会调用原始的函数。而这个原始的函数携带有已绑定和未绑定参数列表的合并。 

# 术语
一个lambda表达式定义了一个函数。具体来说，一个C++的lambda表达式当被评估时构造了一个函数对象，也叫一个仿函数。我们使用lambda仿函数来指一个函数对象。因此，在这里采用的术语中，评估一个lambda表达式这样的结果就是一个lambda仿函数。

## 如何使用这个库
介绍这一部分的目的是需要说明这个库的一些基本功能。我们有相当多的额外的以及特定的案例，但对这些案例的讨论可能要在后面的部分进行。

# 引入示例
在这个部分，我们给定一些基本的示例，是讲如何在STL算法调用的过程中使用BLL lambda表达式。我们以一些简单的表达式作为开始。首先，使用数值1初始化一个列表(a list container):

	list<int> v(10);
	for_each(v.begin(), v.end(), _1 = 1);
表达式`_1 = 1`创建了一个lambda仿函数(a lambda functor),用于将数值1赋给list对象v中的每一个元素 

接下来，我们创建一个容器，容器内部变量的类型为指针类型，让这些指针指向第一个容器v的元素

	vector<int*> vp(10); 
	transform(v.begin(), v.end(), vp.begin(), &_1);
表达式`&_1`创建了函数对象，用于获取容器v中每一个元素的地址，这个地址会被赋给容器vp的对应元素。

接下来的代码段意指改变容器v里面元素的值，函数foo会被调用。元素的初始值会被作为参数传递给函数foo，然后把得到的结果返回给容器v的每一个元素。

	int foo(int);
	for_each(v.begin(), v.end(), _1 = bind(foo, _1));
下面的一步是对vp里面的元素进行排序

	sort(vp.begin(), vp.end(), *_1 > *_2);
在对sort函数的访问中，我们对vp里面的元素进行了降序处理。

最后，利用for_each输出排序的结果，注意一行输出一个元素

	for_each(vp.begin(), vp.end(), cout << *_1 << '\n');
注意，一般普通(即非lambda)表达式作为lambda表达式的子表达式会立刻被评估。这个可能会致使一些意外情况的发生。例如，如果上面的例子被写成

	for_each(vp.begin(), vp.end(), cout << '\n' << *_1);
这个表达式`cout << '\n'`会立刻被执行，结果就是先输出一个换行，再输出vp中的元素。对于中情况，BLL提供函数常量(functions constant)以及变量(var)来分别的转变常量、变量到lambda表达式中，这可以用以防止子表达式立即被执行这种情况的发生，代码如下：

	for_each(vp.begin(), vp.end(), cout << constant('\n') << *_1);
更多关于这些函数constant 和 var 的描述可以参考这部分内容[dalaying constants and variables](http://file:///D:/E/Study/MyWork/Develop/Coding/CPP/Boost/boost_1_67_0/doc/html/lambda/le_in_details.html#lambda.delaying_constants_and_variables)

# lambda仿函数的参数以及返回类型
一个lambda仿函数在调用期间，其实参可以用占位符进行替代。占位符并不指定这些实参的类型。基本的原则是lambda仿函数可以被携带任意类型的参数调用，只要这个可以进行替换操作的lambda表达式隶属合法的C++表达式即可。如表达式`_1+_2`构建了一个二元lambda仿函数，它可以被具备任意类型A及B的两个对象调用，目的是为了定义这样的操作+(A.B)(并且结合以下内容可知，这样可以让BLL知道这个操作的返回类型)。

C++缺乏这样的一种机制————可以推导一个表达式的返回类型。然而，这种重要的机制是实现C++ lambda表达式的关键所在。因此，BLL在一定程度上，具有复杂类型推导的机制，这种机制就是为推导lambda仿函数的返回类型(resulting type)使用的traits类集合。用它来处理表达式，这些表达式的操作数是内置的类型，以及许多带有标准库类型操作数的表达式。这也同样涵盖了许多用户自定义类型的返回类型的操作，尤其是用户自定义的操作满足传统的惯例，即定义了返回类型。

然而，存在这么一种情况，就是当返回类型不能被推导出来的时候。例如，假设你有这么一个定义：

	C operator+(A, B);
这个符合lambda函数调用的表达式其实是错误的，因为返回类型无法被推导出来：

	A a; B b; (_1 + _2)(a, b);
对于上面的问题，这里有两个替代的解决方案。第一个是扩展BLL类型推导系统，以达到涵盖我们自定义类型返回这样的情况(可以参见这部分内容[“扩充返回类型推导系统”](http://file:///D:/E/Study/MyWork/Develop/Coding/CPP/Boost/boost_1_67_0/doc/html/lambda/extending.html))。第二个是使用特化的lambda表达式(ret)，其可以在适当的位置上定义返回类型（可与参看这部分内容[“重写被推导的返回类型”](http://file:///D:/E/Study/MyWork/Develop/Coding/CPP/Boost/boost_1_67_0/doc/html/lambda/le_in_details.html#lambda.overriding_deduced_return_type)）。

	A a; B b; ret<C>(_1 + _2)(a, b);
为支持绑定表达式(bind expression)，返回类型可以被定义为该绑定函数的模板参数

	bind<int>(foo, _1, _2);

# 关于如何将实参传到lambda仿函数中
对于实参，最大的不足是它们不能作为常量右值(non-const rvalues)来使用。例如：

	int i = 1; int j = 2; 
	(_1 + _2)(i, j); // ok
	(_1 + _2)(1, 2); // error (!)
这个不足似乎并不像看上去那样很糟糕。因为lambda仿函数通常是在STL-algorithms内部被调用的，这些起源于解引用迭代器，而解引用操作符很少返回右值。对于上述例子来说，我们放在“右值作为实参传递给仿函数”的那部分再进行讨论。


