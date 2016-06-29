

## ATS Cache 架构

### 简介
Apache Traffic Server不仅是HTTP代理服务器, 还是HTTP缓存服务器。Traffic Server可以缓存任何字节流数据, 不过目前仅支持通过HTTP协议来传输这些字节流数据。当这样的一个字节流被缓存住时(还有相关的HTTP协议头)我们称之为缓存中的一个对象object。每个对象可以通过全局唯一的缓存key来定位。

这篇文档旨在对Traffic Server缓存系统的基本框架和实现细节进行一下描述。为了帮助理解cache系统的内部机制, 我们也会对cache系统的配置做一些简单的讨论。这篇文档对从事TrafficServer核心开发以及插件开发(codebase or plugins)的人员都会很有帮助。这里假定读者已经熟悉了《管理员指南》这部分内容，特别是Http反向代理缓存、缓存配置以及相关的配置文件和具体的取值。

遗憾的是内部的一些术语并不是特别一致, 因此为了试图创造一些一致性，这篇文档会经常以不同的形式来使用这些术语。

### Cache的布局
接下来的章节会介绍持久化下来的cache数据(persistent cache data)是如何组织的。Traffic Server将其持久化存储设备看成常规字节的集合, 假定存储设备上没有其他结构。而且Traffic Server不会使用操作系统上面的文件系统功能, 即便是使用一个文件,那也仅仅是用来标识出一个字节集合。

#### Cache存储
Traffic Server使用的裸存储设备定义在配置文件storage.config中。文件中的每一行定义了一个具有一致性存储特征的`cache设备`(cache span)。

![Two cache spans](https://docs.trafficserver.apache.org/en/latest/_images/cache-spans.png)

Traffic Server的管理员可以根据实际情况将存储空间组织成一系列分卷，这些分卷定义在volume.config配置文件中, `cache分卷`(cache volumes)是管理存储配置的基本单位。

`Cache分卷`的容量可以通过存储百分比来定义, 也可以定义为一个绝对的数值。默认情况下，每一个`cache分卷`定义的存储空间都会分散到所有的`cache设备`中，这是出于健壮性的考虑(一个盘有问题不会影响这个`cache分卷`在其他盘上的存储空间)。`cache分卷`和`cache设备`的交集是`cache带`(cache stripe)，每个`cache设备`会被切分成若干个`cache带`, 而每个`cache分卷`是由一系列来自不同的`cache设备`中的`cache带`组成。

如果`Cache分卷`按下面这样定义:

![volumes](https://docs.trafficserver.apache.org/en/latest/_images/ats-cache-volume-definition.png )

那么对于前面定义的`Cache设备`的实际布局将会如下所示:

![layout](https://docs.trafficserver.apache.org/en/latest/_images/cache-span-layout.png)

`Cache带`是cache设计实现过程中的最基本的单位。一个缓存对象会被完整的存储在单一的`Cache带`中，因此也就存储在单一的`Cache设备中`，对象从不跨`Cache设备`或跨`Cache分卷`存储。每一个对象会通过回源的URL来计算一个hash值，然后得到对应的`Cache分卷`。可以通过配置hosting.config文件来指定那些域名的数据存储在哪些`Cache分卷`中。此外，从4.0.1版本开始可以指定一个`Cache分卷`包含哪些`Cache设备`(也就指定了`Cache带`)。

traffic_server进程启动的时候会根据storage.config和cache.config配置文件来计算`Cache设备`, `Cache分卷`, `Cache带`的布局和结构，因此对这些文件的修改会导致对原有cache数据的全部重新校验。

#### Cache带数据结构
Traffic Server将一个`Cache带`代表的存储区域看做一个一致性的字节集合，而在内部会对每一个`Cache带`进行独立地对待。这一节描述的数据结构对于每一个`Cache带`都是一样的。在代码中会用Vol类来代表`Cache带`, 用CacheVol来代表`Cache分卷`，一个`Cache分卷`由位于所有不同设备上的`Cache带`组成。

    在对一个对象进行操作之前必须先明确这个对象所在的Cache带, 因为每个Cache带分别拥有着独立的索引空间。如果缓存对象所在的Cache带发生变化的话那么这个缓存对象也将失效, 因为在新的Cache带中并没有这个缓存对象的索引。


#### Cache索引
`Cache带`中存储的内容是通过索引(directory)来进行定位的，索引中的每一个元素我们称之为目录项(directory entry)，代码中使用`Dir`来表示。每一个目录项代表了cache中一段连续的存储空间。这里会涉及到各种概念包括分片(fragment)、分段(segment)、文档(doc, ducument)等。本文会使用"分片"Fragment这个术语，这也是代码中最常用到的概念。"文档"Doc这个术语用来表示一个分片的头部数据。目录项Dir被视为通过Cache ID做为key而计算得到的哈希值。查看《索引探测》这一节可以看到如何通过Cache ID来定位一个目录项。默认情况下Cache ID是通过对象的URL来计算得到。

索引数据会被持久化在内存中，这也意味着目录项必须足够的小(目前只有10个字节)，这也将导致可存储的信息不够多。从另外一个方面来考虑，绝大多数的cache miss情况是不需要任何的磁盘I/O操作的，这是一个很大的性能收益。

此外，当一个`Cache带`初始化之后，那么它对应的索引空间大小也就明确下来，而且不会再改变。索引空间的大小和`Cache带`的大小相关(近似线性)，因此Traffic Server的内存占用(footprint)也会和磁盘大小相关。由于索引空间的总大小是不变的，也就意味着占用的内存大小也是固定的，因此当Traffic Server在cache中存储更多的对象的时候并不会消耗额外的内存。如果有足够的内存保证Traffic Server在空白存储的情况下正常运行，那么在cache存满的情况下仍然可以正常运行。

![Dir](https://docs.trafficserver.apache.org/en/latest/_images/cache-directory-structure.png)

每一个目录项中都保存了在`Cache带`中的偏移量(offset)和大小(size)，目录项中存储的大小是一个能够包含分片中实际数据大小的粗略值，实际的大小保存在分片的头部区域(fragment header)中(在磁盘上)。

    只有通过读取磁盘才能得到HTTP头部保存的数据，这部分数据中保存着对象的原始URL, 以及cache key。

索引空间是通过哈希表来组织的，以链表的方式来解决冲突。由于每个目录项很小，因此目录项会被直接作为哈希桶(hash bucket)的链表头。

通过对索引中的所有项进行分组的方式(grouping structure)来实现链表，第一层的分组就是索引桶(directory bucket)，包含了固定数目(目前是4)的目录项。每个索引桶中的第一个目录项将作为这个哈希桶的根。

    The term "bucket" is used in the code to mean both the conceptual bucket for hashing and for a structural grouping mechanism in the directory and so these will be qualified as needed to distinguish them. The unqualified term "bucket" is almost always used to mean the structural grouping in the directory.


多个索引桶会集成为段(segment)，一个`Cache带`中的所有段拥有相同数目的索引桶。在计算一个`Cache带`有多少个段时，要保证每个段要拥有尽可能多的索引桶数目，同时要保证一个段拥有的目录项个数不能超过65535(即2**16-1)。

![segment](https://docs.trafficserver.apache.org/en/latest/_images/dir-segment-bucket.png)

同一个段中的每个目录项以链表的形式组织起来，目录项会体现出向前和向后的索引。由于一个段中的目录项不会超过65535，因此16位足以表示出索引值。在`Cache带`的头部会保存一个目录项索引(entry indices)的数组，数组的每一项是对应段中的空闲目录项链表(entry free list)的链表头。使用中的目录项使用目录桶结构存放.当一个`Cache带`初始化的时候, 每个目录桶中的第一个目录项会被清零(标示为未使用)，而所有的其他项会被放入`Cache带`头部对应的段空闲链表(segment free list)中。这就意味着每个目录桶(directory bucket)中的第一项会被当做哈希桶(hash bucket)的第一项，它们不会被放入空闲列表中，而是会被清零。目录桶中的其他项最好添加到对应的哈希桶中，但不是强制的。每个段中的空闲目录项链表在初始化的时候会让每个目录桶中的其他项顺序的添加进来，先是每个目录桶中的第二项，然后是第三项、第四项。由于空闲链表采用的是先进先出(FIFO)的策略，所以在选择的时候会先选择所有目录桶的第四项，然后才是第三项，以此类推。当需要从一个目录桶中分配出一个新的目录项时，会从第一项到最后一项顺序查找，这样可以让目录桶中的目录项尽可能的本地化(bucket locality,通过Cache ID计算得到的哈希桶会尽量选择同一个目录桶中的目录项)。

    目录桶是存储格式上的划分，每个桶中有4项；而哈希桶则是查找时使用的数据结构，每个哈希桶中的所有冲突项以链表的形式组织。
    计算时哈希桶和目录桶是一一对应的，但是哈希桶中的冲突项可能会多于4，因此这个时候会将其他目录桶中的空闲项拿来用连到本哈希桶的链表中。
    哈希桶在组织链表时会优先选择本哈希桶对应的目录桶中的这4个目录项，然后才会去使用其他目录桶中的空闲项。

![hash](https://docs.trafficserver.apache.org/en/latest/_images/dir-bucket-assign.png)

一个在使用中的目录项会从空闲链表中移除，当这个目录项不再使用时会重新回到空闲链表。当需要把一个分片设置对应的目录项时，会通过cache ID来定位所在的哈希桶(也会拿来定位所在的段和目录桶)，如果对应的目录桶中的第一项并未使用，那么会直接拿来给这个分片使用，否则会查看一下这个目录桶中的其他项，如果有空闲则会拿来使用。如果还是找不到空闲项，将会使用空闲链表中的第一项，这一项会通过相同的next和previous索引被链到哈希桶的冲突链表中，以确保可以通过cache ID来查找到。


#### 存储布局
存储布局(storage layout)指的是`Cache带`的元信息和存放的数据内容，元信息由三部分组成 - 头部、索引数据、尾部。`Cache带`的元信息存储了两份，头部和尾部在代码中使用的是相同的数据结构VolHeaderFooter，这个数据结构的尾部包含一个可变长度的数组，这个数组用来保存每个段的空闲目录链表的表头，每一项包含对应段中空闲链表的第一项的索引，尾部其实是头部的拷贝，但不包含每个段的空闲链表数组。因此头部的大小会受目录项大小影响，但是尾部不会。

![layout](https://docs.trafficserver.apache.org/en/latest/_images/cache-stripe-layout.png)

每一个`Cache带`包含以下几个能够描述其基本布局的变量：

**skip**

`Cache带`数据的开始位置，物理磁盘上最开始的一段数据会被保留下来，这样可以避免对操作系统造成一些干扰，或者这个值也代表了其他的`Cache带`在`Cache设备`上的偏移量。

**start**

`Cache带`元信息之后的数据区在磁盘上的偏移量。

**length**

`Cache`带的字节大小，Vol::len。

**data length**

`Cache带`上内容区的总块数(512字节为一块)，Vol::data_blocks。

    这里必须要留意代码中提到的长度和大小这些词，因为在不同的地方会分别用到三种不同的单位(字节，cache块cache blocks，存储块storage blocks)。

索引区的总大小(目录项的个数)的计算方法是用`Cache带`的大小除以平均对象大小来得到，索引区会消耗等量大小的内存，如果cache存储变大那么Traffic Server消耗的内存也就越多，平均对象大小默认是8000字节，可以通过`proxy.config.cache.min_average_object_size`来配置。增加平均对象大小会减少索引区对内存的占用(memory footprint)，同时也意味着cache中能够存储的不同对象的数量也会减少。


磁盘上的内容区域保存真正的对象数据，内容区域会被当做一个环形的缓冲区(circular buffer)，新对象会覆盖掉最早cache下来的对象。新的缓存对象在`Cache带`中写到的位置被称为写光标(write cursor)，这意味着写光标到达的区域原来所保存的对象将会被淘汰，即便这些对象还没有过期。当一个在磁盘上的对象被覆盖时，并不会立即的检测到，因为索引并没有做更新，而是在后面读取对象分片的时候才会检测到失败。

![cursor](https://docs.trafficserver.apache.org/en/latest/_images/ats-cache-write-cursor.png)

    磁盘上的cache数据从来不会更新

这是一个需要特别注意的事情，更新操作(比如对过期对象进行刷新并收到304响应)实际上就是将对象的新拷贝写到写光标的位置。对象的原有内容将在原来的磁盘位置不变， 当写光标经过该位置时，将会改写它。当`Cache带`中的索引发生更新操作时(内存中)，cache上的原有分片数据将失效，这也是比较常见的存储管理手段。当需要从cache中删除一个对象时，只需要更新一下索引即可，不需要其他的操作，特别是不需要任何的I/O操作。

#### 对象数据结构
每个对象会存储两种类型的数据，元数据(metadata)和内容数据(content data)。元数据包括对象的HTTP header和描述信息，而内容数据包含对象的真正内容，是发送给客户端的字节流。

cache中的对象用Doc这个数据结构来表示，Doc可以认为是分片的头部数据，而且会存储在每个分片的开始位置(对象的每个分片都是一个Doc)。对象的第一个分片被称为`first Doc`并且会保存有对象的元数据，**任何对一个对象的操作都需要先读取这第一个分片**。分片的定位方法是将对象的cache key转换为cacheID然后通过这个cacheID来查找对象的目录项，目录项中保存了对象第一个分片在磁盘上的偏移量和近似大小，然后就会从磁盘上读取出来。对象的第一个分片会包含对象的请求头和响应头以及对象的所有描述属性(比如content length)。

![firstDoc](http://img.blog.csdn.net/20150423184708569)

Traffic Server支持对象内容多样化，也称之为多副本(alternates)。所有副本的全部元信息都会保存在对象的第一个分片中，包括副本的集合和每个副本的HTTP header信息。因此当从磁盘上读取出对象的`first Doc`之后就可以做副本选择(alternate selection)。**如果一个对象拥有多个副本，那么每个副本会独立地分别存放在其他分片中。如果对象只有一个副本，那么对象的内容有可能和元信息同时存放在第一个分片中。每个副本的内容都会对应一个目录项，而每个副本目录项的查找key都会保存在第一片中的元信息中**。

在4.0.1版本之前，header数据会保存在CacheHTTPInfoVector这个结构中，这个结构会被序列化之后存在磁盘上，在这个结构的后面会保存和对象其他分片相关的一些附加信息，因而是变长的。

![CacheHTTPInfoVector](https://docs.trafficserver.apache.org/en/latest/_images/cache-doc-layout-3-2-0.png)

这样存在一个问题，如果一个对象有多个副本，那么只有一个分片table是不够的。因此在元数据中不再单独保存变长的分片信息，而是将分片信息合并到CacheHTTPInfoVector结构中，这样就产生了下面的格式：

![CacheHTTPInfoVector-4.0.1](https://docs.trafficserver.apache.org/en/latest/_images/cache-doc-layout-4-0-1.png)

向量中的每一个元素代表了一个副本，包含的信息有HTTP header、分片表(假如存在)和一个cache key，这个cache key对应的目录项用来定位对象的earliest Doc`，这也是该副本的开始分片的索引。

当一个对象最开始被缓存的时候，它只会有一个副本，因此内容也会同时保存在`first Doc`中(如果内容不大的话)，在代码中称之为常驻副本(resident alternate)，这只会在对象最初被保存的时候出现。如果元数据发生改变(比如发送If-Modified-Since请求之后接收到了304响应)，那么对象的内容数据会被保留在原始分片中，但是会用新的分片来保存对象的`first Doc`(对象内容过小的情况除外)，这样对象不会再有常驻副本，这里提到的过小是要小于配置中指定的`proxy.config.cache.alt_rewrite_max_size`的值，默认4096。

    CacheHTTPInfoVector只会保存在`first Doc`中，包括`earliest Doc`在内的其他Doc中的hlen的值应该是0，否则会被忽略。

大对象会被切分成多个分片在cache中存储下来，如果文档的总长度比`first Doc`或`earliest Doc`大的话就一定是存在分片，在这种情况下会保存一个分片偏移量表(fragment offset table)，用来保存每个分片的第一个字节相对于对象初始字节的偏移量(很显然第一个分片的偏移量总是0)。这样在对大文件处理range请求时会更加的高效，因为range请求中覆盖不到的数据分片会被跳过。序列中的最后一个分片通过接近于对象总长度的分片大小和偏移量来检测，因为没有明确的结尾标志。每个分片是通过序列中的前一片计算得出，第N片的计算公式如下：

    key_for_N_plus_one = next_key(key_for_N);

这里`next_key`是一个全局的函数，用来通过一个已知的cache key确定性的计算得出一个新的cache key。

如果一个对象存在多个分片，那么在保存的时候会首先写包括`earliest Doc`的数据分片，最后再保存`first Doc`。在从磁盘上读取时，会同时校验`first Doc`和`earliest Doc`(确保对象并没有被写光标覆盖)来确保磁盘上保存有完整的对象(这两个Doc将其他Doc夹在中间，所以如果这两个Doc有效的话，整个对象的数据就是有效的)。一个对象的所有分片是排好序的，但这些分片不一定在磁盘上是连续存储的，因为Traffic Server会交替的接收不同对象的数据。

![multi-alternate and multi-fragment object storage](https://docs.trafficserver.apache.org/en/latest/_images/cache-multi-fragment.png)

如果一个对象在cache中被标识为`pinned`的话，那么这个对象在磁盘上的数据就不可以被覆盖，因此会在写光标的位置之前采取疏散策略(evacuation mechanism)，每个分片会被读出来然后再重新写回磁盘。对于正在被疏散的对象会有一个特殊的查找机制，确保这些对象可以在内存中被找到而不是磁盘，因为此时对象在磁盘上的数据是不可靠的。通常会提前在写光标之前做缓存扫描来发现是否有`pinned`对象，因为在写光标紧跟前会有一个固定区域(dead zone)，那里的数据不能被疏散. 待疏散的数据会先从磁盘上读取出来，然后放入写队列中等待写回磁盘。

对象是否可以被`钉住`需要通过cache.config配置文件来指定，并且proxy.config.cache.permit.pinning这个配置的值不能为0(默认是0)。写光标附近的对象如果正在被使用的话，也会自动地采用同样的疏散机制，但并不是通过Dir中的pinned这个位来设置。

### 其他注意事项
一些数据结构相关的观察.

#### Cyclone buffer
因为cache是循环写的，因此对象不会无限期保存在磁盘上，即使对象没有过期(stale)但仍然可能会被覆盖掉。`钉住`一个对象可以确保在写光标经过时还能保留，实际上是先它先读出来再存放回去。如果被钉住的对象过大或者过多的话会导致过多的磁盘开销。最初设计为让管理员来对很小又很频繁访问的对象来设置固定属性。

对象数据过期的目的是防止这些内容发送给客户端，它们并不是真正意义上的删除或清除。存储空间不会立即被回收因为写操作只会在写光标的位置发生，删除一个对象仅仅是删除这个对象在索引区的目录项，这就可以让被删除对象的文档完全失效。

Cache这样设计是因为web内容相对来说都是小对象而且内容经常变化，这样设计也是为了满足高性能低延迟的需求，因为存储上很少出现分片，而且cache miss和对象的删除不需要磁盘的I/O操作，但是在大对象的长期存储上确实不够理想。参见附录《Volume Tagging》中该领域一些工作的详述。

#### 磁盘故障
cache在设计的时候考虑到在一定程度上能够容忍磁盘故障情况的发生。如果一块磁盘发生故障，那么只会造成Cache分卷在这个磁盘上的`Cache带`的数据不可用，不会影响在其他磁盘上的`Cache带`的数据。要做的主要工作中就是保证其他正常`Cache带`上的数据能够继续使用，同时要将哈希到故障磁盘上的数据分配到其他的`Cache带`中，这些工作在下面这个函数中完成：

    AIO_Callback_handler::handle_disk_failure

重新将一块磁盘恢复到正常工作状态有点困难，更改一个cache key所属的`Cache带`会导致缓存中的当前数据不可访问，但是对于发生磁盘故障的磁盘来说，就不需要考虑这个问题，但是假如添加一块新磁盘，确定哪些缓存对象已经被清除掉了，还是有一些复杂机制的。这样的机制，如果存在的话，也仍是在调研中。

### 实现细节

#### 索引目录项

`Cache`带中的索引目录项结构定义如下：

![Dir](http://img.blog.csdn.net/20160628122140375)

class Dir, 在P_CacheDir.h文件中定义的。

`Cache带`的索引区由一组Dir对象组成，每一个目录项代表着`Cache带`上存储的一块缓存对象。因为每个对象至少要有一个索引目录项相对应，因此目录项结构的大小尽量精简。


offset成员表示对象在`Cache带`上的开始字节位置，由40个位来表示，拆分为offset(低24字节)和offset_high(高16字节)来组成。因为每个`Cache带`都有一个索引区，因此这个offset代表的是在这个`Cache带`中的偏移量。

size和big这两个成员用来计算对象分片的大概尺寸，这个只用来表示需要从offset这个偏移量处读取多少字节。分片的实际大小保存在Doc的元信息中，每当读取完之后就可以得到这个值。所以这个大概尺寸至少要和实际大小相等，也可以更大一些，但也会导致过多无用的读取。

分片的大致尺寸的计算方法如下：

(**size** + 1 ) * 2 ^ ( ``CACHE_BLOCK_SHIFT`` + 3 * **big** )

这里的`CACHE_BLOCK_SHIFT`代表一个基本cache块的位长(这里会是9，也就是一个扇区的大小，512字节)。因此替换之后也就是：

( **size** + 1 ) * 2 ^ (9 + 3 * **big**)

因为big这个成员的大小是2比特位，所以系数和大小对应关系如下所示：

![the multiplier of size](http://img.blog.csdn.net/20160628163604878)

如果size的值是0，那表示只有一个系数的大小。

分片大小可以在records.config中进行设置

`proxy.config.cache.target_fragment_size`

这个值应该设置为一个cache目录项系数(cache entry multiplier)的整数倍，不一定设置为2的幂次，大的分片会提高I/O效率，但也会导致有更多的磁盘空间浪费。默认大小是1M，这是一个在大多数环境下都相对合理的数字，不过在某些场景下调整这个数字会得到更好的性能。ATS内部有一个最大4194232字节的限制， 也就是4MB或者2^22，小于Doc结构体的大小。 事实上，较为合理的分片最大大小是4M - 262144 = 3932160字节。

当磁盘存储一个分片时，Dir中的size字段根据分片大小设置所能允许的精细粒度。为此，需要先查询目录项系数表，找到超过该分片大小的最小值，这意味着已经确定了big的值，因此也确定了近似大小的粒度。这也意味着读磁盘时，浪费磁盘I/O的最大可能字节。
When a fragment is stored to disk the size data in the cache index entry is set to the finest granularity permitted by the size of the fragment. To determine this consult the cache entry multipler table, find the smallest maximum size that is at least as large as the fragment. That will indicate the value of big selected and therefore the granularity of the approximate size. That represents the largest possible amount of wasted disk I/O when the fragment is read from disk.

一个`Cache带`中的目录项分组为segment数组，每个段中的目录项不会超过2^16个，段内部可以使用16位值去引用段中的任何其它目录项。

每个段中的目录项进一步分为深度为DIR_DEPTH(当前为4)的桶, 使用标准哈希表方式处理，按照上面的假定，每个段中的桶个数不超过2^14。

#### 目录项探测
目录项探测是指通过一个cache ID来在一个`Cache带`的索引区中查找一个指定的目录项。在程序中通过函数dir_probe()来完成，需要指定三个参数，分别是：cache ID(key)，cache带(d)和上一个冲突项(last_collision)。最后一个参数既是输入又是输出，在探测期间会被修改。

给定一个ID，高64位会被拿来计算属于哪个segment，通过取模运算(模上segment总数)。低64位用来计算属于哪个bucket，也是通过取模运算(模上一个segment中拥有的bucket的总数)。last_collision的值用来保存上一次匹配上的值，这个值是通过dir_probe()返回的。

通过计算之后得到了对应的桶，然后从这个桶中找出一个匹配上的项，通过比较cache ID的低12位和目录项中的cache tag来检查，从目录桶中的第一项开始，然后沿着链表去查找。如果查找到一个tag匹配上的项并且之前没有其他冲突(collision)的项那么就返回这个项并且把这项赋值给last_collision，如果设置了冲突项并且不等于当前match的项那么继续沿着链表查找，如果等于那么就重置collision然后继续查找。

这样设计的目的是在找到上一个冲突项(last_collision)之前忽略所有匹配的项，要返回之后匹配上的项(假如存在)。假如搜索到链表末尾，如果没有上一个冲突项，就返回miss结果，否则假定造成冲突的目录项已经从目录桶中清除，为此清除掉上一个冲突项，重新开始探测。这样做可能反复探测返回值，但是可以保证跳过不合法目录项。If the search falls off the end of the linked list then a miss result is returned (if no last collision), otherwise the probe is restarted after clearing the collision on the presumption that the entry for the collision has been removed from the bucket. This can lead to repeats among the returned values but guarantees that no valid entry will be skipped.

上一个冲突项在一段时间之后被拿来重新发起一次目录项探测，这一点相当重要，因为返回的匹配项可能不是我们实际想要的。虽然cache ID哈希到桶以及标签匹配(tag matching)不大可能产生误报，但是实际上也有产生误报的可能。当读一个分片时，可以得到完整的cache ID，检查后发现有错，将会丢弃这次读，接着进行下一次目录匹配并找到匹配，因为cache虚拟连接只跟踪上一个冲突值。This is important because the match returned may not be the actual object - although the hashing of the cache ID to a bucket and the tag matching is unlikely to create false positives, that is possible. When a fragment is read the full cache ID is available and checked and if wrong, that read can be discarded and the next possible match from the directory found because the cache virtual connection tracks the last collision value.

### Cache的一些操作
当HTTP请求头被解析完并经过remap处理之后就会做一些cache操作，隧道类的事务(tunneled transaction)不会对cache进行操作，因为从不解析header。

这里需要介绍一下术语`cache valid`，表示一个对象可以写入到cache中(比如，一个cache有效的url的DELETE操作本身不会缓存)。这很重要，因为Traffic Server在一个事务处理过程中会多次计算cache有效性，而且只会对cache有效的对象进行操作。在HTTP事务处理过程中这个标准还可能会发生改变，这是为了避免对cache无效的对象进行操作。

cache的三个基本操作是查找(lookup)、读取(read)和写入(write)。cache的删除操作(delete)视为写入操作的特殊情形，因为仅仅是更新对应的索引项。

当客户端的请求被解析完并且确认可以缓存时，会发起cache查找。如果查找成功，那么就会发起一个cache读取的操作，如果查找失败或者读取失败，但是判断内容是可以缓存的，那么会发起一个cache写入的操作。

#### 可缓存性
一个请求对cache进行的第一个操作就是判断这个请求的对象是否cache有效。解析和remap工作完成之后就会进行这个判断，如果不能缓存那么就表示后面的cache操作可以完全忽略，不会做cache的查找和写入操作。有许多预设值和配置选项可以改变对象的可缓存性。另外，缓存性检查在后面的过程中还会检查，特别是在transaction中(比如插件操作和回源响应)。这些检查在本节相关操作中做了恰当说明。

可能影响可缓存性(cacheability)的因素是

*  内建限制(built in constraints)
*  records.config中的设置
*  cache.config中的设置
*  插件中的处理

最初的内部检查和records.config的缓存性改写，在HttpTransact::is_request_cache_lookupable中处理。
这些检查细节包括

##### 可缓存的HTTP方法
请求方法必须是GET，HEAD，POST，DELETE，PUT之一，参见HttpTransaction::is_method_cache_lookupable()

##### 动态URL
ATS会尽量避免缓存动态内容，因为它的内容经常变化，一个URL视作动态的(dynamic)，假如

* 非HTTP或HTTPS
* 带有查询参数(query parameters)
* 结尾是asp
* 路径中带有cgi

通过设置下面的选项为非零值可以禁用该检查
`proxy.config.http.cache.cache_urls_that_look_dynamic`
另外假如cache.config中的规则设置了TTL，并且被匹配上，该检查也不会做。

##### Range请求
只有在records.config中设置proxy.config.http.cache.range.lookup为非零才能检查range请求的可缓存性，这并不意味着range请求就可以缓存，只有它满足缓存条件才可以缓存。另外，设置proxy.config.http.cache.range.write来尝试强制将range请求写到磁盘上面。此时可能没有作用，假如，源站忽略了Range：头，该选项就可以允许响应缓存。为了性能考虑，这默认是禁用的。

插件中可以调用TSHttpTxnReqCacheableSet()来强制设置请求是可缓存的。

#### Cache查找
如果HTTP请求未能确定cache无效，那么就会发起一次cache查找。cache查找用来判断一个对象是否已经在cache中，假如在缓存中，还要进一步定位缓存的位置。某些情况下，cache查找会进一步从磁盘上查找first Doc用来确认对象是否仍然在cache中。

一次cache的查找需要几个基本步骤：

1.  计算cache key
一般通过请求的URL来计算，但也可以被插件中的业务逻辑覆盖掉，据我所知cache的索引字符串并不会被保存下来，因为一般假定就是通过客户端请求头来计算。

2. 确定使用哪个cache带(也是通过cache key得到的)
cache key会被拿来当做一个哈希key来在cache 带(Vol实例)数组中进行查找，这个数组的构建和排列也能够体现出cache带是如何赋值的。

3. cache key还会被拿来计算索引key'，并用它探测在cache带中的索引。此外，各种其他索引也会被考虑进来，比如说aggregation buffer中的数据对应的索引

4. 如果查找到了索引，那么会从磁盘上读取到对应的first Doc，并验证其有效性。
在代码中是通过CacheVC::openReadStartHead()和CacheVC::openReadStartEarliest()这两个紧密相连的函数来完成的。

如果查找成功，那么会创建一个信息更全的目录项(结构体OpenDir)，这里要注意的是目录探测的时候也会检查是否已经有现存的OpenDir结构体了，如果已经存在那么直接返回，不需要再做其它工作。

#### Cache读取
Cache读取是在cache查找成功之后发起的，此时first Doc已经加载到了内存中，可以拿来查找其它信息。在first doc中会包含对象所有副本的HTTP头信息。

有了first doc中的一些信息就可以选中一个副本，这是通过比较客户端请求的信息和存储的所有副本响应头的信息而得到，不过这个也可以在插件中通过TS_HTTP_ALT_SELECT_HOOK点来做控制修改。

现在可以计算对象的新鲜性(freshness)来检查内容是否已经过期(stale)，这主要是检查对象的存放时间，方法是查看一下副本的http header和其他可能的元信息(注意选中副本之后才能做检查http头)。

绝大部分工作会在HttpTransact::what_is_document_freshness这个函数中完成。

首先，假如请求匹配上cache.config中带有TTL配置的配置行，就要检查检查存活期(time to live)值，基于对象放入缓存的时间，而不是http头中的任何数据

其次，假如cache.config中没有设置revalidate-after，检查内部标志needs-revalidate-once，假如已经设置，将该对象标记为过期。

在这些检查之后，在函数HttpTransactHeaders::calculate_document_age中计算对象的age，然后应用任何已经配置的模糊策略(fuzzing)，在函数HttpTransact::calculate_document_freshness_limit中基于已经获得的数据计算出对该age的限制。

records.config中的配置项proxy.config.http.cache.when_to_revalidate决定了如何使用该age，如果设置为0，会使用内建的计算方法来比较文档age和新鲜度限制(freshness limit), 并根据客户端提供的cache control值(max-age, min-fresh, max-stale)来进行修改，除非在cache.config中明确覆盖它们。

假如对象没有过期，它将会发送给客户端，假如过期了，客户端请求可能会被改变为一个If modifeid Since请求去源站验证的请求。

请求的处理使用到标准的虚拟连接隧道技术HttpTunnel，CacheVC充当生产者，客户端NetVC充当消费者。假如请求是Range请求，只需要再略作修改，使用一个变换来选择对象中恰当的数据部分，更或者，请求包含单个的range，可以使用range加速(range acceleration)。

Range加速的原理是这样的：查询earliest Doc中的分片偏移表，该偏移表包含所有分片对第一个分片的偏移，然后就可以加载包含首个请求字节的分片，而不用遍历读取内部的各个分片。

##### Read while write
源码中提到支持写时读(read while write)功能，这就是说，当一个transaction正在写缓存时，同时可以在另一个transaction中从缓存中读出数据返回给客户端，该功能的开启使用设置几个配置项，并且必须在records.config中开启该功能，否则，假如对象正在写入或是更新时，从缓存中读取该对象将会失败。

#### Cache写入
写Cache操作是通过CacheVC对象完成的，CacheVC是一个虚拟的连接(virtual connection)，接收数据并写入cache。如果需要在多个虚拟连接(VConn)之间传递数据的话，一般通过HttpTunnel来完成。写入cache时要将CacheVC对象作为Tunnel的消费者，它会和发送给客户端的虚拟连接并行执行。数据不会先写入cache再写到client, 数据会被拆分开并且同时向两个连接中并行发送，这也就避免了在两个连接上做数据同步。

每个CacheVC的事务是独立处理的，在`cache`带这个层面它们会互相影响，因为它们都会向`Cache带`中写数据。CacheVC内部会一直累积(accumulate)数据直到事务结束或者数据超过目标分片大小，如果是前一种情况的话整个对象会被直接写入`cache带`，如果是后一种情况的话会先将一个分片大小的数据写入`cache带`，然后CacheVC继续操作余下的数据，`cache带`会把这些不同的写请求的数据缓存到aggregation buffer中。

如果一个对象的大小不超过目标分片大小，那么对象会被整个写入cache，不会涉及到顺序。对于大的对象来讲，earliest Doc最先写入cache，first Doc最后写入cache，这有利于判断一个磁盘上的对象是否可以被覆盖，因为写光标会保证如果一个对象的第一个分片(earliest doc所在的)没有被覆盖的话，那么其他分片也不会被覆盖（当一个对象在cache中写完的时候，写光标会在这个对象的first Doc的位置）。

    CacheVC的逻辑需要保证不会一次性提交超过分片大小的写操作。

##### Writing to disk
实际写到磁盘的工作是由专门进行I/O操作的独立线程完成的，也就是AIO线程，缓存逻辑会序列化待写入磁盘的数据，然后将该后面的操作交接给AIO线程，一旦操作完成，AIO线程会发送信号来回调函数给上层的缓存逻辑。


#### 缓存更新
缓存写入已经涉及到缓存中现存的对象被修改的情况，这主要发生在如下几种情况下：

* 对源站发送条件请求(conditional request)并接收到304未修改响应
* 收到源站返回的副本并添加到对象中
* 移除对象的副本(比如，因为DELETE请求)

在上面的每种情况下，对象的元数据都会被修改。因为ATS从来不会更新那些已经缓存的数据，这意味着first Doc会被再次写入缓存中，并更新`cache带`上的目录项。因为已经处理了客户端请求，对象的first Doc被从缓存中读出并放入内存中，副本向量被适当更新(添加或删除目录项，或者修改去包含一个新的HTTP头)并写回磁盘。对多副本情形来说，可能会有不同的CacheVC实例同时来更新，唯一相同的内容就是first Doc，每个副本中剩余的数据都是独立的。


#### Aggregation Buffer
将缓存数据写入磁盘需要通过聚合缓存(aggregation buffer)。每个`cache带`都有一个聚合缓存，为了将系统调用次数减到最小，写入磁盘的数据大小近似与目标分片大小(target fregment size)。使用的算法很简单：数据先聚集在聚合缓存中直至接近(不超过)目标分片大小，然后一次性写入磁盘，相应的`cache带`中的目录项数据会更新磁盘中的数据实际存放位置，然后清空聚合缓存，重新处理。聚合缓存有专门的查找表，以便对象查找能够发现内存中的缓存数据。

因为聚合缓存中的数据对缓存中的其它部分是可见的，特别是缓存查找，没有必要将还没有填充慢的缓存数据急着写入磁盘。事实上，任何这样的数据都是内存缓存的，直至足够多的额外缓存数据到达来填满填充缓存。

目标分片大小对小对象几乎没有影响，因为分片大小只用来划分磁盘写操作，但是对较大的对象有一定的影响，因为它会影响这些对象分成几个分片，存放到`cache带`的不同位置，每个分片写入磁盘时，都有自己的目录项，它们是通过链式法则计算的(每个分片的cache key从前一个分片的cache key计算得到)。假如可能的话，分片表会汇总在earlist Doc中，分片表中的元素表示每个分片对首字节的字节偏移。

#### 疏散机制
默认情况下，一旦写光标在`cache 带`中至少移动一次，在移动过程中，它将会覆盖(事实上是从缓存中清除)经过处的对象。这在某些情况下令人无法接收，就会采用疏散方法来弥补，就是将不希望覆盖的对象从缓存中读出来再写回到缓存中，这样做就会将这样的对象在磁盘上的物理存放位置从写光标的前面移动到写光标的后面。使用疏散对象的方法基于`cache带`中的数据结构(即Vol实例)中的数据。

将`Cache带`上的缓存内容(volume content)划分为互不相交的大小为EVACUATION_BUCKET_SIZE的连续的字节区域，这些区域定义为疏散数据结构(evacuation data structure)。Vol::evacuate是元素代表疏散区域(evacuate region)的数组，每个元素是EvacuationBlock实例的双向链表。每个EvacuationBlock实例都包含一个目录项Dir，它指定了待疏散的分片。现在使用疏散桶(evacuation bucket)来表示上面的疏散区域(数组元素)，让每个EvacuationBlock放入疏散桶中，对应的分片位于上面的疏散桶中，每个疏散桶内的元素排序暂不关注，在疏散过程中会处理排序问题。移动疏散块内的first或是earliest分片就可以疏散对象，疏散操作接着处理对象中的其它分片，将它们添加到疏散块中。注意，这些分片的实际疏散会延迟至写光标到达该分片时，假如earliest分片已经疏散，就无须处理。

有两种类型的疏散：基于读的疏散和强制疏散。疏散块EvacuationBlock有一个读计数(reader count)用于追踪。假如读计数为0，它就是强制疏散，当写光标到达时，目标(假如存在)将会疏散；假如读计数非0，表示当前期望能够读该对象的实体(entity)个数，只要有对该对象的读访问(read access)，读计数就增加1，如果事先没有EvacuationBlock存在，就创建一个读计数为1的EvacuationBlock，当该对象的一次读访问完成时，就将读计数减少1，假如读计数为0，就删除EvacuationBlock。假如EvacuationBlock已经存在，读计数为0，没有被修改，且没有追踪读计数，只要对象存在，对应的疏散就是合法的(valid)。

缓存写来执行疏散操作，基本在代码Vol::aggWrite中实现。该函数处理那些等待写入`Cache带`的挂空缓存虚拟连接(pending cache virtual connections)，这其中有一部分是疏散虚拟连接(evacuation virtual connections)，假如是这种情况，数据一放入聚合缓存(aggregation buffer)就会回调疏散连接的完成回调函数。

当没有更多的缓存虚拟连接需要处理时(由于空队列或是聚合缓存还没有填满)，会调用Vol::evac_range来清空待覆盖的range，外加另外的EVACUATION_SIZE大小的range。检查涵盖该range的疏散桶，假如还要任何元素，就会创建一个新的缓存虚拟连接(一个doc evacuator)，用它来读取最靠近写光标的疏散项(i.e.`cache带`中的最小偏移)，而不会采用通常的聚合写操作。当读完成后，会检查数据的合法性，假如合法，相应的缓存虚拟连接会放入`cache带`的写队列的头部，然后恢复写聚合操作。

在执行缓存写之前，调用Vol::evac_range方法来开始一次疏散。假如在range内的疏散桶内找到任何分片，就选择earliest这样的分片(有最小的偏移，最接近写光标)，从磁盘中读出，挂空聚合缓存的写操作。通过缓存虚拟连接来执行读操作，该虚拟连接也可以充当读缓存，一旦完成读，就将该缓存虚拟连接实例(doc evacuator)放入`cache带`的写队列前部，然后依次写入磁盘。因为分片数据现在在内存中，重写该处的磁盘数据就无关紧要了。

注意，当恢复正常的`cache带`写操作是，相同的检查会再次进行，每次疏散一个分片(假如必要)，将它添加到队列中，依序写到磁盘上。

当疏散的分片写操作完成后，就更新目录新。当读完一个分片是，需要检查该对象是否是多分片的，假如该分片不是first分片，就将下一个分片标记为待疏散，依次地，读取并疏散下一个分片，该逻辑假定副本末尾是这种情况，next key不在该目录中。

上面的交互采用和聚合写逻辑(aggregation write logic)每次一个的策略。假如一个分片与一个正在执行疏散的分片相邻，它可能会放到同一个疏散桶中，因为聚合写(aggregation write)会每次检查下一个要疏散的分片，它会发现那个相邻的分片，在被覆盖之前疏散它。

#### 疏散操作
被疏散的分片主要是活跃的分片(active fragment)，也就是，那些当前打开进行读或是写的分片，这会在上面提到的疏散块(evacuation block)中的reader值中追踪。

假如开启了对象钉住(object pinning)功能，在写光标移动时，就会去进行通常的扫描来探测是否存在待钉住的对象，标记它们为待疏散的。

通过命中疏散(hit evacuation)，分片也可以被疏散，为此需要配置proxy.config.cache.hit_evacuate_percent 和proxy.config.cache.hit_evacuate_size_limit。当读一个分片的时候，需要检查它是否靠近或是在写光标前面，靠近的意思是指，小于`cache带`大小的指定百分比，假如设置默认值10，那么假如分片在`cache带`大小的10%以内，它将会被标记为待疏散的。疏散过大的分片会影响系统性能。

#### 初始化
存储设备的配置文件，默认是storage.config，磁盘初始化由一个Store实例开始，一开始会去读取存储设备的配置文件，对于配置文件中的每行合法配置，会创建一个Span对象。基本上有4种类型：

*  文件
*  路径
*  磁盘
*  裸设备(raw device)

创建完所有的Span实例之后，按照设备ID分组构造一个内部链表，挂接到Store::disk数组中，引用相同目录，磁盘或是裸设备的Span会组合成一个单独的条带(span)，引用相同文件但是具有重合偏移的Span也会组合起来。
ATS启动时调用ink_cache_init()来完成所有这些工作。

配置文件初始化后，调用CacheProcessor::start()来启动cache processor，它完成了许多事情：

对每个合法的Span，创建一个CacheDisk实例，这是一个continuation实例，所以能用来执行Span上那些有可能阻塞的操作。CacheDisk实例最主要的用途是作为参数传递给AIO线程，当一个I/O操作完成后，作为回调函数，然后是分发到AIO线程去完成存储单元的初始化(storage unit initialization)。在所有这些完成后，在cplist_reconfigure函数中，得到的存储(storage)会分散到`Cache分卷`(volumes)，同时也会创建CacheVol实例。

一旦初始化所有的条带(这就是说，所有条带的条带头信息已经成功从磁盘中读出)，`cache带`的分配搭建就完成了。分配信息使用索引数组存放，它们是映射到条带数组的索引。分配信息和条带数组存放在CacheHostRecord实例中。分配初始化(assignment initialization)由大多数分配数组(assignment array)组成，它比条带数组(stripe array)更大。

hosting.config文件中的每行都是一个CacheHostRecord实例，一个更一般的实例。对配置好的实例来说，条带集合由配置行中指定的`cache分卷`确定，假如没有这样的配置行，所有的条带被放入更一般的记录中，否则只有那些标记为默认的条带才会放入更一般的记录中。

	假如没有指定hosting记录，没有指定至少一个默认的`cache分卷`就是一个错误。

build_vol_hash_table函数会调用每个CacheHostRecord实例来初始化分配表。对host记录中的每个`cache带`会生成一个伪随机数序列，该序列以条带的哈希标识符的folded哈希开始，条带的哈希标识符是设备路径，然后是该条带的skip和size值，这样的值是唯一的，这种方法对任何特殊的条带的序列值也是唯一确定的。

每个条带会为存储中的每VOL_HASH_ALLOC_SIZE字节(当前是8MB)从它的序列中得到一个数，这些数与条带索引配对，然后由随机值排序，得到的数组对条带分配表中的每个slot取样，通过最大随机值除以分配表的大小，再取中间值。这样组合得到的伪随机序列依次对每个采样做扫描，不大于采样的首个元素将被找到，与这个值相关的条带被用作分配表的元素。

当这套程序确定下来之后，ATS会受初始条件的影响，包括每个`cache带`的大小。
