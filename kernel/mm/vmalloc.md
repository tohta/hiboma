# mm/vmalloc.c

## USAGE

```c
        void *p = vmalloc(PAGE_SIZE * num_pages);
```

## DEFINITION

```c
/**
 *	vmalloc  -  allocate virtually contiguous memory
 *
 *	@size:		allocation size
 *
 *	Allocate enough pages to cover @size from the page level
 *	allocator and map them into contiguous kernel virtual space.
 *
 *	For tight cotrol over page level allocator and protection flags
 *	use __vmalloc() instead.
 */
void *vmalloc(unsigned long size)
{
       return __vmalloc(size, GFP_KERNEL | __GFP_HIGHMEM, PAGE_KERNEL);
}
```

 * カーネル仮想空間で連続したリニアアドレスを割り当てる
 * 仮想サイズ分のページも割り当て
 * sleep しうるので、割り込みコンテキスト、クリティカルセクションで使えない

## __vmalloc

 ```c
 /**
 *	__vmalloc  -  allocate virtually contiguous memory
 *
 *	@size:		allocation size
 *	@gfp_mask:	flags for the page level allocator
 *	@prot:		protection mask for the allocated pages
 *
 *	Allocate enough pages to cover @size from the page level
 *	allocator with @gfp_mask flags.  Map them into contiguous
 *	kernel virtual space, using a pagetable protection of @prot.
 */
void *__vmalloc(unsigned long size, int gfp_mask, pgprot_t prot)
{
	struct vm_struct *area;
	struct page **pages;
	unsigned int nr_pages, array_size, i;

    /* ページ境界にアライン */
	size = PAGE_ALIGN(size);

    /* 物理ページ数を超えてないかどうか */
	if (!size || (size >> PAGE_SHIFT) > num_physpages)
		return NULL;

    /* vm_struct を割り当てする */
	area = get_vm_area(size, VM_ALLOC);
	if (!area)
		return NULL;

    /* 要求するサイズをページ数に変換 */
	nr_pages = size >> PAGE_SHIFT;
    /* vm_struct のサイズ? */
	array_size = (nr_pages * sizeof(struct page *));

	area->nr_pages = nr_pages;

    /* ? */
	if (array_size > PAGE_SIZE)
		pages = __vmalloc(array_size, gfp_mask, PAGE_KERNEL);
	else
   /* ページサイズ以下なら kmalloc を変わりに呼ぶ */
		pages = kmalloc(array_size, (gfp_mask & ~__GFP_HIGHMEM));
	area->pages = pages;
	if (!area->pages) {
		remove_vm_area(area->addr);
		kfree(area);
		return NULL;
	}
	memset(area->pages, 0, array_size);

	for (i = 0; i < area->nr_pages; i++) {
        /* ページの割り当て。 alloc_page の繰り返し => 非連続のページ */
		area->pages[i] = alloc_page(gfp_mask);
		if (unlikely(!area->pages[i])) {
			/* Successfully allocated i pages, free them in __vunmap() */
			area->nr_pages = i;
			goto fail;
		}
	}

    /* ページを initプロセスの PMD にぶら下げる */
	if (map_vm_area(area, prot, &pages))
		goto fail;
	return area->addr;

fail:
	vfree(area->addr);
	return NULL;
}

EXPORT_SYMBOL(__vmalloc);
```

## get_vm_area

```c
/**
 *	get_vm_area  -  reserve a contingous kernel virtual area
 *
 *	@size:		size of the area
 *	@flags:		%VM_IOREMAP for I/O mappings or VM_ALLOC
 *
 *	Search an area of @size in the kernel virtual mapping area,
 *	and reserved it for out purposes.  Returns the area descriptor
 *	on success or %NULL on failure.
 */
struct vm_struct *get_vm_area(unsigned long size, unsigned long flags)
{
	return __get_vm_area(size, flags, VMALLOC_START, VMALLOC_END);
}
```

### VMALLOC_START, VMALLOC_END

#### i386

high_memory と vmalloc_earlyreserve のサイズで可変になる

```c
#define VMALLOC_START	(((unsigned long) high_memory + vmalloc_earlyreserve + \
			2*VMALLOC_OFFSET-1) & ~(VMALLOC_OFFSET-1))

# define VMALLOC_END	(PKMAP_BASE-2*PAGE_SIZE)
```

#### x86_64

固定のアドレスになっている

```c
#define VMALLOC_START    0xffffff0000000000UL
#define VMALLOC_END      0xffffff7fffffffffUL
```

__get_vm_area で vm_struct を確保する ( vm_area_struct とは違うので注意 )。

vm_struct の定義はこんなん

```c
struct vm_struct {
	void			*addr;
	unsigned long		size;
	unsigned long		flags;
	struct page		**pages;
	unsigned int		nr_pages;
	unsigned long		phys_addr;
	struct vm_struct	*next;
};
```

__get_vm_area の実装を追って行く

 * start - end のアドレス内で vm_sturct を割り当てる
 * start - end (VmallocTotal?) で収まらなきゃ、"allocation failed: out of vmalloc space - use vmalloc=<size> to increase size.\n" を出す

のが肝だろうか 

```c
struct vm_struct *__get_vm_area(unsigned long size, unsigned long flags,
				unsigned long start, unsigned long end)
{
    /* start = VMALLOC_START,  end   = VMALLOC_END   */
	struct vm_struct **p, *tmp, *area;
	unsigned long align = 1;
	unsigned long addr;

	if (flags & VM_IOREMAP) {
		int bit = fls(size);

		if (bit > IOREMAP_MAX_ORDER)
			bit = IOREMAP_MAX_ORDER;
		else if (bit < PAGE_SHIFT)
			bit = PAGE_SHIFT;

		align = 1ul << bit;
	}

    /* アラインされた開始のアドレス */
	addr = ALIGN(start, align);

	area = kmalloc(sizeof(*area), GFP_KERNEL);
	if (unlikely(!area))
		return NULL;

	/*
	 * We always allocate a guard page.
	 */
	size += PAGE_SIZE;
	if (unlikely(!size)) {
		kfree (area);
		return NULL;
	}

	write_lock(&vmlist_lock);
    /* vmlist (strcut vm_struct) がグローバル変数。 vmalloc 用の vm_struct リスト */
    /* start 〜 end 内で収まるように走査? */
	for (p = &vmlist; (tmp = *p) != NULL ;p = &tmp->next) {
		if ((unsigned long)tmp->addr < addr) {
			if((unsigned long)tmp->addr + tmp->size >= addr)
				addr = ALIGN(tmp->size + 
					     (unsigned long)tmp->addr, align);
			continue;
		}
		if ((size + addr) < addr)
			goto out;
		if (size + addr <= (unsigned long)tmp->addr)
			goto found;
		addr = ALIGN(tmp->size + (unsigned long)tmp->addr, align);
		if (addr > end - size)
			goto out;
	}

found:
	area->next = *p;
	*p = area;

	area->flags = flags;
	area->addr = (void *)addr;
	area->size = size;
	area->pages = NULL;
	area->nr_pages = 0;
	area->phys_addr = 0;
	write_unlock(&vmlist_lock);

	return area;

out:
	write_unlock(&vmlist_lock);
	kfree(area);

    /* vm_struct が足らなくなったら warning */
	if (printk_ratelimit())
		printk(KERN_WARNING "allocation failed: out of vmalloc space - use vmalloc=<size> to increase size.\n");
	return NULL;
}
```

## map_vm_area

vm_struct を init_mm? の PGD にぶら下げる?

```c
int map_vm_area(struct vm_struct *area, pgprot_t prot, struct page ***pages)
{
	unsigned long address = (unsigned long) area->addr;
	unsigned long end = address + (area->size-PAGE_SIZE);
	pgd_t *dir;
	int err = 0;

	dir = pgd_offset_k(address);
	spin_lock(&init_mm.page_table_lock);
	do {
		pmd_t *pmd = pmd_alloc(&init_mm, dir, address);
		if (!pmd) {
			err = -ENOMEM;
			break;
		}
		if (map_area_pmd(pmd, address, end - address, prot, pages)) {
			err = -ENOMEM;
			break;
		}

		address = (address + PGDIR_SIZE) & PGDIR_MASK;
		dir++;
	} while (address && (address < end));

	spin_unlock(&init_mm.page_table_lock);
	flush_cache_vmap((unsigned long) area->addr, end);
	return err;
}
```

## /proc/meminfo

```
VmallocTotal:   106488 kB
VmallocUsed:      2660 kB
VmallocChunk:   103396 kB
```

```
		(unsigned long)VMALLOC_TOTAL >> 10,
		vmi.used >> 10,
		vmi.largest_chunk >> 10
```

## vmalloc の ページフォルト

page_fault -> do_page_fault -> __do_page_fault の vmalloc_fault でハンドリングされる

```c
	/*
	 * We fault-in kernel-space virtual memory on-demand. The
	 * 'reference' page table is init_mm.pgd.
	 *
	 * NOTE! We MUST NOT take any locks for this case. We may
	 * be in an interrupt or a critical region, and should
	 * only copy the information from the master page table,
	 * nothing more.
	 *
	 * This verifies that the fault happens in kernel space
	 * (error_code & 4) == 0, and that the fault was not a
	 * protection error (error_code & 9) == 0.
	 */
	if (unlikely(fault_in_kernel_space(address))) {
		if (!(error_code & (PF_RSVD | PF_USER | PF_PROT))) {
			if (vmalloc_fault(address) >= 0)
				return;

			if (kmemcheck_fault(regs, address, error_code))
				return;
		}

		/* Can handle a stale RO->RW TLB: */
```

#### vmalloc_fault

 * カーネル空間でのフォルト
 * フォルトしたアドレスが VMALLOC_START 〜 VMALLOC_END の範囲かどうかを見る
 * read_cr3 で CR3レジスタ = PGD の物理アドレスを取る

```c
/*
 * 32-bit:
 *
 *   Handle a fault on the vmalloc or module mapping area
 */
static noinline int vmalloc_fault(unsigned long address)
{
	unsigned long pgd_paddr;
	pmd_t *pmd_k;
	pte_t *pte_k;

	/* Make sure we are in vmalloc area: */
	if (!(address >= VMALLOC_START && address < VMALLOC_END))
		return -1;

	/*
	 * Synchronize this task's top level page-table
	 * with the 'reference' page table.
	 *
	 * Do _not_ use "current" here. We might be inside
	 * an interrupt in the middle of a task switch..
	 */
	pgd_paddr = read_cr3();

    /* PGD -> PMD -> ... で辿っていく */
	pmd_k = vmalloc_sync_one(__va(pgd_paddr), address);
	if (!pmd_k)
		return -1;

	pte_k = pte_offset_kernel(pmd_k, address);
	if (!pte_present(*pte_k))
		return -1;

	return 0;
}
```