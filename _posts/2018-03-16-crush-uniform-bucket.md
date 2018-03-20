---
layout: post
title: "CRUSH uniform bucket选择算法"
category: ceph
---
在上一篇[CRUSH map中bucket ID取值范围]({{ site.baseurl }}{% post_url 2018-03-15-bucket-id-in-crush-map %})初步讨论CRUSH map中bucket节点基本概念后，这篇重点讨论bucket节点的选择算法。什么是bucket节点的选择算法？因为bucket节点是由多个子bucket节点或者device叶节点组成，这就涉及到下面问题：
1. 如何选择bucket节点下的一个或者多个子节点呢？  
比如，一个host bucket节点下有多个device节点，如何在这些子device节点中选择一个或者多个呢？
2. 因为bucket节点下的子节点会动态地增加或者减少，如何避免尽量少数据移动和提高算法性能呢？

上面两个问题就是bucket节点的选择算法所要解决的！论文《CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data》把这种bucket节点的选择算法称之为`bucket type`：
>CRUSH defines four different kinds of buckets to represent internal (non-leaf) nodes in the cluster hierarchy: uniform buckets, list buckets, tree buckets, and straw buckets. Each bucket type is based on a different internal data structure and utilizes a different function c(r,x) for pseudo-randomly choosing nested items during the replica placement process, representing a different tradeoff between computation and reorganization efficiency.

在Ceph HAMMER版本，定义了一种新的bucket type——straw2用来替换straw bucket type，本篇文章仅仅讨论uniform bucket。uniform bucket有如下特点：
* 子节点的权重（weight）值必须相同；否则不能使用uniform bucket；
* 可以在常数时间选择一个子节点；同时缺点也很明显：如果有子节点增加或者减少，数据几乎需要完全重组。  
所以，uniform bucket适合子节点很少变动的应用场景。

接下来我们详细讨论uniform bucket的实现。
### 1. uniform bucket的结构体
unifrom bucket的结构体定义在头文件src/crush/crush.h中：
{% highlight c++ %}
struct crush_bucket {
	__s32 id;        /*!< bucket identifier, < 0 and unique within a crush_map */
	__u16 type;      /*!< > 0 bucket type, defined by the caller */
	__u8 alg;        /*!< the item selection ::crush_algorithm */
        /*! @cond INTERNAL */
	__u8 hash;       /* which hash function to use, CRUSH_HASH_* */
	/*! @endcond */
	__u32 weight;    /*!< 16.16 fixed point cumulated children weight */
	__u32 size;      /*!< size of the __items__ array */
        __s32 *items;    /*!< array of children: < 0 are buckets, >= 0 items */
};

/** @ingroup API
 * The weight of each item in the bucket when
 * __h.alg__ == ::CRUSH_BUCKET_UNIFORM.
 */
struct crush_bucket_uniform {
       struct crush_bucket h; /*!< generic bucket information */
	__u32 item_weight;  /*!< 16.16 fixed point weight for each item */
};
{% endhighlight %}
crush_bucket_uniform.item_weight表示crush_bucket_uniform.h.items数组里每个item的weight值，可以看到数组中每个item共享相同的weight值。这也是uniform bucket必须要求子节点的weight值必须一样的原因。在crush_add_uniform_bucket_item()函数中同样能看到这种限制：
{% highlight c++ %}
int crush_add_uniform_bucket_item(struct crush_bucket_uniform *bucket, int item, int weight)
{
	/* In such situation 'CRUSH_BUCKET_UNIFORM', the weight
	   provided for the item should be the same as
	   bucket->item_weight defined with 'crush_make_bucket'. This
	   assumption is enforced by the return value which is always
	   0. */
	if (bucket->item_weight != weight) {
	  return -EINVAL;
	}
......
}
{% endhighlight %}
### 2. uniform bucket的选择算法
在了解uniform bucket存储结构后，接下来分析uniform bucket的选择算法，定义在源文件src/crush/mapper.c的bucket_uniform_choose()函数中：
{% highlight c++ %}
static int bucket_perm_choose(const struct crush_bucket *bucket,
			      struct crush_work_bucket *work,
			      int x, int r)
{
......
}

/* uniform */
static int bucket_uniform_choose(const struct crush_bucket_uniform *bucket,
				 struct crush_work_bucket *work, int x, int r)
{
	return bucket_perm_choose(&bucket->h, work, x, r);
}
{% endhighlight %}
bucket_uniform_choose()函数直接调用bucket_perm_choose()函数，bucket_perm_choose()函数的作用是：
* 对输入x、x的第r个副本（或者为EC编码后第r个fragment，r从0开始），在bucket->items数组中选择一个item来存储x的第r个副本（或者第r个fragment）。
* 参数work是算法的辅助存储空间，保存unifrom bucket的选择算法历史计算结果以便复用，crush_work_bucket结构体定义如下：
{% highlight c++ %}
struct crush_work_bucket {
	__u32 perm_x; /* @x for which *perm is defined */
	__u32 perm_n; /* num elements of *perm that are permuted/defined */
	__u32 *perm;  /* Permutation of the bucket's items */
};
{% endhighlight %}
uniform bucket的选择算法步骤：

**Step 1** 1\. 如果是一个新的输入x（work->perm_x != (__u32)x，work保存历史计算结果），重新初始化work结构体：
{% highlight c++ %}
    work->perm_x = x;
    for (i = 0; i < bucket->size; i++)
        work->perm[i] = i;
    work->perm_n = 0;
{% endhighlight %}
2\. 如果work->perm_x == (__u32)x，直接执行Step 2，复用work历史计算结果。

**Step 2** work->perm[0..bucket->size - 1]任何时候保存着0, 1, .., bucket->size - 1这n个数的一种排列组合（包括初始化时）。  
work->perm_n表示当前work->perm数组中前（work->perm_n - 1）个元素进行了（work->perm_n - 1）次随机排列置换。

1\. 如果输入r >= work->perm_n(r = r % bucket->size)，需要对数组work->perm[work->perm_n..r]这（r - work->perm_n + 1）个元素进行随机排序置换，步骤如下：  
对属于区间[work->perm_n, r]的任何index，将work->perm[index]和在work->perm[index..(bucket->size - 1)]中随机挑选的一个元素互换。

bucket_perm_choose()函数对应的代码如下：
{% highlight c++ %}
unsigned int pr = r % bucket->size;
while (work->perm_n <= pr) {
	unsigned int p = work->perm_n;
	if (p < bucket->size - 1) {
		i = crush_hash32_3(bucket->hash, x, bucket->id, p) %
			(bucket->size - p);                  # ------> 生成一个属于[0, bucket->size - p - 1]区间中的随机数i。
		if (i) {                                 # ------> 那么work->perm[p + i]对应的就是work->perm[p..(bucket->size - 1)]中一个随机元素
			unsigned int t = work->perm[p + i];
			work->perm[p + i] = work->perm[p];   # ------> 将work->perm[p]与work->perm[p + i]值互换
			work->perm[p] = t;
		}
	}
	work->perm_n++;
}
{% endhighlight %}
2\. 如果r < work- perm_n，因为work->perm数组中前(work- perm_n - 1)个元素保存着已经进行过随机排列置换的结果，所以直接执行Step 3；

**Step 3** 返回work->perm[r]。

从上面源代码分析可以得出下面结论：
1. bucket_perm_choose()函数实现中，对r == 0时，有特别优化处理；  
事实上就是Step 2步骤中当work->perm_n == 0，r == 0时执行过程。
2. 使用crush_hash32_x函数来计算得到随机值，并不是我们常用的随机函数，而是一种称为rjenkins1的hash函数，而该crush_hash32_3 hash函数输出仅仅与输入x、bucket->id和work->perm数组中index有关；
3. 因为uniform bucket严重依赖随机产生的0, 1, .. , bucket->size - 1的排列组合，一旦改变bucket->size值（比如，增加或者减少uniform bucket子节点），那么对应的排列组合将很可能完全不同，这同时也意味着所有数据都要重新调整，导致大量数据移动。这也就是为什么uniform bucket不建议增加\减少子节点的原因；
4. 前面提到uniform bucket算法效率比较高是常数时间的！对输入x，只要最多生成bucket->size次随机排列过程，那么就可以直接返回。

参看资料：
* [CRUSH map]

[CRUSH map]: http://docs.ceph.org.cn/rados/operations/crush-map/