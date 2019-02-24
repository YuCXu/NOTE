# HashMap源码分析（JDK1.8）

- 概述：HashMap 最早出现在 JDK 1.2中，底层基于散列算法实现。HashMap 允许 null 键和 null 值，在计算哈键的哈希值时，null 键哈希值为 0。HashMap 并不保证键值对的顺序，这意味着在进行某些操作后，键值对的顺序可能会发生变化。另外，需要注意的是，HashMap 是非线程安全类，在多线程环境下可能会存在问题。

- 原理：HashMap 底层是基于散列算法实现，散列算法分为散列再探测和拉链式。HashMap 则使用了拉链式的散列算法，并在 JDK 1.8 中引入了红黑树优化过长的链表。数据结构示意图如下： 对于拉链式的散列算法，其数据结构是由数组和链表（或树形结构）组成。在进行增删查等操作时，首先要定位到元素的所在桶的位置，之后再从链表中定位该元素。比如我们要查询下图结构中是否包含元素35，步骤如下： ①定位元素35所处桶的位置，index = 35 % 16 = 3 ②在3号桶所指向的链表中继续查找，发现35在链表中。 上面就是 HashMap 底层数据结构的原理，HashMap 基本操作就是对拉链式散列算法基本操作的一层包装。不同的地方在于 JDK 1.8 中引入了红黑树，底层数据结构由数组+链表变为了数组+链表+红黑树，不过本质没有变。

  ![img](https://img.mubu.com/document_image/38d32db8-9133-453d-b050-40ff4450e419-2159757.jpg)

- 构造方法： HashMap的构造方法有4个。初始化一些重要变量，比如：loadFactor和threshold。而底层的数据结构则是延迟到插入键值对时再进行初始化。 第一个构造方法很简单，仅将 loadFactor 变量设为默认值。构造方法2调用了构造方法3，而构造方法3仍然只是设置了一些变量。构造方法4则是将另一个 Map 中的映射拷贝一份到自己的存储结构中来，这个方法不是很常用。

  ![img](https://img.mubu.com/document_image/d3003840-de49-40d9-a516-4551662548e6-2159757.jpg)

- 属性：①初始容量、②负载因子、③阈值

  - 一般情况下，都会使用无参构造方法创建HashMap。但当我们对时间和空间复杂度有要求的时候，就可以使用手动调参。在 HashMap 构造方法中，可供我们调整的参数有两个，一个是初始容量 initialCapacity，另一个负载因子 loadFactor。通过这两个设定这两个参数，可以进一步影响阈值大小。但初始阈值 threshold 仅由 initialCapacity 经过移位操作计算得出。他们的作用分别如下：

    ![img](https://img.mubu.com/document_image/837a5bcb-64d4-40f5-b067-ae0ee77cbcea-2159757.jpg)

  - 默认情况下，HashMap 初始容量是16，负载因子为 0.75。这里并没有默认阈值，原因是阈值可由容量乘上负载因子计算而来（注释中有说明），即threshold = capacity * loadFactor。观察构造方法3，发现阈值并不是由上面公式计算而来，而是通过一个方法算出来的。

    ![img](https://img.mubu.com/document_image/d30d94cc-6424-4302-a580-199a902fec4e-2159757.jpg)

  - 找到大于或等于cap的最小2的幂

    ![img](https://img.mubu.com/document_image/f30e14c6-df0e-4209-94fb-b2c2cb24d368-2159757.jpg)

  - 对于 HashMap 来说，负载因子是一个很重要的参数，该参数反应了 HashMap 桶数组的使用情况（假设键值对节点均匀分布在桶数组中）。通过调节负载因子，可使 HashMap 时间和空间复杂度上有不同的表现。当我们调低负载因子时，HashMap 所能容纳的键值对数量变少。扩容时，重新将键值对存储新的桶数组里，键的键之间产生的碰撞会下降，链表长度变短。此时，HashMap 的增删改查等操作的效率将会变高，这里是典型的拿空间换时间。相反，如果增加负载因子（负载因子可以大于1），HashMap 所能容纳的键值对数量变多，空间利用率高，但碰撞率也高。这意味着链表长度变长，效率也随之降低，这种情况是拿时间换空间。

- 其他方法：

  - 查找：先定位键值对所在桶的位置，然后再对链表或者红黑树进行查找。

    ![img](https://img.mubu.com/document_image/87e88c4b-2b3f-4b92-a3ee-c253997c04c8-2159757.jpg)

    - 先分析查找过程的第一步，确定桶的位置，实现代码如下，这里通过(n - 1)& hash即可算出桶的在桶数组中的位置。HashMap 中桶数组的大小 length 总是2的幂，此时，(n - 1) & hash 等价于对 length 取余。但取余的计算效率没有位运算高，所以(n - 1) & hash也是一个小的优化。

      ![img](https://img.mubu.com/document_image/b40af587-a030-4df9-9132-e1d2b1f5d469-2159757.jpg)

    - 其中有一个计算hash的方法，这个方法的源码如下，为什么不直接用键的hashCode方法产生hash呢？

      ![img](https://img.mubu.com/document_image/c3508082-7815-4613-9dc6-f89cc2979af2-2159757.jpg)

    - 看一下下面求余的计算图，计算余数时，由于 n 比较小，hash 只有低4位参与了计算，高位的计算可以认为是无效的。这样导致了计算结果只与低位信息有关，高位数据没发挥作用。为了处理这个缺陷，我们可以上图中的 hash 高4位数据与低4位数据进行异或运算，即 hash ^ (hash >>> 4)。通过这种方式，让高位数据与低位数据进行异或，以此加大低位信息的随机性，变相的让高位数据参与到计算中。此时的计算过程如下：

      ![img](https://img.mubu.com/document_image/d793bbb0-7e77-4704-8aa6-114292fda996-2159757.jpg)

    - 在 Java 中，hashCode 方法产生的 hash 是 int 类型，32 位宽。前16位为高位，后16位为低位，所以要右移16位。上面所说的是重新计算 hash 的一个好处，除此之外，重新计算 hash 的另一个好处是可以增加 hash 的复杂度。当我们覆写 hashCode 方法时，可能会写出分布性不佳的 hashCode 方法，进而导致 hash 的冲突率比较高。通过移位和异或运算，可以让 hash 变得更复杂，进而影响 hash 的分布性。

  - 遍历：对于遍历HashMap，一般采用下面的方式：

    ![img](https://img.mubu.com/document_image/12a99a01-2e88-4f03-b349-d74a194fefd6-2159757.jpg)

    - 从上面代码片段中可以看出，一般都是对 HashMap 的 key 集合或 Entry 集合进行遍历。上面代码片段中用 foreach 遍历 keySet 方法产生的集合，在编译时会转换成用迭代器遍历，等价于：

      ![img](https://img.mubu.com/document_image/23491642-91c3-4e80-959e-4948b2324c31-2159757.jpg)

    - 在遍历 HashMap 的过程中会发现，多次对 HashMap 进行遍历时，遍历结果顺序都是一致的。但这个顺序和插入的顺序一般都是不一致的。产生上述行为的原因是怎样的呢？

      ![img](https://img.mubu.com/document_image/50e27051-8701-407d-84bd-f55e91422c9d-2159757.jpg)

      ![img](https://img.mubu.com/document_image/545f55c7-14a8-44fa-91ed-6e3bfc60f2ee-2159757.jpg)

    - 如上面的源码，遍历所有的键时，首先要获取键集合KeySet对象，然后再通过 KeySet 的迭代器KeyIterator进行遍历。KeyIterator 类继承自HashIterator类，核心逻辑也封装在 HashIterator 类中。HashIterator 的逻辑并不复杂，在初始化时，HashIterator 先从桶数组中找到包含链表节点引用的桶。然后对这个桶指向的链表进行遍历。遍历完成后，再继续寻找下一个包含链表节点引用的桶，找到继续遍历。找不到，则结束遍历。举个例子，假设我们遍历下图的结构：

      ![img](https://img.mubu.com/document_image/48dd859d-1ef3-4f65-a647-69e0625b1b16-2159757.jpg)

    - HashIterator 在初始化时，会先遍历桶数组，找到包含链表节点引用的桶，对应图中就是3号桶。随后由 nextNode 方法遍历该桶所指向的链表。遍历完3号桶后，nextNode 方法继续寻找下一个不为空的桶，对应图中的7号桶。之后流程和上面类似，直至遍历完最后一个桶。以上就是 HashIterator 的核心逻辑的流程，对应下图：

      ![img](https://img.mubu.com/document_image/090e4190-88ee-4394-92ba-70e974cea25d-2159757.jpg)

  - 插入：首先肯定是先定位要插入的键值对属于哪个桶，定位到桶后，再判断桶是否为空。如果为空，则将键值对存入即可。如果不为空，则需将键值对接在链表最后一个位置，或者更新键值对。注意！！！！：首先 HashMap 是变长集合，所以需要考虑扩容的问题。其次，在 JDK 1.8 中，HashMap 引入了红黑树优化过长链表，这里还要考虑多长的链表需要进行优化，优化过程又是怎样的问题。

    ![img](https://img.mubu.com/document_image/f5dc7ac2-2908-43f1-a6e0-8c1ee27c93d2-2159757.jpg)

    ![img](https://img.mubu.com/document_image/73356e8e-7d88-498b-870b-93b725b14080-2159757.jpg)

    - 插入操作的入口方法是 put(K,V)，但核心逻辑在V putVal(int, K, V, boolean) 方法中。putVal 方法主要做了这么几件事情：①当桶数组 table 为空时，通过扩容的方式初始化table；②查找要插入的键值对是否已经存在，存在的话根据条件判断是否用新值替换旧值；③如果不存在，则将键值对链入链表中，并根据链表长度决定是否将链表转为红黑树；④判断键值对数量是否大于阈值，大于的话则进行扩容操作。

  - 扩容机制：java集合框架已经实现了变长的数据结构，比如：ArrayList和HashMap。 在 HashMap 中，桶数组的长度均是2的幂，阈值大小为桶数组长度与负载因子的乘积。当 HashMap 中的键值对数量超过阈值时，进行扩容。

    - HashMap 的扩容机制与其他变长集合的套路不太一样，HashMap 按当前桶数组长度的2倍进行扩容，阈值也变为原来的2倍（如果计算过程中，阈值溢出归零，则按阈值公式重新计算）。扩容之后，要重新计算键值对的位置，并把它们移动到合适的位置上去。

      ![img](https://img.mubu.com/document_image/1233482e-48de-4e90-8210-c89e17a46209-2159757.jpg)

      ![img](https://img.mubu.com/document_image/5b5832d7-00d0-41ba-9126-edb9f3441b4f-2159757.jpg)

      ![img](https://img.mubu.com/document_image/2646a027-e4bc-4559-8fce-489aecbb5a76-2159757.jpg)

    - 上面源码总共做了3件事，分别是：

      - ①计算新桶数组的容量newCap和新阈值newThr
      - ②根据计算出的 newCap 创建新的桶数组，桶数组 table 也是在这里进行初始化的
      - ③将键值对节点重新映射到新的桶数组里。如果节点是 TreeNode 类型，则需要拆分红黑树。如果是普通节点，则节点按原顺序进行分组。

    - 接下来说说第一点和第三点，newCap和newThr计算过程。该计算过程对应resize源码的第一和第二个条件分支：

      ![img](https://img.mubu.com/document_image/3fd241b6-fbce-46e1-af63-27069c5b3ea8-2159757.jpg)

    - 通过这两个条件对不同情况进行判断，进而算出不容的容量值和阈值。

    - 分支一：

      ![img](https://img.mubu.com/document_image/7f1bb98c-3708-47f9-960c-d56b31dd32fc-2159757.jpg)

    - 这里把oldThr > 0情况单独拿出来说一下。在这种情况下，会将 oldThr 赋值给 newCap，等价于newCap = threshold = tableSizeFor(initialCapacity)。我们在初始化时传入的 initialCapacity 参数经过 threshold 中转最终赋值给了 newCap。

    - 嵌套分支：

      ![img](https://img.mubu.com/document_image/763388f1-a2da-43c2-aa54-4a8d09d9527d-2159757.jpg)

    - 分支二：

      ![img](https://img.mubu.com/document_image/14789a69-afa8-4ce4-a4a9-5c38f4b89f2a-2159757.jpg)

    - 在JDK1.8中，重新映射节点需要考虑节点类型。对于树形节点，需要先拆分红黑树再映射。对于链表类型节点，则需先对链表进行分组，然后再映射。需要注意的是，分组后，组内节点的相对位置保持不变。

    - 接下来分析链表怎么分组映射的，我们都知道往底层数据结构中插入节点时，一般都是先通过模运算计算桶位置，接着把节点放入桶中即可。事实上，我们可以把重新映射看做插入操作。在 JDK 1.7 中，也确实是这样做的。但在 JDK 1.8 中，则对这个过程进行了一定的优化，逻辑上要稍微复杂一些。在详细分析前，我们先来回顾一下 hash 求余的过程：

      ![img](https://img.mubu.com/document_image/187373c4-a299-42bc-8820-0aeca672a585-2159757.jpg)

    - 上图中，桶数组大小n=16，hash1与hash2不相等。但因为只有后4位参与求余，所以结果相等。当桶数组扩容后，n由16变成了32，对上面的hash值重新进行了映射：

      ![img](https://img.mubu.com/document_image/5aa3d543-b9d7-4e74-812d-925f9628fc2e-2159757.jpg)

    - 扩容后，参与模运算的位数由4位变为了5位。由于两个 hash 第5位的值是不一样，所以两个 hash 算出的结果也不一样。

      ![img](https://img.mubu.com/document_image/3f697df1-e067-4f98-a0f1-fc52372a5d01-2159757.jpg)

    - 假设我们上图的桶数组进行扩容，扩容后容量 n = 16，重新映射过程如下，依次遍历链表，并计算节点hash&oldCap的值：

      ![img](https://img.mubu.com/document_image/526f0ff0-172e-4f86-b148-7ed17e5d3dd3-2159757.jpg)

    - 如果值为0，将 loHead 和 loTail 指向这个节点。如果后面还有节点 hash & oldCap 为0的话，则将节点链入 loHead 指向的链表中，并将 loTail 指向该节点。如果值为非0的话，则让 hiHead 和 hiTail 指向该节点。完成遍历后，可能会得到两条链表，此时就完成了链表分组：

      ![img](https://img.mubu.com/document_image/c1e61522-767f-4349-bca4-195f077f29ec-2159757.jpg)

    - 最后再将这两条链接存放到相应的桶中，完成扩容。如下图，从图中可以发现，重新映射后，两条链表中的节点顺序并未发生变化，还是保持了扩容前的顺序。

      ![img](https://img.mubu.com/document_image/8b7273e5-0bce-46b0-92d9-0a31aab3d650-2159757.jpg)

  - 链表的树化、红黑树链化与拆分：

    - JDK 1.8对HashMap实现进行了改进，最大的改进莫过于引入了红黑树处理频繁的碰撞，代码的复杂度也随之上升。

      ![img](https://img.mubu.com/document_image/5fe6870f-ec36-4390-8f4f-80c08fd2c003-2159757.jpg)

      ![img](https://img.mubu.com/document_image/9349526b-ffb4-4032-b200-86a43801ee99-2159757.jpg)

    - 在扩容过程中，树化要满足两个条件：

      - ①链表长度大于等于 TREEIFY_THRESHOLD
      - ②桶数组容量大于等于 MIN_TREEIFY_CAPACITY

    - 引入第二个条件原因：当桶数组容量较小时，键值对节点hash的碰撞率可能会比较高，进而导致链表长度较长。这时候应该优先扩容，而不是立马树化。毕竟高碰撞率是因为桶数组容量较小引起的，这个是主因。容量小时，优先扩容可以避免一些列的不必要的树化过程。同时，桶容量较小时，扩容会比较频繁，扩容时需要拆分红黑树并重新映射。

    - treeifyBin方法，这个方法主要的作用是将普通链表转成由TreeNode型节点组成的链表，最后调用treeify是将该链表转为红黑树。TreeNode继承自Node类，所以 TreeNode 仍然包含 next 引用，原链表的节点顺序最终通过 next 引用被保存下来。我们假设树化前，链表结构如下：

      ![img](https://img.mubu.com/document_image/45ac292e-225f-495c-8c2b-339076bff86d-2159757.jpg)

    - HashMap 在设计之初，并没有考虑到以后会引入红黑树进行优化。所以并没有像 TreeMap 那样，要求键类实现 comparable 接口或提供相应的比较器。但由于树化过程需要比较两个键对象的大小，在键类没有实现 comparable 接口的情况下，怎么比较键与键之间的大小了就成了一个棘手的问题。为了解决这个问题，HashMap 是做了三步处理，确保可以比较出两个键的大小，如下：

      - ①比较键与键之间 hash 的大小，如果 hash 相同，继续往下比较
      - ②检测键类是否实现了 Comparable 接口，如果实现调用 compareTo 方法进行比较
      - ③如果仍未比较出大小，就需要进行仲裁了，仲裁方法为 tieBreakOrder

    - 通过上面三次比较，最终就可以比较出孰大孰小。比较出大小后就可以构造红黑树了，最终构造出的红黑树如下：

      ![img](https://img.mubu.com/document_image/07297e35-443c-4ec6-9507-19ba98606550-2159757.jpg)

    - 橙色的箭头表示 TreeNode 的 next 引用。由于空间有限，prev 引用未画出。可以看出，链表转成红黑树后，原链表的顺序仍然会被引用仍被保留了（红黑树的根节点会被移动到链表的第一位），我们仍然可以按遍历链表的方式去遍历上面的红黑树。

  - 红黑树的拆分：如上节所说，在将普通链表转成红黑树时，HashMap 通过两个额外的引用 next 和 prev 保留了原链表的节点顺序。这样再对红黑树进行重新映射时，完全可以按照映射链表的方式进行。这样就避免了将红黑树转成链表后再进行映射，无形中提高了效率。

    ![img](https://img.mubu.com/document_image/8e3cec8e-2df9-45bd-9944-3392970963e2-2159757.jpg)

    ![img](https://img.mubu.com/document_image/f31990ad-74f4-46fd-9f1e-7233c743abbb-2159757.jpg)

    - 从源码上可以看得出，重新映射红黑树的逻辑和重新映射链表的逻辑基本一致。不同的地方在于，重新映射后，会将红黑树拆分成两条由 TreeNode 组成的链表。如果链表长度小于 UNTREEIFY_THRESHOLD，则将链表转换成普通链表。否则根据条件重新将 TreeNode 链表树化。举个例子说明一下，假设扩容后，重新映射上图的红黑树，映射结果如下：

      ![img](https://img.mubu.com/document_image/85cd5a03-9e2b-42f6-80d1-2f8405fac729-2159757.jpg)

    - 红黑树链化：红黑树中仍然保留了原链表节点顺序。有了这个前提，再将红黑树转成链表就简单多了，仅需将 TreeNode 链表转成 Node 类型的链表即可。相关代码如下：

      ![img](https://img.mubu.com/document_image/84905536-c1ac-4b99-a9ac-8316d3116760-2159757.jpg)

- 细节问题：

  - 被 transient 所修饰 table 变量：在 Java 中，被该关键字修饰的变量不会被默认的序列化机制序列化。桶数组 table 是 HashMap 底层重要的数据结构，不序列化的话，别人还怎么还原呢？
  - HashMap 并没有使用默认的序列化机制，而是通过实现readObject/writeObject两个方法自定义了序列化的内容。这样做是有原因的，试问一句，HashMap 中存储的内容是什么？不用说，大家也知道是键值对。所以只要我们把键值对序列化了，我们就可以根据键值对数据重建 HashMap。有的朋友可能会想，序列化 table 不是可以一步到位，后面直接还原不就行了吗？这样一想，倒也是合理。但序列化 talbe 存在着两个问题：
    - ①table 多数情况下是无法被存满的，序列化未使用的部分，浪费空间
    - ②同一个键值对在不同 JVM 下，所处的桶位置可能是不同的，在不同的 JVM 下反序列化 table 可能会发生错误。
  - 第二个问题解释一下。HashMap 的get/put/remove等方法第一步就是根据 hash 找到键所在的桶位置，但如果键没有覆写 hashCode 方法，计算 hash 时最终调用 Object 中的 hashCode 方法。但 Object 中的 hashCode 方法是native 型的，不同的 JVM 下，可能会有不同的实现，产生的 hash 可能也是不一样的。也就是说同一个键在不同平台下可能会产生不同的 hash，此时再对在同一个 table 继续操作，就会出现问题。