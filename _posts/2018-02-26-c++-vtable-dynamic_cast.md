---
layout: post
title:  "用C＋＋ Vtable分析dynamic_cast"
category: C++
---
在C++中，派生类和基类指针或者引用之间的转换有两种形式：
1. Upcasting（派生类到基类的转换）  
在public继承情况下，编译器自动进行Upcasting。因为构造派生类对象时先构造基类对象，各基类对象在派生类对象的地址偏移和大小在编译时就确定，这种转换是安全的。  
在protected和private继承情况下，Upcasting会有g++编译错误：  
`error：‘x’ is an inaccessible base of ‘y’`
2. Downcasting（基类到派生类的转换）  
编译器无法自动进行DownCasting。因为指向基类对象的指针或者引用可能仅仅就是指向一个不包含任何派生类的成员的基类对象，如果允许转换的话，会导致派生类对象访问不存在的成员。这时就需要C++ RTTI（Run-time Type Identification，运行时类型识别）的强制类型转换dynamic_cast操作符。

本文所有测试代码都是在Ubuntu 16.04 64 bit和g++ 5.3.1加上-std=c++11选项的环境下测试。

### 1. 没有虚函数
{% highlight c++ %}
class Base {
public:
    void foo()
    {
        std::cout << "Base::foo()\n";
    }
};

class Derived : public Base {
public:
    void foo()
    {
        std::cout << "Derived::foo()\n";
    }
};

Base *basePtr = new Derived();
basePtr->foo();
Derived *derivedPtr = dynamic_cast<Derived *>(basePtr);
assert(derivedPtr);
derivedPtr->foo();
delete derivedPtr;
{% endhighlight %}
在没有任何虚函数情况下，g++编译出错：  
`error: cannot dynamic_cast ‘basePtr’ (of type ‘class Base*’) to type ‘class Derived*’ (source type is not polymorphic)`  
也就是说dynamic_cast必须在多态时才能使用。这种情况下可以使用static_cast来代替dynamic_cast。

### 2. 虚函数
只需要将上面Base类的foo函数加上virtual关键字。可以得到如下结果：  
{% highlight c++ %}
Derived::foo()
Derived::foo()
{% endhighlight %}
加上virtual关键字后，我们可以查看C++对象的内存布局，在g++编译时加上`-fdump-class-hierarchy`选项，编译后会生成一个`<C++ source file name>.002t.class`文件:
{% highlight c++ %}
Vtable for Base
Base::_ZTV4Base: 3u entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI4Base)
16    (int (*)(...))Base::foo

Vtable for Derived
Derived::_ZTV7Derived: 3u entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI7Derived)
16    (int (*)(...))Derived::foo
{% endhighlight %}
因为是64 bit的Ubuntu OS，所以Vtable每个表项占用8 byte。g++使用的是[Itanium C++ ABI]规范，我们根据Itanium C++ ABI来具体分析下Vtable里各表项含义。在[Virtual Table Components and Order]章节，前两项是关于虚继承的，暂时不考虑：
1.  offset_to_top

    >The offset to top holds the displacement to the top of the object from the location within the object of the virtual table pointer that addresses this virtual table, as a  ptrdiff_t. It is always present. The offset provides a way to find the top of the object from any base subobject with a virtual table pointer. This is necessary for dynamic_cast<void*> in particular.

    构造派生类Derived对象时，Derived的Vtable会替代Base的Vtable，当运行`Base *basePtr = new Derived()`时，basePtr指向的是Derived的Vtable而不是Base的Vtable，所以dynamic_cast根据offset_to_top值重新获取到派生类对象的首地址。因为Derived类只有一个Base基类，所以offset_to_top值为0。后面会分析有多个直接基类情况下offset_to_top值的变化。
2.  typeinfo pointer

    >The typeinfo pointer points to the typeinfo object used for RTTI. It is always present. All entries in each of the virtual tables for a given class must point to the same typeinfo object. A correct implementation of typeinfo equality is to check pointer equality, except for pointers (directly or indirectly) to incomplete types. The typeinfo pointer is a valid pointer for polymorphic classes, i.e. those with virtual functions, and is zero for non-polymorphic classes.

    如果执行`typeid(*basePtr).name()`，得到的是`7Derived`（数字7表示类名Derived的长度）而不是`4Base`，同时也验证basePtr确实指向的是Derived的Vtable。

### 3. 带虚函数的多个直接基类
{% highlight c++ linenos %}
class Base1 {
public:
    virtual void foo1()
    {
        std::cout << "Base1::foo1()\n";
    }
};

class Base2 {
public:
    virtual void foo2()
    {
        std::cout << "Base2::foo2()\n";
    }
};

class Derived : public Base1, public Base2 {
public:
    void foo1()
    {
        std::cout << "Derived::foo1()\n";
    }

    void foo2()
    {
        std::cout << "Derived::foo2()\n";
    }
};

Derived derived;
Base1 *base1Ptr = &derived;
std::cout << typeid(*base1Ptr).name() << std::endl;
base1Ptr->foo1();
Base2 *base2Ptr = &derived;
std::cout << typeid(*base2Ptr).name() << std::endl;
base2Ptr->foo2();

Derived *derived1Ptr = dynamic_cast<Derived *>(base1Ptr);
assert(derived1Ptr);
derived1Ptr->foo1();
Derived *derived2Ptr = dynamic_cast<Derived *>(base2Ptr);
assert(derived2Ptr);
derived2Ptr->foo2();
{% endhighlight %}

下面再分析`<C++ source file name>.002t.class`文件里的内容：
{% highlight c++ %}
Vtable for Derived
Derived::_ZTV7Derived: 7u entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI7Derived)
16    (int (*)(...))Derived::foo1
24    (int (*)(...))Derived::foo2
32    (int (*)(...))-8
40    (int (*)(...))(& _ZTI7Derived)
48    (int (*)(...))Derived::_ZThn8_N7Derived4foo2Ev

Class Derived
   size=16 align=8
   base size=16 base align=8
Derived (0x0x7fb91b5d8690) 0
    vptr=((& Derived::_ZTV7Derived) + 16u)
  Base1 (0x0x7fb91b52b600) 0 nearly-empty
      primary-for Derived (0x0x7fb91b5d8690)
  Base2 (0x0x7fb91b52b660) 8 nearly-empty
      vptr=((& Derived::_ZTV7Derived) + 48u)
{% endhighlight %}
* 基类Base2的offset_to_top变为-8，因为基类Base1对象的大小为8 byte，基类Base2对象的首地址=Derived对象的首地址 + Base1对象的大小。如果改变基类Base1的大小，比如在基类Base1里定义数据成员，基类Base2的offset_to_top的值会跟着改变；
* 其实Base1、Base2和Derived对象共享同一个Derived的Vtable。只不过，各自的vprt指向Vtable不同的偏移地址；
* Base1和Base2所指向的typeinfo pointer均为_ZTI7Derived，奇怪的是：即使Base1和Base2之间没有继承关系，Base1和Base2之间竟然可以相互用dynamic_cast强制转换为对方，而用static_cast会编译出错：
{% highlight c++ %}
Derived derived;
Base1 *base1Ptr = &derived;
Base2 *base2Ptr = dynamic_cast<Base2 *>(base1Ptr);
assert(base2Ptr);
std::cout << typeid(*base2Ptr).name() << std::endl;
base2Ptr->foo2();
{% endhighlight %}
运行结果是：
{% highlight c++ %}
7Derived
Derived::foo2()
{% endhighlight %}
对这种情况：说明各基类的指针先用dynamic_cast强制转换到派生类指针，然后派生类指针再Upcasting转换到各基类。

### 参看资料
* [RTTI的实现细节]
* [Itanium C++ ABI]

[Itanium C++ ABI]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html
[Virtual Table Components and Order]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-components
[RTTI的实现细节]: http://hex108.github.io/notes/c++/rtti.html