---
layout: post
title: "虚继承的Vtable"
category: C++
---
在上一篇“[多重继承的Vtable]({{ site.baseurl }}{% post_url 2018-02-27-c++-vtable-in-multiple-inheritance %})”分析多重继承的Vtable后，这篇继续分析虚继承的Vtable。在虚继承下，对给定虚基类，无论该类在派生层次中作为虚基类出现多少次，只继承一个共享的基类子对象。虚继承除了能解决多个相同基类对象引起的歧义外，还可以节省对象内存空间（共享一个虚基类对象）。

### 1. 没有虚函数和虚继承
{% highlight c++ linenos %}
class VBase {
public:
    long long int m = 0x01020304;  // C++11
};

class Derived : public VBase {
public:
    long long int n = 0x05060708;
};

Derived d;
{% endhighlight %}
用g++带上`-std=c++11`和`-g`选项编译，然后用gdb看看对象d内存情况：
{% highlight c++ %}
(gdb) p/x d
$2 = {<VBase> = {m = 0x1020304}, n = 0x5060708}
(gdb) p &d
$3 = (Derived *) 0x7fffffffe400
(gdb) x/2xg 0x7fffffffe400
0x7fffffffe400: 0x0000000001020304      0x0000000005060708
{% endhighlight %}
所以可以推测出，对象d的内存布局如下：
{% highlight c++ %}
d --> +------------+
      |  VBase::m  |
      +------------+
      | Derived::n |
      +------------+
{% endhighlight %}

### 2. 没有虚函数的虚继承
将上面例子中第6行加上virtual关键字：
{% highlight c++ %}
class Derived : virtual public VBase {
{% endhighlight %}
编译后再用gdb看看对象d的内存：
{% highlight c++ %}
(gdb) p/x d
$2 = {<VBase> = {m = 0x1020304}, _vptr.Derived = 0x400980 <VTT for Derived>, n = 0x5060708}
(gdb) p &d
$3 = (Derived *) 0x7fffffffe400
(gdb) x/3xg 0x7fffffffe400
0x7fffffffe400: 0x0000000000400980      0x0000000005060708
0x7fffffffe410: 0x0000000001020304
{% endhighlight %}
加上virtual关键字变成虚继承后，对象d内存布局有两处变化：
1. 虚基类VBase对象非常意外地被放在对象d的最末尾而不再是起始地址；
2. 对象d大小从16变成24，为了支持虚继承增加了一个vptr，指向VTT而不是vtable。接下来我们先分析下这个vptr，然后用gdb看看这个vptr内存。

首先看看g++带上`-fdump-class-hierarchy`选项后，后缀名为*.002t.class文件的内容：
{% highlight c++ %}
Vtable for Derived
Derived::_ZTV7Derived: 3u entries
0     16u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI7Derived)

VTT for Derived
Derived::_ZTT7Derived: 1u entries
0     ((& Derived::_ZTV7Derived) + 24u)

Class Derived
   size=24 align=8
   base size=16 base align=8
Derived (0x0x7f18c0d42f08) 0
    vptridx=0u vptr=((& Derived::_ZTV7Derived) + 24u)
  VBase (0x0x7f18c0d0a480) 16 virtual
      vbaseoffset=-24
{% endhighlight %}
类Derived的vtable增加了一个新的表项：vbase_offset。因为虚基类对象在最终派生类只存在一份，而该虚基类在继承层次中可能被继承多次，那么就产生一个问题：继承于虚基类的对象们是如何找到他们所共享的虚基类对象的首地址呢？这就是vbase_offset的作用。Itanium C++ABI对[vbase_offset]的描述如下：
>Virtual Base (vbase) offsets are used to access the virtual bases of an object. Such an entry is added to the derived class object address (i.e. the address of its virtual table pointer) to get the address of a virtual base class subobject.

用gdb查看对象d的内存时，Derived的vptr指向的是VTT而不是vtable。Itanium C++ABI对[VTT]的定义如下：
>An array of virtual table addresses, called the VTT, is declared for each class type that has indirect or direct virtual base classes.

在内存中，[VTT位置]位于vtable后面：
>Following the primary virtual table of a derived class are secondary virtual tables for each of its proper base classes, except any primary base(s) with which it shares its primary virtual table.

用gdb debug时得到对象d的vptr指向的是VTT首地址（0x400980），因为在内存中VTT在table后面，所以vtable首地址（0x400968）= VTT首地址（0x400980）- vtable大小（0x18）：
{% highlight c++ %}
(gdb) # ask gdb to automatically demangle C++ symbols
(gdb) set print asm-demangle on
(gdb) set print demangle on
(gdb) x/16xg 0x400968
0x400968 <vtable for Derived>:  0x0000000000000010      0x0000000000000000
0x400978 <vtable for Derived+16>:       0x0000000000400988      0x0000000000400980 <------ VTT memory
0x400988 <typeinfo for Derived>:        0x00000000006010b8      0x00000000004009b0
0x400998 <typeinfo for Derived+16>:     0x0000000100000000      0x00000000004009c0
0x4009a8 <typeinfo for Derived+32>:     0xffffffffffffe803      0x6465766972654437
0x4009b8 <typeinfo name for Derived+8>: 0x0000000000000000      0x0000000000601060
0x4009c8 <typeinfo for VBase+8>:        0x00000000004009d0      0x0000657361425635
0x4009d8:       0x000000543b031b01      0xfffffcb800000009
{% endhighlight %}
因为对象d的vtable包含3个表项（24 bytes），而gdb是16 bytes对齐显示，所以VTT信息无法显示出来，那么看看从地址0x400960开始的内存内容：（vtable首地址0x400968 - 0x8）：
{% highlight c++ %}
(gdb) x/16xg 0x400960
0x400960 <_IO_stdin_used>:      0x0000000000020001      0x0000000000000010
0x400970 <vtable for Derived+8>:        0x0000000000000000      0x0000000000400988
0x400980 <VTT for Derived>:     0x0000000000400980      0x00000000006010b8
0x400990 <typeinfo for Derived+8>:      0x00000000004009b0      0x0000000100000000
0x4009a0 <typeinfo for Derived+24>:     0x00000000004009c0      0xffffffffffffe803
0x4009b0 <typeinfo name for Derived>:   0x6465766972654437      0x0000000000000000
0x4009c0 <typeinfo for VBase>:  0x0000000000601060      0x00000000004009d0
0x4009d0 <typeinfo name for VBase>:     0x0000657361425635      0x000000543b031b01
{% endhighlight %}
可以大概推测出，对象d的内存布局如下：
{% highlight c++ %}
                                     +-----------------------------+
       +----------------+            |      16 (vbase_offset)      |
       | Derived's vptr |----+       +-----------------------------+
mi --> +----------------+    |       |       0 (top_offset)        |
       |   Derived::n   |    |       +-----------------------------+
       +----------------+    |       | ptr to typeinfo for Derived |
       |    VBase::m    |    +-----> +-----------------------------+
       +----------------+            |       VTT for Derived       |
                                     +-----------------------------+
{% endhighlight %}

### 3. 带虚函数的虚继承
{% highlight c++ %}
class VBase {
public:
    virtual void vbfoo1() {}
    virtual void vbfoo2() {}

    long long int m = 0x01020304;  // C++11
};

class Derived : virtual public VBase {
public:
    virtual void vbfoo1() {}
    virtual void dfoo1() {}

    long long int n = 0x05060708;
};

Derived d;
{% endhighlight %}
编译后先用gdb看看对象d的内存：
{% highlight c++ %}
(gdb) p/x d
$1 = {<VBase> = {_vptr.VBase = 0x400a00 <vtable for Derived+72>, m = 0x1020304}, _vptr.Derived = 0x4009d0 <vtable for Derived+24>, n = 0x5060708}
(gdb) p &d
$2 = (Derived *) 0x7fffffffe3f0
(gdb) x/4xg 0x7fffffffe3f0
0x7fffffffe3f0: 0x00000000004009d0      0x0000000005060708
0x7fffffffe400: 0x0000000000400a00      0x0000000001020304
{% endhighlight %}
增加虚函数后，虚基类VBase对象增加了一个vptr。在虚继承下，被共享的虚基类VBase对象也占用一个独立的vptr。这和没有虚继承的单继承内存布局是不一样。接着用g++带上`-fdump-class-hierarchy`选项编译后，更直观地看看后缀名为*.002t.class文件的内容：
{% highlight c++ %}
Vtable for Derived
Derived::_ZTV7Derived: 11u entries
0     16u                               <---- vbase_offset
8     (int (*)(...))0                   <---- top_offset
16    (int (*)(...))(& _ZTI7Derived)    <---- typeinfo for Derived
24    (int (*)(...))Derived::vbfoo1
32    (int (*)(...))Derived::dfoo1
40    0u                                                     <---- vbase_offset。因为这就是VBase的vtable首地址也就是共享的虚基类对象的首地址。
48    18446744073709551600u                                  <---- 0xFFFFFFFFFFFFFFF0，目前还不知道什么作用。
56    (int (*)(...))-16                                      <---- top_offset
64    (int (*)(...))(& _ZTI7Derived)                         <---- typeinfo for Derived
72    (int (*)(...))Derived::_ZTv0_n24_N7Derived6vbfoo1Ev    <---- virtual thunk to Derived::vbfoo1()
80    (int (*)(...))VBase::vbfoo2

VTT for Derived
Derived::_ZTT7Derived: 2u entries
0     ((& Derived::_ZTV7Derived) + 24u)
8     ((& Derived::_ZTV7Derived) + 72u)

Class Derived
   size=32 align=8
   base size=16 base align=8
Derived (0x0x7f894fa52f08) 0
    vptridx=0u vptr=((& Derived::_ZTV7Derived) + 24u)                      <---- Derived's vtable address + 24 bytes
  VBase (0x0x7f894fa1a480) 16 virtual
      vptridx=8u vbaseoffset=-24 vptr=((& Derived::_ZTV7Derived) + 72u)    <---- Derived's vtable address + 72 bytes
{% endhighlight %}
可以大概推测出，这时对象d的内存布局如下：
{% highlight c++ %}
                                         +------------------------------------+
                                         |          16 (vbase_offset)         |
                                         +------------------------------------+
                                         |           0 (top_offset)           |
                                         +------------------------------------+
                                         |    ptr to typeinfo for Derived     |
                            +----------> +------------------------------------+
d --> +----------------+    |            |          Derived::vbfoo1           |
      | Derived's vptr |----+            +------------------------------------+
      +----------------+                 |          Derived::dfoo1            |
      |   Derived::n   |                 +------------------------------------+
      +----------------+                 |          0 (vbase_offset)          |
      |  VBase's vptr  |---------+       +------------------------------------+
      +----------------+         |       |       0xFFFFFFFFFFFFFFF0           |
      |    VBase::m    |         |       +------------------------------------+
      +----------------+         |       |          -16 (top_offset)          |
                                 |       +------------------------------------+
                                 |       |    ptr to typeinfo for Derived     |
                                 +-----> +------------------------------------+
                                         | virtual thunk to Derived::vbfoo1() |
                                         +------------------------------------+
                                         |           VBase::vbfoo2            |
                                         +------------------------------------+
{% endhighlight %}
目前对VTT的作用了解有限，只能从g++生成的后缀名为*.002t.class文件中看出，VTT里的内容和vptr值一致。对虚继承常见的菱形继承，内存布局相似，因为篇幅原因不再详细展开。

参看资料：
* [Itanium C++ ABI]
* [C++ Vtable Example]
* [C++ vtables - Part 1 - Basics]
* [What is the VTT for a class?]

[vbase_offset]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-components
[VTT]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-ctor-vtt
[VTT位置]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-components
[Itanium C++ ABI]: http://itanium-cxx-abi.github.io/cxx-abi/abi.html
[C++ Vtable Example]: https://itanium-cxx-abi.github.io/cxx-abi/cxx-vtable-ex.html
[C++ vtables - Part 1 - Basics]: https://shaharmike.com/cpp/vtable-part1/
[What is the VTT for a class?]: https://stackoverflow.com/questions/6258559/what-is-the-vtt-for-a-class