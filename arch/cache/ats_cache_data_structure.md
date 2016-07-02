

## ATS缓存数据结构


#### HttpTunnel类
数据传输驱动器(data transfer driver)，包含一个生产者(producer)集合，每个生产者连接到一个或是多个消费者(comsumer)。隧道(tunnel)处理事件和缓冲区以便数据能从生产者移动到消费者，数据会尽可能保存在引用计数类型的缓冲区中。只有数据发生变动，或者在数据源(它从ATS外部获取数据)和数据接收端(它将数据发送到ATS外部)的情况下，才会发生拷贝操作。

#### HTTPCacheAlt类
定义在HTTP.h中，它是一个缓存对象中单个副本的元数据(metadata)。包含下面的信息：

*  副本的earliest Doc对应的Dir
*  请求头和响应头信息
*  分片偏移表(fragment offset table)
*  源站请求和响应的时间戳(timestamp)

#### HTTPInfo类
定义在HTTP.h中，它是HTTPCacheAlt的包装类。它提供了外部API来访问包装类内部的数据，它只含有一个指向包装类实例的指针(可能为NULL)。

#### CacheHTTPInfo类
HTTPInfo类的typedef。

#### CacheHTTPInfoVector类
定义在P_CacheHttp.h中，它是HTTPInfo对象组成的数组，充当一个对象所有副本的信息仓库。

#### OpenDir类
一个打开的目录项(directory entry)，包括一个`Dir`所有的信息，外加从first Doc中获取的额外信息。

#### CacheVC类
接收输入数据并写到缓存中的虚拟连接类。

* int CacheVC::openReadStartHead(int event, Event *e)
执行读取一个缓存对象(cached object)的初始化工作
* int CacheVC::openReadStartEarliest(int event, Event *e)
执行读取一个缓存对象的副本(alternate)的初始化工作

#### CacheVol类
保存`volume.config`配置文件中一行的数据的类，一行表示一个`缓存分卷`。

#### CacheControlResult类
保存`cache.config`配置文件中一行的数据的类。

#### EvacuationBlock类
用于记录疏散的相关信息(record for evacuation)。

#### Vol类
表示`cache分卷`内的一个存储单元(过时的叫法storage unit，现在叫作cache strip)，也叫作volume，注意跟磁盘分卷的那个volume是有区别的。

* off_t Vol::segments
`缓存带`中的段的个数，由该`缓存带`中的所有目录项(directory entry)除以一个段中的目录项数得到，是个粗略估计值。
* off_t Vol::buckets
`缓存带`中的桶的个数，由目录段中的所有目录项(directory entry)除以DIR_DEPTH(当前为4)得到。是个粗略估计值，按照当前的定义值，这个数大约是16384(2^16/4)，目录桶用来作为索引哈希(index hash)的目标。
* DLL&lt;EvacuationBlock&gt; Vol::evacuate
元素为EvacuationBlock的桶组成的数组，它按照大小排序，以便每个疏散带(evacuation span)都有一个EvacuationBlock桶。
* off_t len
`缓存带`的字节长度。
* int Vol::evac_range(off_t low, off_t high, int evac_phase)
假如从*low*到*high*的字节带上存在任何EvacuationBlock，就开始一次疏散。假如没有疏散发生，返回0，否则返回非零值。

#### Doc类
在P_CacheVol.h中定义。

* uint32_t Doc::magic
校验值，对合法文档(document)设为DOC_MAGIC。
* uint32_t Doc::len
包含HTTP头长度，分片表和Doc结构体的段的长度。
* Doc::total_len
整个文档的总长度，不包含元数据，但是包含HTTP头信息。
* Doc::first_key
文档(document)的首个索引键值，用于定位`cache带`中的缓存对象。
* Doc::key
分片的索引键值(index key)，分片键值可以通过链式方法计算，使得下一个和上一个分片的键值可以从当前键值计算出来。
* uint32_t Doc::hlen
文档头(即元数据)长度，注意不是HTTP头的长度。
* uint8_t Doc::ftype
分片类型，当前只用到CACHE_FRAG_TYPE_HTTP，其它类型用于后续缓存扩展，目前还没有实现。
* uint24_t Doc::flen
`分片表`(fragment table)长度，假如存在分片表的话。一个缓存对象只有first Doc分片中才含有分片表。分片表是从第一个分片首字节开始计算时，各分片中HTTP响应内容(不包含元数据和HTTP头)的相对字节偏移所组成的列表。分片表中的第一个元素表示的是第二个分片中的字节偏移，类似于数组从索引1开始计算，因为第一个分片的字节偏移是总是0，无须计算在内。这样做的目的是为了在range请求的快速查找。假如给定first Doc分片，包含range请求首字节的分片将会直接计算和读取，不需要更多的磁盘访问。
ATS 3.3.0之后移除了。
* uint32_t Doc::sync_serial
* uint32_t Doc::write_serial
* uint32_t pinned
钉住对象(pinned object)的标志和计时器
* uint32_t checksum

#### VolHeaderFooter类

#### 参考文献
https://docs.trafficserver.apache.org/en/latest/developer-guide/architecture/data-structures.en.html