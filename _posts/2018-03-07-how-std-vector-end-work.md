---
layout: post
title: "分析C++ vector::end的实现原理"
category: C++
---
先看看遍历std::vector的代码：
{% highlight c++ %}
std::vector<int> v;
for (vector<int>::iterator iter = v.begin(); iter != v.end(); ++iter)
{
    // do someting with iter
}
{% endhighlight %}
其实，只需要将std::vector进行替换，STL所有的container（顺序容器和关联容器）的遍历都可以套用上面的代码。这篇文章重点讨论vector::end()函数的实现。

STL container的遍历一般都是借助于与container相对应的iterator来完成，要进行遍历就会涉及到从哪儿开始和到哪儿结束—也就是遍历范围的问题，STL container对应的遍历范围为：[begin(), end())：
* begin()：返回的是指向container第一个元素的iterator；
* end()：仅仅是一个占位符（placeholder），返回的iterator指向的是container最后一个元素的后面一个元素（iterators refer to one past the last element），所以遍历时是不能访问这个元素的。下面着重分析下该end()函数实现原理。
* begin() == end()意味着container是空的。

### 1. vector::end的相关的源代码
Ubuntu 16.04, g++ 5.3.1环境下，std::vector源代码位于：/usr/include/c++/5/bits/stl_vector.h。或者g++编译时带上`-g`选项，然后用gdb单步debug含vector的代码就可以找到vector源代码文件。先看看end()函数的实现（注意：下面源代码里的行号显示的都是/usr/include/c++/5/bits/stl_vector.h文件里的行号）：
{% highlight c++ %}
 559       /**
 560        *  Returns a read/write iterator that points one past the last
 561        *  element in the %vector.  Iteration is done in ordinary
 562        *  element order.
 563        */
 564       iterator
 565       end() _GLIBCXX_NOEXCEPT
 566       { return iterator(this->_M_impl._M_finish); }
{% endhighlight %}
end()函数的实现主要涉及到两个上下文：通用的iterator和vector底层存储结构。下面先列出相关的代码，然后再一步步详细分析：
{% highlight c++ %}
  71   template<typename _Tp, typename _Alloc>
  72     struct _Vector_base
  73     {
  74       typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template
  75         rebind<_Tp>::other _Tp_alloc_type;
  76       typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer
  77         pointer;
  78
  79       struct _Vector_impl
  80       : public _Tp_alloc_type
  81       {
  82     pointer _M_start;
  83     pointer _M_finish;
  84     pointer _M_end_of_storage;
         ......
 163     public:
 164       _Vector_impl _M_impl;
 189     };

 213   template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
 214     class vector : protected _Vector_base<_Tp, _Alloc>
 215     {
 221       typedef _Vector_base<_Tp, _Alloc>          _Base;
 227       typedef typename _Base::pointer                    pointer;
 231       typedef __gnu_cxx::__normal_iterator<pointer, vector> iterator;
1496     };
{% endhighlight %}

### 2. 类型\_Vector\_base::pointer的实现
vector底层数据的存储结构和iterator都依赖于类型\_Vector\_base::pointer，其定义如下：
{% highlight c++ %}
typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template rebind<_Tp>::other _Tp_alloc_type;
typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer pointer;
{% endhighlight %}
* typename _Tp：表示的是vector里存储的元素类型；
* typename _Alloc：是负责内存分配和回收相关的，缺省情况下是std::allocator类。

这里使用了C++ traits技巧，引用C++之父的话来解释什么是traits:
>Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation details". - Bjarne Stroustrup

traits主要是为使用者提供类型信息。

下面用一个实例化的vector\<int\>类型来分析_Tp_alloc_type和pointer这两个类型定义。因为类型pointer依赖类型_Tp_alloc_type，所以先实例化分析类型_Tp_alloc_type。
{% highlight c++ %}
vector<int> v;
{% endhighlight %}
等价于
{% highlight c++ %}
vector<int, std::allocator<int> > v;
{% endhighlight %}
那么实例化的_Tp_alloc_type的定义：
{% highlight c++ %}
typedef typename __gnu_cxx::__alloc_traits<std::allocator<int> >::template rebind<int>::other _Tp_alloc_type;
{% endhighlight %}
__gnu_cxx::__alloc_traits定义在源文件/usr/include/c++/5/ext/alloc_traits.h中：
{% highlight c++ %}
 94 template<typename _Alloc>
 95   struct __alloc_traits
 96 #if __cplusplus >= 201103L
 97   : std::allocator_traits<_Alloc>    <---- since C++11
 98 #endif
 99   {
102     typedef std::allocator_traits<_Alloc>           _Base_type;
......
167     template<typename _Tp>
168       struct rebind
169       { typedef typename _Base_type::template rebind_alloc<_Tp> other; };
......
210   };
{% endhighlight %}
展开后得到：
{% highlight c++ %}
typedef typename std::allocator_traits<std::allocator<int> >::template rebind_alloc<int> _Tp_alloc_type;
{% endhighlight %}
std::allocator_traits定义在源文件/usr/include/c++/5/bits/alloc_traits.h中：
{% highlight c++ %}
 62   template<typename _Alloc, typename _Tp>
 63     struct __alloctr_rebind<_Alloc, _Tp, true>
 64     {
 65       typedef typename _Alloc::template rebind<_Tp>::other __type;
 66     };

 82   template<typename _Alloc>
 83     struct allocator_traits
 84     {
......
199       template<typename _Tp>
200     using rebind_alloc = typename __alloctr_rebind<_Alloc, _Tp>::__type;
......
438     };
{% endhighlight %}
进一步展开得到：
{% highlight c++ %}
typedef typename std::allocator<int>::template rebind<int>::other _Tp_alloc_type;
{% endhighlight %}
std::allocator定义在/usr/include/c++/5/bits/allocator.h：
{% highlight c++ %}
 91   template<typename _Tp>
 92     class allocator: public __allocator_base<_Tp>
 93     {
103       template<typename _Tp1>
104         struct rebind
105         { typedef allocator<_Tp1> other; };
124     };
{% endhighlight %}
最终实例化的_Tp_alloc_type定义为：
{% highlight c++ %}
typedef std::allocator<int> _Tp_alloc_type;
{% endhighlight %}
在实例化的vector\<int, std::allocator\<int\> >定义中，本来就已经知道实例化的std::allocator\<int\>，为什么不直接使用呢？

这要从一系列带模板参数的rebind函数作用说起。这些rebind函数并不是仅仅服务于vector container，而是服务于所有STL container。STL的一些container并不仅仅只管理实例化时使用者指定的类型的内存（即使用者的数据），同时还需要管理它们内部数据结构所需要的内存。

比如std::list container（就是我们常说的双向链表），std::list还需要维护一个内部使用的_List_node数据结构（该_List_node包含指向前一个和后一个_List_node的指针，和使用者的数据）。实例化的std::list\<int, std::allocator\<int\> >只有一个实例化的std::allocator\<int\>，但是std::list内部又需要std::allocator\<\_List\_node\>来管理内存。从上面代码分析过程可以看出，这些一系列带模板参数的rebind函数可以从一个实例化的std::allocator\<_Tp1\>推导得到另一个实例化的std::allocator\<_Tp2\>（当然，\_Tp1和\_Tp2可以是相同的类型，也可以是不同的类型）。

得到_Tp_alloc_type的类型后，那么展开pointer类型定义：
{% highlight c++ %}
typedef typename __gnu_cxx::__alloc_traits<std::allocator<int> >::pointer pointer;
{% endhighlight %}
__alloc_traits::pointer相关的代码：
{% highlight c++ %}
 94 template<typename _Alloc>
 95   struct __alloc_traits
 96 #if __cplusplus >= 201103L
 97   : std::allocator_traits<_Alloc>
 98 #endif
 99   {
......
101 #if __cplusplus >= 201103L
102     typedef std::allocator_traits<_Alloc>           _Base_type;
......
104     typedef typename _Base_type::pointer            pointer;
210   };
{% endhighlight %}
进一步展开得到：
{% highlight c++ %}
typedef typename std::allocator_traits<std::allocator<int> >::pointer pointer;
{% endhighlight %}
std::allocator_traits::pointer相关的代码：
{% highlight c++ %}
 82   template<typename _Alloc>
 83     struct allocator_traits
 84     {
 88       typedef typename _Alloc::value_type value_type; <==> std::allocator<int>::value_type <==> int
 89
 90 #define _GLIBCXX_ALLOC_TR_NESTED_TYPE(_NTYPE, _ALT) \
 91   private: \
 92   template<typename _Tp> \
 93     static typename _Tp::_NTYPE _S_##_NTYPE##_helper(_Tp*); \
 94   static _ALT _S_##_NTYPE##_helper(...); \
 95     typedef decltype(_S_##_NTYPE##_helper((_Alloc*)0)) __##_NTYPE; \
 96   public:
 97
 98 _GLIBCXX_ALLOC_TR_NESTED_TYPE(pointer, value_type*) <==> _GLIBCXX_ALLOC_TR_NESTED_TYPE(pointer, int*)
 99
100       /**
101        * @brief   The allocator's pointer type.
102        *
103        * @c Alloc::pointer if that type exists, otherwise @c value_type*
104       */
105       typedef __pointer pointer;
......
438     };
{% endhighlight %}
接着把宏_GLIBCXX_ALLOC_TR_NESTED_TYPE(pointer, int*)展开：
{% highlight c++ %}
          private:
          template<typename _Tp>
          static typename _Tp::pointer _S_pointer_helper(_Tp*);  # 1 <-----------------+
          static int* _S_pointer_helper(...);                    # 2                   |
                                                                 #                     |
          typedef decltype(_S_pointer_helper((std::allocator<int>*)0)) __pointer; # ---+
          public:
{% endhighlight %}
因为带模板参数的_S_pointer_helper函数更精确匹配，所以可以更进一步展开得到：
{% highlight c++ %}
typedef std::allocator<int>::pointer __pointer; <==> typedef int * __pointer;
{% endhighlight %}
最终得到：
{% highlight c++ %}
typedef int * pointer;
{% endhighlight %}
得到pointer类型后，实例化vector<int>::iterator展开得到：
{% highlight c++ %}
typedef __gnu_cxx::__normal_iterator<int *, vector> iterator;
{% endhighlight %}
__gnu_cxx::__normal_iterator定义在/usr/include/c++/5/bits/stl_iterator.h。__normal_iterator类型比较简单，就是对指针类型进一步封装而已，使得iterator可以像指针一样使用。

同时\_M\_start，\_M\_finish和\_M\_end\_of\_storage均int *类型，下面从vector代码看它们的作用：
{% highlight c++ %}
  71   template<typename _Tp, typename _Alloc>
  72     struct _Vector_base
  73     {
......
 181     private:
 182       void
 183       _M_create_storage(size_t __n)
 184       {
 185     this->_M_impl._M_start = this->_M_allocate(__n);
 186     this->_M_impl._M_finish = this->_M_impl._M_start;
 187     this->_M_impl._M_end_of_storage = this->_M_impl._M_start + __n;
 188       }
 189     };

 213   template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
 214     class vector : protected _Vector_base<_Tp, _Alloc>
 215     {
......
 546       iterator
 547       begin() _GLIBCXX_NOEXCEPT
 548       { return iterator(this->_M_impl._M_start); }
 564       iterator
 565       end() _GLIBCXX_NOEXCEPT
 566       { return iterator(this->_M_impl._M_finish); }
......
 912       void
 913       push_back(const value_type& __x)
 914       {
 915     if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
 916       {
 917         _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
 918                                  __x);
 919         ++this->_M_impl._M_finish;
 920       }
{% endhighlight %}
从上面代码可以看出，\_M\_start指向vector的第一个元素；\_M\_finish指向vector末尾的下一个可用的位置（也就是我们常说的，最后一个元素的下一元素），这也是为什么在遍历时遇到_M_finish时就表示遍历已经结束的原因。

### 3. 遍历结束条件是用!=还是<？
先看下面的测试代码：
{% highlight c++ linenos %}
std::vector<int> v;
for (std::vector<int>::iterator iter = v.begin(); iter < v.end(); ++iter)
{
    // do someting with iter
}
std::list<int> l;
for (std::list<int>::iterator iter = l.begin(); iter < l.end(); ++iter)
{
    // do someting with iter
}
{% endhighlight %}
第7行出现编译error：
{% highlight c++ linenos %}
vector.cc:7:58: error: no match for ‘operator<’ (operand types are ‘std::__cxx11::list<int>::iterator {aka std::_List_iterator<int>}’ and ‘std::__cxx11::list<int>::iterator {aka std::_List_iterator<int>}’)
     for (std::list<int>::iterator iter = l.begin(); iter < l.end(); ++iter)
                                                          ^
{% endhighlight %}
原因是：只有std::vector和std::deque同时支持`<`和`!=`，而其它的container只支持`!=`。

### 4. vector::iterator越界问题
先看下面代码：
{% highlight c++ %}
std::vector<int> v {1, 2, 3};
for (std::vector<int>::iterator iter = v.begin(); iter != v.end(); iter += 2)
{
    std::cout << *iter << " ";
}
{% endhighlight %}
上面代码中，iter永远也不会等于v.end()。

当遍历结束判断条件失效时，会导致遍历无法结束，进而出现访问越界问题，最终导致程序crash。对支持`<`运算符的container可以用`<`代替`!=`；对不支持`<`运算符的container只能使用者自己保证越界问题。