---
layout: post
title:  "C++构造函数中的异常"
category: C++
---
C++构造函数是否可以抛出异常？  
答案是肯定的。因为构造函数没有返回值，如果构造函数发生错误（比如，异常或者资源暂时不可用），如何让使用者知道对象并没有成功创建呢？在构造函数里抛出异常是一个很不错的解决方法。不过，在构造函数中抛出异常并不是唯一的解决方法，在C++ FAQ的“[How can I handle a constructor that fails?]”中还提到：可以把创建失败的对象设置为“zombie”状态。在大多数情况下，在构造函数中抛出异常是一种更好的选择。
关于构造函数抛出异常，从下面两个问题来展开分析：
1. 如果构造函数发生错误，那么已经创建的子对象（比如，类的数据成员，类的父类），这些子对象是否能正确析构？
2. 用new创建对象指针时，会先申请内存，然后用构造函数来初始化申请的内存空间。如果构造函数发生错误，new申请的内存是否会发生泄漏？

### 1. 已经创建的子对象析构问题
在C++中，构造函数初始化其父类和数据成员，而析构函数是构造函数的逆操作——释放数据成员和父类所占用的资源。如果构造函数失败，也就意味着该对象并不真正存在，那么其对应的析构函数也就无法被调用。因为析构函数是被类实例化的对象所调用的；而且自定义的析构函数中也无法很容易地区分哪些子对象已经成功创建，哪些没有被创建。下面引用文章[Constructor Failures]对这种情况更形象的描述：
>Incidentally, this is why a destructor will never be called if the constructor didn't succeed -- there's nothing to destroy. "It cannot die, for it never lived." Note that this makes the phrase "an object whose constructor threw an exception" really an oxymoron. Such a thing is even less than an ex-object... it never lived, never was, never breathed its first. It is a non-object.

在析构函数无法被调用的情况下，那么编译器是否能保证已经成功创建的子对象能被正确析构呢？  
答案同样是肯定的！因为一个类实例化对象时，会实例化该对象的一系列子对象（比如父类，或者数据成员）；也就是说这些子对象是它们所属于的对象的一部分，而且无法脱离其所属于的对象单独存在。一个对象和该对象所包含的一系列子对象的生命周期是一致的。一旦类实例化时构造函数失败，意味着该对象生命周期结束，那么该对象中所有已创建的子对象生命周期也就是意味着结束（还没来得急创建或者创建失败的子对象因为不存在，也就是不存在生命周期结束的问题），编译器能保证调用那些已经创建的子对象的析构函数来释放资源。

在调用构造函数实例化对象时，编译器是如何区分哪些子对象需要调用其析构函数，而其它的子对象则不需要呢？  
目前还没有深入了解这方面的详细资料。我猜想可以借助于构造函数和析构函数互逆特点来实现：因为构造函数根据类所继承的父类列表顺序和定义的数据成员顺序依次来初始化这些子对象；而析构函数则完全相反顺序：先调用自己的析构函数（析构所定义的数据成员），然后再按继承层次依次向上调用各基类析构函数。这个过程可以抽象为一系列入栈出栈动作。在调用构造函数时，按照子对象创建顺序依次将已经创建的子对象加入一个特殊的“栈”中；一旦发生构造函数错误，这个特殊“栈”保存着所有已经成功创建的子对象，依次弹出栈顶子对象并调用其析构函数来释放其所占用的资源。

下面从代码的角度来验证：
{% highlight c++ linenos %}
#include <iostream>
#include <stdexcept>
#include <memory>

class Base1 {
public:
    Base1() { std::cout << "Base1's ctor!\n"; }
    ~Base1() { std::cout << "Base1's dtor\n"; }
};

class Base2 : public Base1 {
public:
    Base2() { std::cout << "Base2's ctor!\n"; }
    ~Base2() { std::cout << "Base2's dtor\n"; }
};

class Base3 {
public:
    Base3() { std::cout << "Base3's ctor!\n"; }
    ~Base3() { std::cout << "Base3's dtor\n"; }
};

class Data1 {
public:
    Data1() { std::cout << "Data1's ctor!\n"; }
    ~Data1() { std::cout << "Data1's dtor\n"; }
};

class Data2 {
public:
    Data2() { std::cout << "Data2's ctor!\n"; }
    ~Data2() { std::cout << "Data2's dtor\n"; }
};

class Derived: public Base2, public Base3 {
public:
    Derived() : Base2(), Base3(), pd2(new Data2())
    {
        std::cout << "Derived's ctor!\n";
        std::cout << "Now throw a exception in Derived's ctor...\n";
        throw std::runtime_error("A fake runtime_error exception in Derived's ctor\n");
    }

    ~Derived()
    {
        std::cout << "Derived's dtor!\n";
        delete pd2;
    }

private:
    Data1 d1;
    Data2 *pd2;    // memory leak if constructor got exception.
    // std::unique_ptr<Data2> pd2;
};

int main()
{
    try {
        Derived d;
    } catch(std::exception &e) {
        std::cerr << "exception caught: " << e.what() << std::endl;
        exit(1);
    }

    return 0;
}
{% endhighlight %}
上面代码的行为和前面分析是一致的！代码输出结果：
{% highlight c++ %}
Base1's ctor!
Base2's ctor!
Base3's ctor!
Data1's ctor!
Data2's ctor!
Derived's ctor!
Now throw a exception in Derived's ctor...
Data1's dtor
Base3's dtor
Base2's dtor
Base1's dtor
exception caught: A fake runtime_error exception in Derived's ctor
{% endhighlight %}
虽然编译器能保证已经成功创建的子对象的析构函数能被调用，然而对fd或者普通指针的数据成员却无能为力。从上面代码输出可以看出，指向Data2的指针pd2初始化了，但是没有被释放。因为fd或者普通指针等资源的释放一般是在对象的析构函数中，然而这种情况下对象的析构函数是不会被调用的，这就会造成了fd或者内存泄漏。为了防止fd或者内存泄漏，构造函数中应该使用RAII封装fd这类资源或者智能指针。RAII和智能指针能保证如果它们成功初始化，生命周期结束时一定能被析构。C++ FAQ的[How should I handle resources if my constructors may throw exceptions?]有更详细的解释。  
那么，用std::unique_ptr代替普通指针，将上面代码的第47和52行注释掉，并且把第53行取消注释：
{% highlight c++ %}
47         // delete pd2;

52     // Data2 *pd2;    // memory leak if constructor got exception.
53     std::unique_ptr<Data2> pd2;    // since C++11
{% endhighlight %}
g++带上`-std=c++11`选项编译后，再看看运行结果，这时输出就和我们预期完全一致：
{% highlight c++ %}
Base1's ctor!
Base2's ctor!
Base3's ctor!
Data1's ctor!
Data2's ctor!
Derived's ctor!
Now throw a exception in Derived's ctor...
Data2's dtor
Data1's dtor
Base3's dtor
Base2's dtor
Base1's dtor
exception caught: A fake runtime_error exception in Derived's ctor
{% endhighlight %}
如果构造函数出现异常，是否可以在构造函数中捕获异常呢？比如，在构造函数出现异常时，输出一些debug相关的信息，等等。  
这时就需要使用`函数try块（Function Try Blocks）`语法。函数try块既能处理构造函数体的异常，也能处理构造函数初始化列表的异常。比如，将类Derived的构造函数加上函数try块：
{% highlight c++ %}
    Derived()
    try : Base2(), Base3(), pd2(new Data2())
    {
        std::cout << "Derived's ctor!\n";
        std::cout << "Now throw a exception in Derived's ctor...\n";
        throw std::runtime_error("A fake runtime_error exception in Derived's ctor\n");
    } catch(std::exception &e) {
        std::cerr << "exception caught: " << e.what() << ", and will re-throw\n";
        throw;
    }
{% endhighlight %}
运行结果如下：
{% highlight c++ %}
Base1's ctor!
Base2's ctor!
Base3's ctor!
Data1's ctor!
Data2's ctor!
Derived's ctor!
Now throw a exception in Derived's ctor...
Data2's dtor
Data1's dtor
Base3's dtor
Base2's dtor
Base1's dtor
exception caught: A fake runtime_error exception in Derived's ctor
, and will re-throw
exception caught: A fake runtime_error exception in Derived's ctor
{% endhighlight %}

### 2. new操作符所申请的内存问题
如果构造函数失败，是无法得到new操作符所申请的内存首地址，这个内存是否会发生泄漏？  
答案是肯定不会！编译器会保证回收所申请的内存空间。
new创建对象指针时，主要完成下面两个步骤：
1. 先用operator new分配对象的内存（对象的大小在编译阶段就已经确定）。如果这步发生失败，那么对应的内存并不存在，也就没有内存泄漏问题，而且还会抛出std::bad_alloc异常；
2. 调用构造函数来初始化第一步所分配的内存空间。这一步是通过placement new操作符完成的。编译会try...catch这一步，如果出现异常，会释放第一步所申请的内存空间。

“C++FAQ”的“[In p = new Fred(), does the Fred memory “leak” if the Fred constructor throws an exception?]”有详细解释，下面引用里面的伪码：
{% highlight c++ %}
// Original code: Fred* p = new Fred();
Fred* p;
void* tmp = operator new(sizeof(Fred));
try {
  new(tmp) Fred();  // Placement new
  p = (Fred*)tmp;   // The pointer is assigned only if the ctor succeeds
}
catch (...) {
  operator delete(tmp);  // Deallocate the memory
  throw;                 // Re-throw the exception
}
{% endhighlight %}

参看资料：
* [C++ FAQ]
* [Constructor Failures]
* C++ Primer, Fourth Edition

[How can I handle a constructor that fails?]: https://isocpp.org/wiki/faq/exceptions#ctors-can-throw
[Constructor Failures]: http://www.gotw.ca/publications/mill13.htm
[How should I handle resources if my constructors may throw exceptions?]: https://isocpp.org/wiki/faq/exceptions#selfcleaning-members
[In p = new Fred(), does the Fred memory “leak” if the Fred constructor throws an exception?]: https://isocpp.org/wiki/faq/freestore-mgmt#new-doesnt-leak-if-ctor-throws
[C++ FAQ]: https://isocpp.org/wiki/faq