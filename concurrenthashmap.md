# ConcurrentHashMap jdk1.8 



ConcurrentHashMap是java并发编程中十分重要的容器类，它来自java.util.concurrent包。在jdk1.8中这里做了一些更新，比如使用了红黑树优化查询性能，再比如使用到了无锁CAS和synchronized结合的方法，替代了jdk1.7中分段锁的并发解决方案，同时使用到了volatile关键字，对于学习JAVA并发编程来说，是非常值得一看值得参考的源代码。本文以分析jdk1.8下的ConcurrentHashMap为切入点，主要介绍ConcurrentHashMap的核心数据结构、核心读写方法，并由此引申介绍其中用到的算法和技术，如红黑树、transient、volatile关键字、锁技术等等。

ConcurrentHashMap的核心数据结构包括数组、链表和红黑树，和jdk1.7相比最大的区别，使用红黑树提升了查询的性能，避免链表无限增长。

## ConcurrentHashMap核心数据结构



![concurrenthashmap datastructure](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/ds.jpg)

由上图所示，ConcurrentHashMap包含一个有限长度的链表数组，链表在超过阈值时会树化为一颗以当前桶的第一个节点为根节点的红黑树，数组长度超过阈值将进行数组扩容。

代码中的定义方式为

![nodearray](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/nodearray.jpg)

ConcurrentHashMap类内部定义了一个静态类Node<K,V>，表示每一个节点，每一个节点包含当前节点key的哈希值hash、key值、value值、以及指向下一个节点的next指针。其中，value和next都是volatile类型，保证内存可见性。

![node](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/node.jpg)

- **put方法**

![concurrenthashmap put](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/conput.jpg)

1. 对key计算哈希值hashcode
2. 遍历哈希数组
3. 根据哈希值确定桶的位置，判断当前桶为空，CAS自旋写入
4. 判断当前数组需要扩容，扩容数组
5. 当前桶不为空且不需要扩容时，使用synchronized关键字对代码块加锁，写入数据
6. 判断当前节点为红黑树节点，使用红黑树插入的方式写入
7. 如果链表长度大于TREEIFY_THRESHOLD，要将当前链表转换为红黑树

- **get方法**

由于使用了volatile关键字，保证了内存的可见性，get方法无需加锁就可以获取到最新的值

![](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/conget.jpg)

1. 对key计算哈希值hashcode
2. 根据key查找对应的value，find根据e的类型为Node或者TreeNode选择不同的实现，如果是红黑树则使用红黑树查询

TreeNode的find方法

![](https://raw.githubusercontent.com/Titansonia/studyjavawithhappiness/master/pics/treenodeget.jpg)

## transient关键字：不需要被序列化的字段修饰符

使用场景：

1、 类中的字段值可以根据其它字段推导出来，如一个长方形类有三个属性：长度、宽度、面积（示例而已，一般不会这样设计），那么在序列化的时候，面积这个属性就没必要被序列化了；

2、根据业务需要，决定一些字段不需要被序列化。

## 红黑树
红黑树的本质是一颗平衡二叉搜索树
具有二叉搜索树和平衡二叉树的特性
关键在于自平衡
左旋右旋和变色
### 红黑树的定义和性质
红黑树是一种含有红黑结点并能自平衡的二叉查找树。它必须满足下面性质：
* 性质1：每个节点要么是黑色，要么是红色。
* 性质2：根节点是黑色。
* 性质3：每个叶子节点（NIL）是黑色。
* 性质4：每个红色结点的两个子结点一定都是黑色
* 性质5：任意一结点到每个叶子结点的路径都包含数量相同的黑结点


### 红黑树的左旋和右旋
红黑树的左旋相当于以要旋转的节点为中心，将子树整体向左旋转，该节点变成子树的根节点，原来的父节点A变成了左孩子，如果右孩子C有左孩子D，则将其变为A的右孩子；
红黑树的右旋相当于以要旋转的节点为中心，将子树整体向右旋转，该节点变成子树的根节点，原来的父节点A变成右孩子，如果B节点有右孩子，则变为A节点的左孩子。
![](concurrenthashmap/BBDCEE57-CFC6-4875-9A75-FC7D9A0173D8.png)

### 红黑树的插入
分情景考虑
* 情景A：需要插入的节点为根节点，则直接把该节点颜色改为黑色
* 情景B：需要插入的节点其父节点是黑色节点，则不需要调整，因为插入的节点会初始化为红色节点，红色节点不会影响树的平衡；
* 情景C：插入的节点的祖父节点为null，即插入节点的父节点为根节点，直接插入即可（因为根节点为黑色）
* 情景D：插入节点的父节点和祖父节点都存在，并且其父节点是祖父节点的左节点（暗示插入节点的父节点为红色节点）
D1：插入节点的叔叔节点是红色，则将父节点和叔叔节点都改为黑色，然后祖父节点改为红色（祖父节点为根节点则还是为黑色节点），插入节点指向祖父节点PP。处理之后，如果PP的父节点为黑色，则不需要处理，若为红色，则需要以PP作为插入节点继续调整。
![](concurrenthashmap/3185EBC9-E20C-4E20-A0A0-C472E2478BD5.png)
D2：插入节点的叔叔节点是黑色（子节点等于空节点不存在）或者不存在
	D2-1：若插入节点是其父节点的右孩子，则将父节点左旋
![](concurrenthashmap/8F1BE2FC-6912-4B56-ABCD-5E79CF8CF0F1.png)
	D2-2：若插入节点是其父节点的左孩子，将父节点变成黑色节点，祖父节点变成红色节点，祖父节点右旋
![](concurrenthashmap/7173F634-D3D5-44A5-8002-F8B3ED994942.png)

* 情景E
插入节点的父节点和祖父节点都存在，并且其父节点是祖父节点的右节点（暗示插入节点的父节点为红色节点）
E1：插入节点的叔叔节点是红色，则将父亲节点和叔叔节点都改成黑色，然后祖父节点改成红色即可。
![](concurrenthashmap/C50FECAD-107E-4F40-A2B9-66678AB91F39.png)

E2：插入节点的叔叔节点是黑色或不存在：
	E2-1 若插入节点是其父节点的左孩子，则将其父节点右旋
![](concurrenthashmap/3015B29C-DB45-4BEF-9269-993031C87DAB.png)
	E2-2 若为右孩子，则将其父节点变成黑色节点，将其祖父节点变成红色节点，然后将其祖父节点左旋。
![](concurrenthashmap/07FE6526-2747-460E-81D2-5AAE82503868.png)

可以参考以下总结：
![](concurrenthashmap/2392382-fa2b78271263d2c8.png)
红黑树的生长是自底向上的

https://www.cnblogs.com/mfrank/p/9227097.html)

再哈希rehash
使用红黑树查询
使用synchronized替换reentrantlock
谈谈jdk1.8对synchronized的优化

悲观锁和乐观锁

从jdk1.8开始，java对synchronized同步锁关键字做了优化

synchronized优化原理可以参考这一篇，写得很棒 https://aijishu.com/a/1060000000015565

总结一下主要包含两个方面，首先是增加了锁升级机制，升级顺序为偏向锁、轻量级锁、重量级锁，根据场景锁强度逐级递增；另外就是在JVM增加了可以操作锁级别的参数。

读写并发

读读并发

写写并发

锁和CAS



参考资料

[JAVA api doc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html)

[HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！ | crossoverJie’s Blog](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)

30张图带你彻底理解红黑树 - 简书](https://www.jianshu.com/p/e136ec79235c)

[【Java入门提高篇】Day25 史上最详细的HashMap红黑树解析 - 弗兰克的猫 - 博客园](