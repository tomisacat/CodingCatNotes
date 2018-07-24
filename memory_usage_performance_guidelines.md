# Memory Usage Performance Guidelines

### Virtual Memory

iOS 和 macOS 都使用了 Virtual Memory 系统，对于 32bit 系统，寻址空间为 4GB，64bit 系统寻址空间为 18EB。macOS 支持 backing store，在内存不足时将暂时用不到的内存数据写到磁盘上。

> 但是在 MacOS Maverick 系统时已经开始使用内存压缩技术

iOS 系统面临内存不足压力时，对于只读数据（比如说代码页）简单的将它们从内存中移除，对于可写数据则给 app 发送内存警告。

> 从 iOS7 开始也使用了内存压缩技术

VM 为每个进程创建一个逻辑地址空间，并将这个空间划分为均匀大小的内存块，这个块称为一页（Page）。CPU 和它的内存管理单元（MMU）维护了一个页表，这个表将程序的逻辑地址映射到真实内存的硬件地址上。这个转换的工作由 CPU 和 MMU 自动完成，对应用程序来说是透明的。

对于应用程序来说，它的逻辑地址空间总是可以访问的，但是当它访问一个不再物理内存上的地址时，就会产生缺页错误（page fault）。此时，VM 系统调用一个特殊的缺页句柄（handler），它会暂停执行当前进程，找到物理内存上一块未使用的空间并将需要用到的数据从磁盘加载到这块空间，然后更新页表并将控制权重新返回给进程，这一步称为页面调度（Paging）。

对早期的 OS X 和 iOS 来说，页大小为 4KB，后来，基于 A7 和 A8  64bit 处理器的操作系统则对外暴露 16KB 每页大小，但对于物理页依然是 4KB 大小。基于 A9 处理器的则都是 16KB 大小。

一个进程的逻辑地址空间由一系列被映射的内存区域（region）组成，每个这样的内存区域包含多个虚拟内存页以及各自特定的属性，例如继承（这个区域的一部分由“父区域”映射得到），写保护，是否已连线（wired，也就是说，不能被分页调度出去）。因为这样的内存区域包含一系列的页，因此他们是按页对齐的，这意味着一个区域的开始也是一页的开始，一个区域的尾端也是一页的尾端。

内核为逻辑地址空间每一块区域创建一个 VM object 对象，每个 VM object 包含一个关联内存区域与分页器（默认的或者 vnode）的映射表。默认分页器由系统创建，负责管理非寄宿虚拟内存页，vnode 分页器实现了内存映射的文件访问机制，它让用户像读写内存中的数据一样直接读写文件。

一个 VM object 也可以映射到另一个 VM object，内核使用这种方法实现写时复制区域（copy-on-write regions）功能。这个功能允许不同进程（或一个进程里的不同代码块）共享内存页直到某个进程尝试写数据到这个页里，这个功能通常用在加载系统库的时候。

每个 VM object 包含以下字段：

| Field | Description |
| :-: | :-: |
| Resident pages | A list of the pages of this region that are currently resient in physical memory. |
| Size | The size of the region, in bytes. |
| Pager | The pager responsible for tracking and handling the pages of this region in backing store |
| Shadow | Used for copy-on-write optimization |
| Copy | Used for copy-on-write optimization |
| Attributes | Flags indicating the state of various implementation details  |

如果 VM object 包含 copy-on-write 操作（vm_copy），那么 `shadow` 和 `copy` 字段可能指向其他 VM object。否则都是 NULL

### Wired Memory

Wired Memory，也称为 resident memory，它存储内核代码和数据结构，并且永远都不能换页移出到磁盘上。应用程序，框架，以及其他用户层软件都不能分配 wired memory（但它们依然可以影响 wired memory 的分配）。例如，一个程序创建了线程和端口，那么它就隐式地为所需要的内核资源创建了 wired memory。

| Resource | Wired Memory Used by Kernel |
| :-: | :-: |
| Process | 16KB |
| Thread | blocked in a continuation-5KB; blocked-21KB |
| Mach Port | 116 bytes |
| Mapping | 32 bytes |
| Library | 2 KB plus 200 bytes for each task that uses it |
| Memory region | 160 bytes |

> 上面的数据可能随系统升级发生变化，它只是告诉你一个大概的系统资源消耗

可以看到，每个线程，进程和库都对系统内存（resident footprint of the system）产生较大影响。除了应用程序会产生 wired memory 之外，内核使用下面这些实体的时候也需要 wired memory：

* VM objects
* the virtual memory buffer cache
* I/O buffer caches
* drivers

Wired data structure 同样与物理页以及映射表产生关联，因此当系统内存增加时，即使你什么都没做，wired memory 数量也会增大。当启动系统并且不运行任何其他时，wired memory 可能会消耗大约 14MB 内存（系统一共消耗  64MB）或 17MB （系统一共消耗 128 MB）。

### Page lists in the Kernel

内核维护下面三个系统级的物理内存页：

* active lists：包含当前映射到物理内存并且近期使用过的内存页
* inactive lists：包含当前存储在物理内存中但近期没有使用的内存页。这些页包含合法数据但可能会随时被系统从内存中移除
* free lists：包含物理内存中没有与任何 VM object 对象地址空间关联的内存页。

当 free lists 页数低于阈值（由物理内存大小决定）时，分页器通过调整 inactive lists 里的页面来保持页面队列的均衡。如果某一页最近被使用过，那么就将它放入到 active lists 末尾。对 OS X 来说，如果一个 inactive 页包含未写入 backing store 的数据，那么将它放 free list 之前要先将数据写入磁盘。而对于 iOS 来说，已修改但当前不活跃的页（modified by inactive pages）必须由应用自己处理。

### Paging Out Process

如果 free lists 页数低于阈值，则内核会执行以下操作：

1. 如果 active lists 里的某一页近期没有使用，则将它移动到 inactive lists 里
2. 如果 inactive lists 里的某一页近期没有使用，则查询它的 VM object
3. 如果这个 VM object 从来没有被分页，内核会调用初始化方法为它分配一个默认的分页器
4. 这个 VM object 的默认分页器会尝试将它的数据写回
5. 如果上面处理成功，内核释放这页关联的物理内存并将这页从 inactive list 移动到 free lists 里

> 对于 iOS 来说，系统会通知应用自己释放内存

### Paging In Process

当代码试图访问一个没有映射到物理内存的虚拟地址时会产生一个内存访问错误，它包括以下两种分类错误：

* soft fault：想要访问的页数据已经存在物理内存上但还没有与这个进程产生关联映射
* hard fault：想要访问的页不在物理内存上但存在于 backing store（或者映射到了一个文件上）。这也称之为缺页错误。

当错误发生时，内核会寻找想要访问区域的 VM object，接着会查看这个 VM object 的 resident pages 表，如果想要访问的页就在这个表里，则内核会产生一个 soft fault，否则就产生一个 hard fault。

对于 soft fault，内核会将包含这一页的物理内存与这个进程的虚拟地址空间关联，并将这一页标记为 active，如果这次访问是一个写操作，则内核还会将这也标记为 modified。

对于 hard fault，VM object 的分页器会寻找这一页在 backing store 或者磁盘上文件里的位置，修改页表信息并将这页数据移动到物理内存。

### malloc

如果使用 malloc 申请大小很小的内存，它会向一个内存列表（或者叫内存池）请求所需要的内存，使用完之后使用 free 释放内存时会将这块内存放回到这个内存池里。内存池本身由多个虚拟内存页组成，系统使用 vm_allocate 来喂你自动分配和管理。

分配小内存块时，malloc 分配的内存按 16bytes 对齐，因此如果你申请的内存小于 16bytes 则会分给你一个大小为 16 bytes 的内存块，如果大于 16 bytes，则分配 16 的倍数大小的内存块（例如申请 24bytes 会得到 32 bytes 大小的内存块）。

如果使用 malloc 申请一块较大的内存，malloc 会直接使用 vm_allocate 获取一块逻辑地址空间，但此时并不会立刻为它分配对应的物理内存，内核会做如下操作：

1. 创建一个 map entry，这个数据结构定义了内存起始和结束的地址
2. 这个内存范围由默认分页器控制
3. 创建并初始化一个 VM object，并将它与 map entry 关联

此时，物理内存和 backing store 都还没有页面与之关联，当代码访问这段内存块时会产生一个缺页错误，OS X 会执行如下操作：

1. 从 free lists 申请一页并用 0 初始化
2. 在 VM object 的 resident pages 列表里插入上面申请的那一页
3. 使用一个称为 pmap 的数据结构来映射虚拟内存和物理内存。pmap 里包含页表（page table）。

分配大内存块时，malloc 分配的内存对齐大小与虚拟内存页大小相同或者 4096bytes。

从上面可以知道，分配大内存时使用 malloc 与直接使用 vm_allocate 是一样的：

```c
void* AllocateVirtualMemory(size_t size)
{
    char*          data;
    kern_return_t   err;
 
    // In debug builds, check that we have
    // correct VM page alignment
    check(size != 0);
    check((size % 4096) == 0);
 
    // Allocate directly from VM
    err = vm_allocate(  (vm_map_t) mach_task_self(),
                        (vm_address_t*) &data,
                        size,
                        VM_FLAGS_ANYWHERE);
 
    // Check errors
    check(err == KERN_SUCCESS);
    if(err != KERN_SUCCESS)
    {
        data = NULL;
    }
 
    return data;
}
```

##### allocating memory in batches

可以用 malloc_zone_batch_malloc 一次性分配多个大小相同的内存块，它比多次调用 malloc 效率更高，尤其是如果单个内存块大小小于 4KB 时性能最好。

##### using malloc memory zones

所有的内存块都分配在 malloc zone （也称为 malloc heap）里，是一个可变长度的虚拟内存区域，它有自己的 free list 和内存池，主要用于创建用途或生命周期类似的内存块。你可以在这个 zone 里创建许多对象或者内存块，使用完之后直接销毁这个 zone，它会自动释放这些创建出来的对象和内存块而不需要你手动一个个去释放。理论上来说，使用 zone 可以减少内存空间的浪费和换页操作。

> zone 这个术语一般与 heap，pool 是同义词

## 延伸

* [Understanding iOS Memory](https://docs.google.com/document/d/1J5wbf0Q2KWEoerfFzVcwXk09dV_l1AXJHREIKUjnh3c/edit#)
* [Finding iOS memory](http://liam.flookes.com/wp/2012/05/03/finding-ios-memory/)
* [Handling low memory conditions in iOS and Mavericks](http://newosxbook.com/articles/MemoryPressure.html)
