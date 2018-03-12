---
layout: post
title:  "多重继承的Vtable"
category: C++
---
在上一篇“[用C＋＋ Vtable分析dynamic_cast]({{ site.baseurl }}{% post_url 2018-02-26-c++-vtable-for-dynamic_cast %})”初步分析Vtable后，这篇继续分析多重继承的Vtable。

{% highlight c++ %}
class Base1 {
public:
    virtual void b1foo1()
    {
        std::cout << "Base1::b1foo1()\n";
    }
    virtual void b1foo2() {}

    long long int a;
};

class Base2 {
public:
    virtual void b2foo1()
    {
        std::cout << "Base2::b2foo1()\n";
    }
    virtual void b2foo2() {}

    long long int b;
};

class MI : public Base1, public Base2 {
public:
    virtual void b1foo1()
    {
        std::cout << "MI::b1foo1()\n";
    }
    virtual void b2foo1()
    {
        std::cout << "MI::b2foo1()\n";
    }
    virtual void mifoo1() {}
    virtual void mifoo2() {}

    long long int c;
};

MI mi;
{% endhighlight %}
根据我们对C++多态的理解，我们大概能勾画出MI对象的内存布局：
* 构造对象mi时，根据其基类列表继承顺序，依次构造Base1对象、Base2对象和MI类的数据成员；
* 基类Base1虚函数表中Base1:b1foo1()会被MI::b1foo1()替换；
* 基类Base2虚函数表中Base2:b2foo1()会被MI::b2foo1()替换；
* 派生类MI新增的虚函数MI::mifoo1()和MI::mifoo2()会放在其第一个基类Base1虚函数表的末尾。

对象mi的内存布局描述如下：
{% highlight c++ %}
                            +------------------------+
                            |     0 (top_offset)     |
                            +------------------------+
mi --> +----------+         | ptr to typeinfo for MI |
       |   vptr   |-------> +------------------------+
       +----------+         |      MI::b1foo1()      |
       |     a    |         +------------------------+
       +----------+         |     Base1::b1foo2()    |
       |   vptr   |         +------------------------+
       +----------+         |      MI::mifoo1()      |
       |     b    |         +------------------------+
       +----------+         |      MI::mifoo2()      |
       |     c    |---+     +------------------------+
       +----------+   |     |    -16 (top_offset)    |
                      |     +------------------------+
                      |     | ptr to typeinfo for MI |
                      +---> +------------------------+
                            |      MI::b2foo1()      |
                            +------------------------+
                            |     Base2::b2foo2()    |
                            +------------------------+
{% endhighlight %}
然后我们加上`-fdump-class-hierarchy`选项，看看是否和g++生成派生类MI的Vtable一致？
{% highlight c++ %}
Vtable for MI
MI::_ZTV2MI: 11u entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI2MI)
16    (int (*)(...))MI::b1foo1
24    (int (*)(...))Base1::b1foo2
32    (int (*)(...))MI::b2foo1
40    (int (*)(...))MI::mifoo1
48    (int (*)(...))MI::mifoo2
56    (int (*)(...))-16
64    (int (*)(...))(& _ZTI2MI)
72    (int (*)(...))MI::_ZThn16_N2MI6b2foo1Ev
80    (int (*)(...))Base2::b2foo2

Class MI
   size=40 align=8
   base size=40 base align=8
MI (0x0x7f339f8c7620) 0
    vptr=((& MI::_ZTV2MI) + 16u)
  Base1 (0x0x7f339fa05660) 0
      primary-for MI (0x0x7f339f8c7620)
  Base2 (0x0x7f339fa056c0) 16
      vptr=((& MI::_ZTV2MI) + 72u)
{% endhighlight %}
其实和我们想象的有差别，主要差别是：派生类虚函数MI::b2foo1()是如何覆盖第二个基类的Base2::b2foo1()虚函数。
1.  g++首先用派生类虚函数MI::b1foo1()替换第一个基类Base1虚函数表中的Base1:b1foo1();
2.  然后把派生类MI剩下所有的虚函数按照定义顺序依次加到第一个基类Base1虚函数表的末尾；
3.  最后第二个基类Base2虚函数表中的Base2:b1foo1()被一个特殊的_ZThn16_N2MI6b2foo1Ev symbol所替代。

我们用`c++filt`解析下这个特殊的symbol：
{% highlight c++ %}
$ c++filt _ZThn16_N2MI6b2foo1Ev
non-virtual thunk to MI::b2foo1()
{% endhighlight %}
wiki "[thunk in object-oriented programming]"中对`thunk`的描述：
>Thunks are useful in object-oriented programming platforms that allow a class to inherit multiple interfaces, leading to situations where the same method might be called via any of several interfaces.

[thunk] wiki中多重继承的例子和本篇是一样的，仅仅类名不同，但是为了方便理解，我把wiki中的类名和虚函数替换为本篇对应的类名和虚函数名。
>As an alternative, the compiler can generate an adjustor thunk along with MI's implementation of b2foo1() that adjusts the instance address by the required amount and then calls the method. The thunk can appear in MI's dispatch table for Base2, thereby eliminating the need for callers to adjust the address themselves.

为什么需要`thunk`，为什么不是直接在Base2虚函数表里用MI::b2foo1()覆盖Base2::b2foo1()呢？该wiki中另外一句话解决了我对多态认识的一个误区：
> If it refers to an object of type MI, the compiler must ensure that MI's b2foo1 implementation receives an instance address for the entire MI object, rather than the inherited Base2 part of that object.

先用一个多态的例子来解释：
{% highlight c++ %}
MI mi;
std::cout << "mi's address: " << &mi << std::endl;
Base1 *b1 = &mi;
std::cout << "b1's value:   " << b1 << std::endl;
b1->b1foo1();
Base2 *b2 = &mi;
std::cout << "b2's value:   " << b2 << std::endl;
b2->b2foo1();
{% endhighlight %}
这是最基本的多态例子，运行的结果是：
{% highlight c++ %}
mi's address: 0x7fff643e2ef0
b1's value:   0x7fff643e2ef0
MI::b1foo1()
b2's value:   0x7fff643e2f00
MI::b2foo1()
{% endhighlight %}
* 因为Base1是派生类MI的第一个基类，mi对象中Base1对象的首地址就是mi对象的首地址，所以b1指向的就是mi对象的首地址；
* 因为sizeof(Base1)为16 byte，所以mi对象中第二个基类Base2对象的首地址（b2指向的地址）= mi对象的首地址 + 第一个基类大小（0x10）。

根据多态原理，在运行时`b2->b2foo1()`实际上调用的是派生类MI::b2foo1()定义的虚函数，那么就存在一个问题：b2指向的仅仅是对象mi中的一个子对象，而MI::b2foo()定义的虚函数可能会使用一些在派生类MI新增加的数据成员？所以Base2的虚函数表不能简单地用MI::b2foo1()覆盖Base2::b2foo1()，如wiki描述那样，调用MI::b2foo1()函数必须是一个完整的MI对象，而不能仅仅是MI对象的子对象。Base2虚函数表中的“non-virtual thunk to MI::b2foo1()”会把b2重新强制转换为指向完整MI对象的首地址（用上一篇中提到的dynamic_cast），这时就得到MI对象完整的vtable，同时也就能安全地调用MI::b2foo1()虚函数。

虽然b1->b1foo1()在运行时调用的也是派生类MI::b1foo1()定义的虚函数，但是b1指向的地址其实就是完整MI对象的首地址。

参看资料：
* [thunk]
* [What is the VTT for a class?]

[thunk]: https://en.wikipedia.org/wiki/Thunk
[thunk in object-oriented programming]: https://en.wikipedia.org/wiki/Thunk#Object-oriented_programming
[What is the VTT for a class?]: https://stackoverflow.com/questions/6258559/what-is-the-vtt-for-a-class
