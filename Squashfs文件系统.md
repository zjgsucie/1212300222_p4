##Squashfs 4.0 文件系统

####[Linux源文档](https://github.com/torvalds/linux/blob/v3.18/Documentation/filesystems/squashfs.txt)

Squashfs是Linux下的一种只读压缩文件系统类型。它使用zlib/lzo/xz等压缩算法来压缩文件，节点及目录。Squashfs文件系统内的节点非常小巧并且所有的数据块都排列紧凑，通过这种方式来降低数据存储开销。数据块大小可以取在4KB到1MB，但默认大小为128KB。

Squashfs为通用只读文件系统所用，常用来保存归档文件

####文件系统特性


Squashfs 文件系统 VS Cramfs 文件系统特性:

Linux内核文档翻译之Squashfs文件系统

Squashfs 会将数据、节点及目录进行压缩。另外，节点和目录数据是字节对齐并且是高度紧凑的。每个压缩的节点平均仅占用8字节长度（确切长度会因为文件类型而不同，比如普通文件、目录、符号链接及块/字符设备节点等）。

####使用 Squashfs文件系统
Squashfs
squashfs作为一种只读文件，必须用mksquashfs工具来创建压缩的squashfs文件系统。mksquash工具及unsquashfs等工具可以从http://www.squashfs.org网站获取。这些squashfs相关工具代码树现在已经被集成到kernel.org内：
git://git.kernel.org/pub/scm/fs/squashfs/squashfs-tools.git

####Squashfs 文件系统设计

Squashfs 文件系统最多包含9种字节对齐的模块：
Linux内核文档翻译之Squashfs文件系统
压缩的数据块就像从某个目录里读文件一样写到文件系统并进行冗余检查。当文件数据被写入到文件系统， completed inode, directory, fragment, export, uid/gid lookup 和xattr tables也会同时被写入到文件系统里面。 

 + 索引节点

 元数据 (索引节点和目录) 一般被压缩成为8KB大小的数据块。每个压缩块前都有2字节的前缀头，当某数据块被压缩时前缀头的最高位会被置1，当然-noI 选项被置1或者压缩部分的大小大于未压缩大小时，数据块将会被解压。

 索引节点会被打包进元数据块，但并未以块大小对齐，因此索引节点会覆盖压缩数据块。索引节点是用一个48位数来标示的，这个48位数包括含有该索引点的压缩元数据块的位置及索引节点在块内的偏移量<block,offset>。
 为了提高压缩率，Squashfs文件系统针对不同的文件类型（普通文件、目录文件、设备文件等等）采用不同的索引节点，并且这些不同索引节点有不同的数据位数。为了进一步提高压缩率，普通文件索引节点和目录文件索引节点定义如下：为常用的普通文件和目录文件优化并且保存了额外信息的节点类型

 + 目录

 和索引节点一样，目录文件同样是被打包进元数据块内，只是被存放在目录表内而以。同样的，目录文件的存取是通过包含该目录的元数据块的起始地址及块内偏移来完成的<block,offset>。

 目录不是通过简单的文件名列表而是一种稍显巧妙的方式来组织的。该组织方式充分利用了文件的索引节点会存放在同一元数据块内的事实，因此可能共享其块起始地址。因此目录组织结构包括两部分：包含共享块起始地址的目录头和使用了共享块起始地址的顺序目录项。当索引节点的起始块地址改变时，目录头也会被修改。目录头和目录项会尽可能的重复。
 目录包含一个用来快速检索文件的目录索引，并且目录是有序的。目录索引会在每个元数据块储存一个条目，而每个条目里则存放着该元数据块的第一个目录头对应的索引或者文件名。目录名是以字母顺序排列的，检索时内核会线性地搜索大于被查找文件名第一个字母的文件名，这样包含该文件名的元数据块的位置就可以确定了。这种方法的核心所在是这种索引方法可以保证要检索的元数据块不依赖于目录的长度，另外的优点是这种方法不需要额外的内存及磁盘存储空间。

 + 文件数据

 普通文件由一系列连续的压缩数据块及压缩片段块组成（尾端压缩块）。每个数据块的压缩大小存放在文件节点的块列表内。
 为了加快读大“数据”(256MB或者更大)的速度，squashfs实现了一个缓存从块索引映射到磁盘上的数据的缓存索引。
 这种缓存索引允许squashfs处理大文件（1.75TiB级别的）但仅仅在磁盘上保存一个简单小巧的块列表。这种缓存又被划分为更小的槽，每个槽可以缓存224GiB大小的文件（128KiB块）；大文件即可以使用多个槽，比如1.75TiB使用8个槽。缓存索引被设计为一种内存高效的方法，默认情况下大小为16KiB。

 + 片段检索表（这部分感觉读的不顺，后面再翻译）

 Regular files can contain a fragment index which is mapped to a fragment
 location>第二个索引表用来定位这些，第二个索引表可以提高存取速度（因为它很小），在挂载时读取并缓存在内存中。

 + 导出表

 为了使Squashfs文件系统成为可导出文件系统（通过NFS等），可以选择（通过mksquashfs工具加-no-exports来禁用）包含一个inode号到inode磁盘磁盘位置的查找表。需要使用Squashfs来使能inode号到inode在磁盘上的位置的映射，当导出代码重新实例过期/刷新的inode 时这个很有必要。这个导出表会压缩进元数据块。第二个索引表用来定位这些，第二个索引表可以提高存取速度（因为它很小），在挂载时读取并缓存在内存中。

 + 扩展属性表

 扩展属性表包含每个节点的扩展属性。 每个索引节点的扩展属性被存储在一个列表中，每个列表条目包含一个type，name和value字段。type字段使用扩展属性前缀("user","trusted."等)来编码，它也确定name/value字段应该被解释。目前type用来指明value值是否内嵌存储（在这种情况下，value字段包含扩展属性值） ，或者外部存储（在这种情况下，value字段中存储有扩展属性实际存储位置的指针） 。这使得较大的属性值可以通过外部存储的方式来提高扫描检索性能，同时 它还允许该值被"反拷贝"，该值仅被存储一次，其他的就可以并发使用一个引用来访问该值。

 扩展属性列表被打包压缩成8K元数据块。为了减少节点的开销，一个32位扩展属性ID被保存而不是存储扩展属性的磁盘位置到每个inode内。这个32位扩展属性ID使用第二个扩展ID检索表映射了扩展属性列表的位置。


####未来的工作及突出问题

 + 未来工作列表

 实现 ACL 支持

 + Squashfs 内部缓存

 Blocks in Squashfs are compressed.  To avoid repeatedly decompressing
 recently accessed data Squashfs uses two small metadata and fragment caches.

 The cache is not used for file datablocks, these are decompressed and cached in
 the page-cache in the normal way.  The cache is used to temporarily cache
 fragment and metadata blocks which have been read as a result of a metadata
 (i.e. inode or directory) or fragment access.  Because metadata and fragments
 are packed together into blocks (to gain greater compression) the read of a
 particular piece of metadata or fragment will retrieve other metadata/fragments
 which have been packed with it, these because of locality-of-reference may be
 read in the near future. Temporarily caching them ensures they are available
 for near future access without requiring an additional read and decompress.

 In the future this internal cache may be replaced with an implementation which
 uses the kernel page cache.  Because the page cache operates on page sized
 units this may introduce additional complexity in terms of locking and
 associated race conditions.