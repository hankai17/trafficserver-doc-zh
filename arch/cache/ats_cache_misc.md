
#### 将追加的主题

* 内存常驻副本(resident alternates)
* 缓存对象刷新(object refresh)

####  缓存一致性(Cache Consistency)
ATS缓存是完全一致性的，除非你不小心踢掉电源，让ATS突然关机。如果要禁用硬盘驱动器自身的缓存，你需要使用下面的命令
 
 	hdparm -W0

缓存系统会校验可用文档(document)的所有数据，并默认将不完整的文档标记为读MISS。无须小心翼翼地关闭ATS，你只需简单杀死ATS进程，下次ATS启动后，缓存恢复代码fsck会每隔一定时间运行。

ATS启动后，会检查两种目录版本(注意`cache带`中会有首尾两个VolHeaderFooter结构)，最后会将合法的目录读入内存中。ATS会从上次突然中断的写光标处开始向前移动，读取所有写到磁盘中的`分片`(fragment)，并更新目录(就像基于日志的文件系统)。在写操作的时候，读操作会停下来，直至看到上次合法的写头部(last valid write header)，因为磁盘扇区的重新排序，写操作不一定是原子的。那些新更新的操作会写到非法的(invalid)版本里(万一系统启动过程中崩溃了呢)，然后系统启动了。

#### 缓存分卷标签(Volume Tagging)
当前`缓存分卷`在缓存设备上的分配有些随意，一个增强版本[1]允许storage.config分配特定的`缓存带`给特定的`缓存分卷`，但是通常volume.config中仍然需要配置这些`缓存分卷`，特别是要映射那些域名到指定的`缓存分卷`。这样做的一个主要用途就是映射指定类型的内容到不同的缓存设备(SSD或是普通机械硬盘)上。比如在storage.config中配置：

	/dev/sda object_size=8k volume=1
	/dev/sdb object_size=1M volume=2

#### ATS版本升级
当前对缓存格式的任何变动都会导致ATS清除缓存。当升级ATS版本时，一定要记住这一点。

#### 缓存Key控制
缓存key默认是请求的URL。有两种可能的选择，原始URL和重映射(remapped)后的URL，使用哪个URL，由下面的配置项决定：
proxy.config.url_remap.pristine_host_hdr
这是`INT`值，假如设置为0(禁用)，使用remapped的URL，假如设置为非0值(启用)，使用原始URL。这个配置项也会影响发送给源站的请求中的Host头的值，假如是0，使用重定向后URL中的hostname，否则使用原始URL中的hostname，此外没有其它影响。

对于缓存来说，如果没有使用到重定向规则，或者原始URL和重定向URL仅是一一对应的映射关系，对缓存来说，基本上没有啥影响。

假如多个原始URL映射到同一个重定向URL，情况就有些复杂。假如启用了本真头(pristine header)，对不同原始URL的请求将会存储为不同的对象，假如禁用，就只会使用重定向后的URL存储，这会造成冲突(collision)。假如这些内容不相同，这就很糟糕，但是假如它们的内容都相同(类似原始URL仅是仅是同一源站资源的别名这种情况)，这就比较有用了。

假如原有重映射规则改变了，就会有问题。假如原始URL重映射到一个不同的服务器地址，上面的配置项将会决定是已经缓存的对象响应新的客户端请求(启用时)，还是不响应(不启用)。类似地，假如重映射规则中的原始URL改变了，那么假如禁用本真头，来自最初原始URL的缓存对象将会响应客户端请求，而不是更新后的原始URL对应的缓存对象来响应客户端请求。

这些冲突自身来说没有好坏之分，管理员需要根据自己的业务场景来确定合适的值并做相应的设置。

假如需要更大程度地控制，可以在插件中调用API函数TSHttpTxnCacheLookupUrlSet()或TSCacheUrlSet() 来设置专门的缓存key。TSCacheUrlSet()接口能最早在TS_HTTP_READ_REQUEST_HDR_HOOK钩子中调用，但最晚必须在TS_HTTP_POST_REMAP_HOOK钩子中调用。每个`事物`(transaction)中只能调用一次。多次调用该API没有任何作用。

改变缓存key的插件必须对缓存HIT和缓存MISS这两种情况做出一致性处理。因为对映射到同一个缓存key的不同请求，缓存应该做同样的处理。直接使用URL是这样做的，任何其它的替代品也应该是这样。这完全是插件的责任，对ATS内核来说，没有办法知道这些细节。

当使用TSHttpTxnCacheLookupUrlSet()或TSCacheUrlSet() 设置了新的缓存url之后，可以调用TSHttpTxnCacheLookupUrlGet()函数来查看新设置的缓存url，应该使用TSUrlCreate()生成的URL定位符(URL location)来作为第三个参数，而不是从客户端请求中获取url_loc.

一般要求缓存url是形如URL的字符串，但是也可以完全随意，并且不一定要求含有path。比如，假如Network Geographic公司想使用它自己的缓存key来存储每个资源，使用文档GUID作为key的一部分，缓存key类似

	ngeo://W39WaGTPnvg

格式ngeo专门拿出来说，是因为它不是一个合法的URL格式(scheme)，因而也不会与任何合法的URL冲突。

假如将重要和不重要的数据都编码进URL，可能很有用，事实上，可创建和使用只含有重要数据部分的url来存放数据，而不必使用多个不同的URL(仅在不重要的数据部分有区别)来存放潜在相同的内容。

比如假设Network Geographic公司的内容URL使用文档GUID和Refferrer头键(referral key)组合来编码：

	http://network-geographics-farm-1.com/doc/W39WaGTPnvg.2511635.UQB_zCc8B8H

这里Referrer头的值为http://network-geographics-farm-1.com/doc，GUID为W39WaGTPnvg.2511635.UQB_zCc8B8H。我们对每个可能的Referrer不想提供相同的内容(还要考虑后面的GUID呀)，而是，我们想使用一个插件来将这种情况转换为前面的例子(只有GUID)，仅是Referrer key不同的请求(后面GUID必须一样)将会引用相同的缓存对象。注意我们也映射下面的url到相同的缓存key

	http://network-geographics-farm-56.com/doc/W39WaGTPnvg.2511635.UQB_zCc8B8H

当内容完全相同时，这将会很方便地让不同的源站返回相同的内容，插件可以改变缓存key，或者不改变，这依赖请求头中的相关数据。比如，假如请求的PATH不在`doc`目录中，就不改变缓存key。假如需要区分服务器，可以轻易从请求URL获取内容并用来组合缓存key。实现者可以很自由地从缓存key中提取出所有的相关元素。

当没有明确要求时，组合缓存key基于HTTP请求头，实际上，因为一致性要求它通常很有必要。因为缓存查找发生在回源连接之前，我们无法获取到HTTP响应头的任何信息，除了仅有的请求头，最常用的情形是上面描述的，我们的目的就是剔除URL中不影响存储内容的元素，来将缓存容量最小化，同时提高缓存命中率。

#### 参考文档
https://docs.trafficserver.apache.org/en/latest/developer-guide/architecture/consistency.en.html


[1]:https://issues.apache.org/jira/browse/TS-1728