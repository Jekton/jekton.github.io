---
title: Linux 内核页表的创建
date: 2018-11-18 18:49:38
categories: Linux
tags: Linux
description: 内核跟普通的应用一样，为了使用虚拟内存，也需要一个给 CPU 设置一个页表。在这篇文章中，我们就一起来了解 Linux 是如何为内核创建页表的。
---

> 源码使用 Linux 2.6.24，基于 x86 平台；参考书是《深入理解 LINUX 内核》第三版

内核跟普通的应用一样，为了使用虚拟内存，也需要一个给 CPU 设置一个页表。在这篇文章中，我们就一起来了解 Linux 是如何为内核创建页表的。需要注意的是，这里我并不打算详细讲解页表的方方面面，硬件相关的基础知识，读者可以参考《深入理解LINUX内核》第3版第2章。本文的目的在于，作为该书的补充，基于真实的源码来讲解这一过程。


# 临时内核页表的构造

x86 系统刚刚启动时候运行在实模式下，这个时候线性地址就是物理地址。为了进入 32 位保护模式，首先就要启用分页（paging）。这就要求我们构建一个页表；这张页表把线性地址映射转换为物理地址。由于不同的计算机的配置不一样，他们需要的页表大小、页表个数也都不一样，所以需要在运行时动态分配页表，这就要求我们具有动态内存分配能力。

为了解决构造页表时候的鸡生蛋蛋生鸡问题，Linux 使用了一个临时的内核页表。它只有两个页表（这里的页表指的是用来索引页框的最后一级页表）。在不启用 PAE (Page Addression Extension) 和 PSE（Page Size Extension）的情况下，一个页表可以指向 `10^2 = 1024` 个内存页，一个内存页 4K，所以两个页表允许我们索引 8M 的内存。

顶层的页目录（page directory）使用全局变量 `swapper_pg_dir` 定义，下面是它的声明：
```C
// ${linux_source}/include/asm-x86/pgtable_32.h

// empty_zero_page 在后面也会用到，这里就一并列出来了
extern unsigned long empty_zero_page[1024];
extern pgd_t swapper_pg_dir[1024];
```
他在 `head_32.S` 里面定义的：
```x86
# ${linux_source}/arch/x86/kernel/head_32.S

/*
 * BSS section
 */
.section ".bss.page_aligned","wa"
	.align PAGE_SIZE_asm
ENTRY(swapper_pg_dir)
	.fill 1024,4,0
ENTRY(swapper_pg_pmd)
	.fill 1024,4,0
ENTRY(empty_zero_page)
	.fill 4096,1,0
```
这里的 `.fill 1024,4,0` 的意思是用 0 填充 1024 个 4 byte 长度的内存（一个页目录项（page table entry）的大小是 32 bit）。

接下来是变量 `pg0`：
```C
// ${linux_source}/include/asm-x86/pgtable_32.h

/* The boot page tables (all created as a single array) */
extern unsigned long pg0[];
```

`pg0` 通过指示链接器，放在了 bss 段的后面。
```
SECTIONS
{
  /* 前面那些都略去了 */

  .bss : AT(ADDR(.bss) - LOAD_OFFSET) {
	__init_end = .;
	__bss_start = .;		/* BSS */
	*(.bss.page_aligned)
	*(.bss)
	. = ALIGN(4);
	__bss_stop = .;
  	_end = . ;
	/* This is where the kernel creates the early boot page tables */
	. = ALIGN(4096);
	pg0 = . ;
  }

  /* ... */
}
```

有了 `swapper_pg_dir` 和 `pg0` 后，接下来的工作就是对它们进行初始化。此时还处于实模式下，这部分工作是由汇编代码完成的。
```assembly
# ${linux_source}/arch/x86/kernel/head_32.S

/*
 * Initialize page tables.  This creates a PDE and a set of page
 * tables, which are located immediately beyond _end.  The variable
 * init_pg_tables_end is set up to point to the first "safe" location.
 * Mappings are created both at virtual address 0 (identity mapping)
 * and PAGE_OFFSET for up to _end+sizeof(page tables)+INIT_MAP_BEYOND_END.
 *
 * Warning: don't use %esi or the stack in this code.  However, %esp
 * can be used as a GPR if you really need it...
 */
# __PAGE_OFFSET 是 0xc000 0000，所以 page_pde_offset 是 0xc00
page_pde_offset = (__PAGE_OFFSET >> 20);

default_entry:
	# __PAGE_OFFSET 是 3G，pg0 是虚拟地址，减去 __PAGE_OFFSET 后就得到了
	# pg0 的物理地址。我们把 pg0 的物理地址放在了 edi 寄存器里
	movl $(pg0 - __PAGE_OFFSET), %edi
	# 同理，这里把 swapper_pg_dir 的物理地址放在 edx
	movl $(swapper_pg_dir - __PAGE_OFFSET), %edx
	# page directory/table entry 的低 12 位都是一些标志物，各个位代表的含义
	# 读者可以参考 https://wiki.osdev.org/Paging 或者书中的第 52 页
	movl $0x007, %eax			/* 0x007 = PRESENT+RW+USER */
10:
	# 下面这两行代码对熟悉 C 语言的读者可能会造成一定的困扰。如果从 C 语言的角度
	# 来看，它们是把地址 &pg0 + 7 放到了 swapper_pg_dir 的第一项；但问题在于，
	# 为什么要 +7？
	# 其实这里的 7 和前面那个 7 一样，指的是页目录项的标志物 PRESENT+RW+USER，
	# pg0 的地址是 4K 对齐的，这意味着他的地址的低 12 位都为 0，加上 7 以后，刚
	# 好就是我们所需要的页目录项的值。
	leal 0x007(%edi),%ecx			/* Create PDE entry */
	movl %ecx,(%edx)			/* Store identity PDE entry */
	# 书里有说明，我们要把 0x0000 0000 ~ 0x007f ffff 和 0xc000 0000 ~ 0xc07f ffff
	# 都映射到物理地址 0x0000 0000 ~ 0x007f ffff，下面这一行设置的 0xc000 0000
	# 对应的页目录项。
	# 这里的问题在于，按照书里的说明，我们应该设置的是第 0x300 项，这里是加上的却是 0xc00。
	# 这里需要提一下平时用 C 语言时编译器帮我们做的事。当我们写下 int *p = NULL; p+2
	# 的时候，编译器知道 int 是 4 个字节，所以 p+2 会汇编代码里面是 +8。
	# 一个 PDE 也是 32 位，所以真正的偏移量是 0x300 << 2 = 0xc00
	movl %ecx,page_pde_offset(%edx)		/* Store kernel PDE entry */
	# edx + 4 以后，就是下一个页目录项了，下个循环将会继续初始化（一共两个页目录项）
	addl $4,%edx
	# 一个页表有 1024 个页表项，这里初始化一个在接下来的循环里面用到的计数器
	movl $1024, %ecx
11:
	# stosl 把 %eax 的内容复制到物理地址 ES:EDI，也就是 pg0 处；并且 %edi + 4
	stosl
	# 加上 0x1000 后，%eax 指向下一个页
	addl $0x1000,%eax
	# %ecx -= 1，如果 %ecx 不为 0，跳转到 11 处。这里总共会循环 1024 次，初始化 1024 个页表项。
	loop 11b
	/* End condition: we must map up to and including INIT_MAP_BEYOND_END */
	/* bytes beyond the end of our own page tables; the +0x007 is the attribute bits */
	leal (INIT_MAP_BEYOND_END+0x007)(%edi),%ebp
	cmpl %ebp,%eax
	jb 10b
	# 到这里的时候，%edi 的值是我们映射的最后一个页表项的地址，这里我们把它存到变量
	# init_pg_tables_end 里。init_pg_tables_end 在 setup_32.c 里定义
	movl %edi,(init_pg_tables_end - __PAGE_OFFSET)

	# 下面是固定映射的，这部分就先不看了
	/* Do an early initialization of the fixmap area */
	movl $(swapper_pg_dir - __PAGE_OFFSET), %edx
	movl $(swapper_pg_pmd - __PAGE_OFFSET), %eax
	addl $0x67, %eax			/* 0x67 == _PAGE_TABLE */
	movl %eax, 4092(%edx)

	xorl %ebx,%ebx				/* This is the boot CPU (BSP) */
	jmp 3f
```

前面代码的最后一行是一个 `jmp 3f`，下面，我们就看看这个 `3` 处的代码。

# 启用分页

构建好临时内核页表后，接下来就该启用分页了。
```
# ${linux_source}/arch/x86/kernel/head_32.S

3:
/*
 * Enable paging
 */
	movl $swapper_pg_dir-__PAGE_OFFSET,%eax
	# %cr3 寄存器存放的是页表的地址
	movl %eax,%cr3		/* set the page table pointer.. */
	movl %cr0,%eax
	# cr0 的最高位是 Paging 位，置 1 后启用分页
	# 关于 cr0，参考 https://en.wikipedia.org/wiki/Control_register#CR0
	orl $0x80000000,%eax
	movl %eax,%cr0		/* ..and set paging (PG) bit */
```

CPU 的分页机制现在已经启用了，但是我们的页表还是不完整的，剩下部分将会使用 C 语言来完成。

# 构建线性地址的内核页表

完整的页表构建是从函数 `pagetable_init` 开始的：
```C
// ${linux_source}/arch/x86/mm/init_32.S

static void __init pagetable_init (void)
{
	unsigned long vaddr, end;
	pgd_t *pgd_base = swapper_pg_dir;

	/* Enable PSE if available */
	if (cpu_has_pse)
		set_in_cr4(X86_CR4_PSE);

	/* Enable PGE if available */
	if (cpu_has_pge) {
		set_in_cr4(X86_CR4_PGE);
		__PAGE_KERNEL |= _PAGE_GLOBAL;
		__PAGE_KERNEL_EXEC |= _PAGE_GLOBAL;
	}

	kernel_physical_mapping_init(pgd_base);

	// 下面是固定映射相关的内容，这里就先忽略了
}
```

实际的页表构建是在函数 `kernel_physical_mapping_init` 完成的：
```C
// ${linux_source}/arch/x86/mm/init_32.c

/*
 * This maps the physical memory to kernel virtual address space, a total 
 * of max_low_pfn pages, by creating page tables starting from address 
 * PAGE_OFFSET.
 */
static void __init kernel_physical_mapping_init(pgd_t *pgd_base)
{
	unsigned long pfn;
	pgd_t *pgd;
	pmd_t *pmd;
	pte_t *pte;
	int pgd_idx, pmd_idx, pte_ofs;

	// PAGE_OFFSET 是 0xc000 0000，这里拿的内核虚拟地址第一项对应的 pgd 的 index
	pgd_idx = pgd_index(PAGE_OFFSET);
	pgd = pgd_base + pgd_idx;
	pfn = 0;	// pfn 代表 page frame number

	// 初始化 pgd。pgd 的项数由 PTRS_PER_PGD 定义，在最普通的情况下，它是 1024。
	// 如果启用了 PAE，则等于 4
	for (; pgd_idx < PTRS_PER_PGD; pgd++, pgd_idx++) {
		// 32 位的系统一般是 2 级页表结构（为什么说它是一般，读者后面就会知道了）
		// 每个 pgd 项都指向一个 pmd，one_md_table_init 初始化一个 pmd。
		// 建议读者这里先跳过本函数后面部分，看完 one_md_table_init 再回过头来继续往下看
		pmd = one_md_table_init(pgd);
		// max_low_pfn 是被内核直接映射的最后一个页框的页框号，参考书中第 72 页
		if (pfn >= max_low_pfn)
			// 超过 max_low_pfn 的 pte 可以不初始化，但 pmd 必须初始化，所以用 continue
			continue;
		// 对不启用 PAE 的系统来说，这里的 pmd 就是 pgd，PTRS_PER_PMD 等于 1。
		// 如果启用 PAE，PTRS_PER_PMD 等于 512。
		// 这里的 pmd 相当于页目录（Page Directory）,下面的循环里初始化每个页目录项（每个页目录项
		// 指向一个页表项）
		for (pmd_idx = 0; pmd_idx < PTRS_PER_PMD && pfn < max_low_pfn; pmd++, pmd_idx++) {
			// address 是当前（物理）页框开头对应的虚拟地址
			unsigned int address = pfn * PAGE_SIZE + PAGE_OFFSET;

			/* Map with big pages if possible, otherwise create normal page tables. */
			if (cpu_has_pse) {
				// pfn + PTRS_PER_PTE - 1 是当前 pmd 能够索引的最大的页框号
				// * PAGE_SIZE + PAGE_OFFSET + (PAGE_SIZE-1) 就是当前 pmd 做能够指向的最大的
				// 地址。也就是说，pmd 的地址范围是 [address, address2]
				unsigned int address2 = (pfn + PTRS_PER_PTE - 1) * PAGE_SIZE + PAGE_OFFSET + PAGE_SIZE-1;
				if (is_kernel_text(address) || is_kernel_text(address2))
					// pmd 包含了内核的 text 段，所以加上了 exec 标记
					set_pmd(pmd, pfn_pmd(pfn, PAGE_KERNEL_LARGE_EXEC));
				else
					set_pmd(pmd, pfn_pmd(pfn, PAGE_KERNEL_LARGE));

				// 启用 PSE 后就不需要 pte 了。
				// 对于启用了 PAE 的机器来说，一页是 2^(9+12) = 2M
				// 没有 PAE 则是 2^(10+12) = 4M
				pfn += PTRS_PER_PTE;
			} else {
				pte = one_page_table_init(pmd);

				for (pte_ofs = 0;
				     pte_ofs < PTRS_PER_PTE && pfn < max_low_pfn;
				     pte++, pfn++, pte_ofs++, address += PAGE_SIZE) {
					if (is_kernel_text(address))
						set_pte(pte, pfn_pte(pfn, PAGE_KERNEL_EXEC));
					else
						set_pte(pte, pfn_pte(pfn, PAGE_KERNEL));
				}
			}
		}
	}
}


/*
 * Creates a middle page table and puts a pointer to it in the
 * given global directory entry. This only returns the gd entry
 * in non-PAE compilation mode, since the middle layer is folded.
 */
static pmd_t * __init one_md_table_init(pgd_t *pgd)
{
	pud_t *pud;
	pmd_t *pmd_table;
		
#ifdef CONFIG_X86_PAE
	if (!(pgd_val(*pgd) & _PAGE_PRESENT)) {
		// 启用 PAE 的情况下，32 bit 的虚拟地址分为 2 9 9 12，pgd 有
		// 2^2 = 4 项；pmd 是 2^9 = 512 项；然后是 pte 2^9 = 512 项；
		// pte 在 kernel_physical_mapping_init 中初始化。
		// PAE 相关知识参考书上第 56 页

		// bootmem 相关的后面昨晚单独的一篇文章来讲述，这里假装内存被
		// 神奇地分配出来就好
		pmd_table = (pmd_t *) alloc_bootmem_low_pages(PAGE_SIZE);

		// 虚拟化相关的东西，忽略就好
		paravirt_alloc_pd(__pa(pmd_table) >> PAGE_SHIFT);
		set_pgd(pgd, __pgd(__pa(pmd_table) | _PAGE_PRESENT));
		pud = pud_offset(pgd, 0);
		if (pmd_table != pmd_offset(pud, 0))
			BUG();
	}
#endif
	// 在不启用 PAE 的情况下，下面返回的 pmd_table 其实就是 pgd（也就是
	// 直接从 pgd 到 pte，两者都是 2^10 = 1024 项）
	pud = pud_offset(pgd, 0);
	pmd_table = pmd_offset(pud, 0);
	return pmd_table;
}


// 这个函数就比较平凡了，没有什么好说的
/*
 * Create a page table and place a pointer to it in a middle page
 * directory entry.
 */
static pte_t * __init one_page_table_init(pmd_t *pmd)
{
	if (!(pmd_val(*pmd) & _PAGE_PRESENT)) {
		pte_t *page_table = NULL;

#ifdef CONFIG_DEBUG_PAGEALLOC
		page_table = (pte_t *) alloc_bootmem_pages(PAGE_SIZE);
#endif
		if (!page_table)
			page_table =
				(pte_t *)alloc_bootmem_low_pages(PAGE_SIZE);

		paravirt_alloc_pt(&init_mm, __pa(page_table) >> PAGE_SHIFT);
		set_pmd(pmd, __pmd(__pa(page_table) | _PAGE_TABLE));
		BUG_ON(page_table != pte_offset_kernel(pmd, 0));
	}

	return pte_offset_kernel(pmd, 0);
}
```

这部分代码其实有 4 中情况：有 PAE 和没有 PAE两种，这两种又分别有 PSE 启不启用两种情况。读者可以分情况一个一个看，分情况弄清楚后，再合并一起看。


# 固定映射的线性地址、非连续内存区的线性地址

处于篇幅和学习目的考虑，固定映射、非连续内存的处理在这里就先略去了，以后有机会再单独开一篇文章补上。内核页表的创建相关的代码我们就先看到这里。

