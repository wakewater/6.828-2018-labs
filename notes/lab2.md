# MIT 6.828 lab2 通关指南

>这篇是我自己探索实现 MIT 6.828 lab2 的笔记记录，会包含一部分代码注释和要求的翻译记录，以及踩过的坑/个人的解决方案

lab2 主要是关于内存管理的部分。内存管理包含两个组件：

- 内核的物理内存分配器：
  - 任务将是维护数据结构，该数据结构记录哪些物理页是空闲的，哪些是已分配的，以及多少进程正在共享每个分配的页。您还将编写例程来分配和释放内存页面。
- 虚拟内存
  - 您将根据我们提供的规范修改JOS以设置MMU的页表。


实验2包含以下新的源文件:

- inc/memlayout.h：描述了必须通过修改pmap.c来实现的虚拟地址空间的布局
- kern/pmap.c
- kern/pmap.h：PageInfo 用于跟踪哪些物理内存页可用的结构
- kern/kclock.h：操纵PC的电池供电时钟和CMOS RAM硬件，其中BIOS记录PC包含的物理内存量。
- kern/kclock.c

## 第1部分：物理页面管理

操作系统必须跟踪物理RAM的哪些部分空闲以及当前正在使用哪些部分，现在，您将编写物理页面分配器：它使用struct PageInfo对象的链接列表（与xv6不同，它们不嵌入在空闲页面中）跟踪哪些页面是空闲的，每个对象都对应于一个物理页面。

那么接下来就进入练习1的内容，我们可以先去看看需要做什么再回过来看代码：

在kern/pmap.c文件中，为以下功能实现代码:

- boot_alloc()
- mem_init()
- page_init()
- page_alloc()
- page_free()

这两个部分的测试函数在 check_page_free_list() 和  check_page_alloc()，也许可以添加一点 assert() 进行验证。

这部分需要做不少了解性的工作，但我觉得帮助比较大的方向还是直接去看相应函数里面的提示和测试用例；毕竟这些写的都已经比较详细了：

先从  boot_alloc() 开始。它是一个简单的物理内存分配器，仅在JOS设置其虚拟内存系统时使用。这里的分配地址，实际上就是简单的更新地址值，在看完注释之后应该很快就可以开始写：

```c
static void *
boot_alloc(uint32_t n)
{
	static char *nextfree;
	char *result;

	if (!nextfree) {
		extern char end[];
		nextfree = ROUNDUP((char *) end, PGSIZE);
	}

	if (n == 0) {
		return nextfree;
	} else if (n > 0) {
		result = nextfree;
		nextfree += ROUNDUP(n, PGSIZE);
		return result;
	}

	return NULL;
}

```

mem_init() 需要我们设置一个两层的页表，实际上这部分的内容不仅仅只包含在物理页面分配中，也包含了lab2余下的部分。我们可以先取消掉 panic 试试看：

很不幸，立马爆个 `Triple fault. ` 出来了...不过还是能得到一部分有用的信息，它可以告诉我们有多少物理内存空间：

```
Physical memory: 131072K available, base = 640K, extended = 130432K
```

接下来我们就继续把这个 panic 取消掉，然后一步步调试。

根据 mem_init() 里面的下一步描述，我们需要使用 boot_alloc 分配一个 struct PageInfo 的数组，这一部分应该也很简单：

```c
pages = (struct PageInfo*)boot_alloc(npages * sizeof(struct PageInfo));
memset(pages, 0, npages * sizeof(struct PageInfo));
```

（注意看对应英文的注释）

下一步就是 page_init() 函数，这一步我觉得它的注释比较混乱，但实际上需要注意的部分就是各个内存片段节点之间的顺序：

我们可以用打印log的方式打印出相关信息查看：

- npages: 32768
- npages_basemem: 160
- PGNUM(PADDR(kern_pgdir)): 279 
- PGNUM(boot_alloc(0)): 344
- PGNUM((void*)EXTPHYSMEM): 256
- PGNUM((void*)IOPHYSMEM): 160

这几个之间一部分是IO的空洞，一部分是内核代码和我们分配记录的page信息，这部分要注意留空不分配；再仔细观察一下 check_page_free_list，尝试测试驱动开发：

（余下的一部分可用的工具类函数记得查询一下相关头文件）

```c
void
page_init(void)
{
	size_t i;
	for (i = 1; i < PGNUM(IOPHYSMEM); i++) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}

	for (i = PGNUM(PADDR(boot_alloc(0))); i < npages; i++) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
	
}
```

接下来的两个函数就很简单了，无非就是链表头结点的插入和删除而已，把它当做一个栈来用：

- page_alloc()

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
	// Fill this function in
	struct PageInfo *result;
	if (page_free_list){
		result = page_free_list;
		page_free_list = page_free_list->pp_link;
		if (alloc_flags & ALLOC_ZERO) {
			memset(page2kva(result),0,PGSIZE);
		}
		result->pp_link = NULL;
		result->pp_ref = 0;
		return result;
	} else {
		return NULL;
	}
}
```

- page_free()

```c
void
page_free(struct PageInfo *pp)
{
	assert(pp->pp_ref == 0);
	assert(!pp->pp_link);

	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

然后目前就可以通过这两个测试用例啦！

## 第2部分：虚拟内存

在进行其他操作之前，请熟悉x86的保护模式内存管理架构：`分段`和`页面转换`（不过我没看）。练习二希望你去阅读一下相关内容。

在x86术语中，`虚拟地址`由段选择器和段内的偏移量组成: 一个线性地址 是段转换之后但页面翻译之前得到的东西，物理地址是转换完全之后得到的。

可以参考下面这张图：

```
           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

在 boot/boot.S中，我们安装了全局描述符表（GDT），该表通过将所有段基址设置为0并将限制设置为来有效地禁用段转换0xffffffff。因此，“选择器”无效，线性地址始终等于虚拟地址的偏移量。在实验3中，我们将需要与分段进行更多的交互才能设置特权级别，但是对于 lab2 内存转换，我们可以在整个JOS实验中忽略分段，而只关注页面转换。

练习3提供了一些帮助性的工具：

- xp 命令在 qemu 里面检查物理内存；
- x 在 gdb 里面可以看虚拟内存；
- info pg 看页表；
- info mem 映射了哪些虚拟地址范围以及具有哪些权限的概述。

