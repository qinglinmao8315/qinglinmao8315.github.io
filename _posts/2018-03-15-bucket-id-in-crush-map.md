---
layout: post
title: "CRUSH map中bucket ID取值范围"
category: ceph
---
Ceph使用CRUSH（Controlled Replication Under Scalable Hashing）算法来计算对象的位置信息，而CRUSH算法依赖于CRUSH map。Ceph官方文档对[CRUSH map]的描述：
>CRUSH需要一张集群的地图，且使用CRUSH把数据伪随机地存储、检索于整个集群的OSD里。CRUSH图包含OSD列表、把设备汇聚为物理位置的“桶”列表、和指示CRUSH如何复制存储池里的数据的规则列表。

CRUSH map中包含一个树型或者森林型的层次结构来描述数据中心的OSDs物理拓扑位置，CRUSH rule在这种层次结构基础上定义对象的放置策略。借助于OSDs物理拓扑结构，CRUSH可以保证对象的不同副本或者对象用EC编码后的fragments放置在不同的故障域（比如，不同的主机，不同的机架，等等）来提高可靠性。
引用论文《CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data》对这种层次结构的描述：
> The cluster map is composed of devices and buckets, both of which have numerical identifiers and weight values associated with them.  
Buckets can contain any number of devices or other buckets, allowing them to form interior nodes in a storage hierarchy in which devices are always at the leaves.

CRUSH map的这种层次结构是由device（也可以称之为OSD）和bucket两种不同类型元素组成，不同的device或者bucket由唯一的ID来标识。

device在层次结构中处于叶子节点，那么这些device节点定义格式为：
{% highlight c++ %}
#devices
device {num} {osd.name}
{% endhighlight %}
其中，{num}为从0开始的正自然数。例如：
{% highlight c++ %}
#devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3
{% endhighlight %}
bucket在层次结构中一般表示物理位置，由其它bucket节点或叶子节点组成。为了形象描述bucket在拓扑结构中的物理位置，可以自定义不同type的bucket：
{% highlight c++ %}
type {num} {bucket-name}
{% endhighlight %}
例如：
{% highlight c++ %}
# types
type 0 osd
type 1 host
type 2 rack
{% endhighlight %}
默认情况下，`type 0`代表的是叶子节点，可以根据自己的喜好将其定义为osd、disk等等。根据host或者rack等等这些自定义的bucket可以很形象地描述bucket在数据中心的物理位置。所以，bucket节点定义格式为：
{% highlight c++ %}
[bucket-type] [bucket-name] {
        id [a unique negative numeric ID]
        weight [the relative capacity/capability of the item(s)]
        alg [the bucket type: uniform | list | tree | straw ]
        hash [the hash type: 0 by default]
        item [item-name] weight [weight]
}
{% endhighlight %}
本篇文章主要讨论`id`字段的取值范围，其它字段暂时不展开讨论可以参看官方文档。文档里并没有解释为什么bucket ID必须是“小于0的负自然数”，所以带着这个问题尝试从源代码里找到原因。CRUSH map相关的结构体定义在头文件src/crush/crush.h中，结构体crush_bucket对应的就是bucket节点，从源代码里注释也可以看到bucket ID必须是小于0的自然数：
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
{% endhighlight %}
那么CRUSH map又是如何来组织这些层次结构的bucket节点呢？
{% highlight c++ %}
struct crush_map {
        /*! An array of crush_bucket pointers of size __max_buckets__.
         * An element of the array may be NULL if the bucket was removed with
         * crush_remove_bucket(). The buckets must be added with crush_add_bucket().
         * The bucket found at __buckets[i]__ must have a crush_bucket.id == -1-i.
         */
	struct crush_bucket **buckets;
......
};
{% endhighlight %}
CRUSH实现中并没有采用常见的树这种数据结构，反而只是使用简单的数组来表示。在注释里提到的crush_add_bucket()函数定义在src/crush/builder.c中：
{% highlight c++ %}
int crush_add_bucket(struct crush_map *map,
		     int id,
		     struct crush_bucket *bucket,
		     int *idout)
{
	int pos;

	/* find a bucket id */
	if (id == 0)
		id = crush_get_next_bucket_id(map);
	pos = -1 - id;
......
    /* add it */
	bucket->id = id;
	map->buckets[pos] = bucket;
};
{% endhighlight %}
关于bucket ID可以得出下面的结论:
1. bucket ID指定了该bucket在crush_map.buckets数组中存储位置：-1 - crush_bucket.id，所以需要使用者保证其唯一性和取值范围为小于0的负自然数。
2. 从crush_add_bucket()函数可以看出，crush_map.buckets保存的是所有非叶子bucket节点，那么叶子device节点保存在哪里了？  
在CRUSH map中，bucket节点是由其它子bucket节点或者叶子device节点组成的，存储在crush_bucket.items数组中。  
所以，crush_bucket.items数组中元素分为两类：  
小于0的表示是bucket ID，可以根据该bucket ID在crush_map.buckets找到对应的bucket节点；  
大于等于0的表示是device number，可以根据该device number在之前定义的device列表中找到对应的OSD hostname，有了OSD hostname就可以访问该OSD。  
这就是为什么bucket ID为负自然数的原因。
3. 在CRUSH map中真正可以被访问并存储数据的只有device叶子节点（也就是OSD）；而bucket节点只是方便管理数据中心物理拓扑位置而抽象出来的节点，这些节点并不提供存储服务。

参看资料：
* [CRUSH map]

[CRUSH map]: http://docs.ceph.org.cn/rados/operations/crush-map/