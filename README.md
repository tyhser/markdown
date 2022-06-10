# enable CMA in centos7.8(linux-3.10.0-1127.el7.x86_64)
## environment
不建议用高版本gcc编译centos7.8(linux-3.10.0-1127.el7.x86_64) 内核, 本人使用qemu下的centos7.8编译后传回主机
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)

## 1. put patch reference from [内核补丁参考](https://lkml.kernel.org/lkml/528D4A1A.2050100@zytor.com/T/) on kernel source code

```
---
 arch/x86/Kconfig               | 2 +-
 arch/x86/include/asm/swiotlb.h | 7 +++++++
 arch/x86/kernel/amd_gart_64.c  | 2 +-
 arch/x86/kernel/pci-swiotlb.c  | 9 ++++++---
 arch/x86/pci/sta2x11-fixup.c   | 6 ++----
 include/linux/swiotlb.h        | 2 ++
 lib/swiotlb.c                  | 2 +-
 7 files changed, 20 insertions(+), 10 deletions(-)

diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index e903c71..b15df8b 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -39,7 +39,7 @@ config X86
 	select ARCH_WANT_OPTIONAL_GPIOLIB
 	select ARCH_WANT_FRAME_POINTERS
 	select HAVE_DMA_ATTRS
-	select HAVE_DMA_CONTIGUOUS if !SWIOTLB
+	select HAVE_DMA_CONTIGUOUS
 	select HAVE_KRETPROBES
 	select HAVE_OPTPROBES
 	select HAVE_KPROBES_ON_FTRACE
@@ -66,7 +66,7 @@ config X86
        select HAVE_KVM
        select HAVE_ARCH_KGDB
        select HAVE_ARCH_TRACEHOOK
-       select HAVE_GENERIC_DMA_COHERENT if X86_32
+       select HAVE_GENERIC_DMA_COHERENT
        select HAVE_EFFICIENT_UNALIGNED_ACCESS
        select USER_STACKTRACE_SUPPORT
        select HAVE_REGS_AND_STACK_ACCESS_API
diff --git a/arch/x86/include/asm/swiotlb.h b/arch/x86/include/asm/swiotlb.h
index 977f176..ab05d73 100644
--- a/arch/x86/include/asm/swiotlb.h
+++ b/arch/x86/include/asm/swiotlb.h
@@ -29,4 +29,11 @@ static inline void pci_swiotlb_late_init(void)
 
 static inline void dma_mark_clean(void *addr, size_t size) {}
 
+extern void *x86_swiotlb_alloc_coherent(struct device *hwdev, size_t size,
+					dma_addr_t *dma_handle, gfp_t flags,
+					struct dma_attrs *attrs);
+extern void x86_swiotlb_free_coherent(struct device *dev, size_t size,
+					void *vaddr, dma_addr_t dma_addr,
+					struct dma_attrs *attrs);
+
 #endif /* _ASM_X86_SWIOTLB_H */
diff --git a/arch/x86/kernel/amd_gart_64.c b/arch/x86/kernel/amd_gart_64.c
index b574b29..8e3842f 100644
--- a/arch/x86/kernel/amd_gart_64.c
+++ b/arch/x86/kernel/amd_gart_64.c
@@ -512,7 +512,7 @@ gart_free_coherent(struct device *dev, size_t size, void *vaddr,
 		   dma_addr_t dma_addr, struct dma_attrs *attrs)
 {
 	gart_unmap_page(dev, dma_addr, size, DMA_BIDIRECTIONAL, NULL);
-	free_pages((unsigned long)vaddr, get_order(size));
+	dma_generic_free_coherent(dev, size, vaddr, dma_addr, attrs);
 }
 
 static int gart_mapping_error(struct device *dev, dma_addr_t dma_addr)
diff --git a/arch/x86/kernel/pci-swiotlb.c b/arch/x86/kernel/pci-swiotlb.c
index 6c483ba..77dd0ad 100644
--- a/arch/x86/kernel/pci-swiotlb.c
+++ b/arch/x86/kernel/pci-swiotlb.c
@@ -14,7 +14,7 @@
 #include <asm/iommu_table.h>
 int swiotlb __read_mostly;
 
-static void *x86_swiotlb_alloc_coherent(struct device *hwdev, size_t size,
+void *x86_swiotlb_alloc_coherent(struct device *hwdev, size_t size,
 					dma_addr_t *dma_handle, gfp_t flags,
 					struct dma_attrs *attrs)
 {
@@ -28,11 +28,14 @@ static void *x86_swiotlb_alloc_coherent(struct device *hwdev, size_t size,
 	return swiotlb_alloc_coherent(hwdev, size, dma_handle, flags);
 }
 
-static void x86_swiotlb_free_coherent(struct device *dev, size_t size,
+void x86_swiotlb_free_coherent(struct device *dev, size_t size,
 				      void *vaddr, dma_addr_t dma_addr,
 				      struct dma_attrs *attrs)
 {
-	swiotlb_free_coherent(dev, size, vaddr, dma_addr);
+	if (is_swiotlb_buffer(dma_to_phys(dev, dma_addr)))
+		swiotlb_free_coherent(dev, size, vaddr, dma_addr);
+	else
+		dma_generic_free_coherent(dev, size, vaddr, dma_addr, attrs);
 }
 
 static struct dma_map_ops swiotlb_dma_ops = {
diff --git a/arch/x86/pci/sta2x11-fixup.c b/arch/x86/pci/sta2x11-fixup.c
index 9d8a509..5ceda85 100644
--- a/arch/x86/pci/sta2x11-fixup.c
+++ b/arch/x86/pci/sta2x11-fixup.c
@@ -173,9 +173,7 @@ static void *sta2x11_swiotlb_alloc_coherent(struct device *dev,
 {
 	void *vaddr;
 
-	vaddr = dma_generic_alloc_coherent(dev, size, dma_handle, flags, attrs);
-	if (!vaddr)
-		vaddr = swiotlb_alloc_coherent(dev, size, dma_handle, flags);
+	vaddr = x86_swiotlb_alloc_coherent(dev, size, dma_handle, flags, attrs);
 	*dma_handle = p2a(*dma_handle, to_pci_dev(dev));
 	return vaddr;
 }
@@ -183,7 +181,7 @@ static void *sta2x11_swiotlb_alloc_coherent(struct device *dev,
 /* We have our own dma_ops: the same as swiotlb but from alloc (above) */
 static struct dma_map_ops sta2x11_dma_ops = {
 	.alloc = sta2x11_swiotlb_alloc_coherent,
-	.free = swiotlb_free_coherent,
+	.free = x86_swiotlb_free_coherent,
 	.map_page = swiotlb_map_page,
 	.unmap_page = swiotlb_unmap_page,
 	.map_sg = swiotlb_map_sg_attrs,
diff --git a/include/linux/swiotlb.h b/include/linux/swiotlb.h
index a5ffd32..e7a018e 100644
--- a/include/linux/swiotlb.h
+++ b/include/linux/swiotlb.h
@@ -116,4 +116,6 @@ static inline void swiotlb_free(void) { }
 #endif
 
 extern void swiotlb_print_info(void);
+extern int is_swiotlb_buffer(phys_addr_t paddr);
+
 #endif /* __LINUX_SWIOTLB_H */
diff --git a/lib/swiotlb.c b/lib/swiotlb.c
index fe978e0..6e4a798 100644
--- a/lib/swiotlb.c
+++ b/lib/swiotlb.c
@@ -369,7 +369,7 @@ void __init swiotlb_free(void)
 	io_tlb_nslabs = 0;
 }
 
-static int is_swiotlb_buffer(phys_addr_t paddr)
+int is_swiotlb_buffer(phys_addr_t paddr)
 {
 	return paddr >= io_tlb_start && paddr < io_tlb_end;
 }
diff --git a/drivers/base/dma-contiguous.c b/drivers/base/dma-contiguous.c
index ...
---a/drivers/base/dma-contiguous.c
+++a/drivers/base/dma-contiguous.c
@@ -288,7 +288,7 @@ struct page *dma_alloc_from_contiguous(struct device *dev, size_t cou
                align = CONFIG_CMA_ALIGNMENT;

        pr_debug("%s(cma %p, count %d, align %d)\n", __func__, (void *)cma,
-                count, align);
+                (int)count, align);

        if (!count)
                return NULL;
diff --git a/arch/x86/kernel/pci-dma.c b/arch/x86/kernel/pci-dma.c
index 77a4e62..bc8d25e 100644
--- a/arch/x86/kernel/pci-dma.c
+++ b/arch/x86/kernel/pci-dma.c
@@ -147,14 +150,20 @@ void dma_generic_free_coherent(struct device *dev, size_t size, void *vaddr,
 bool arch_dma_alloc_attrs(struct device **dev, gfp_t *gfp)
 {
        if (!*dev)
                *dev = &x86_dma_fallback_dev;

        *gfp &= ~(__GFP_DMA | __GFP_HIGHMEM | __GFP_DMA32);
        *gfp = dma_alloc_coherent_gfp_flags(*dev, *gfp);

-       if (!is_device_dma_capable(*dev))
+#if 0
+       if (!is_device_dma_capable(*dev)) {
                return false;
+       }
+#endif
+
        return true;

 }

```
## 2. Make sure configure parameters correct
	make sure:
```
	CONFIG_DMA_CMA=y
	CONFIG_CMA=y

#Default contiguous memory area size:
	CONFIG_CMA_SIZE_MBYTES=160
	CONFIG_CMA_SIZE_SEL_MBYTES=y
# CONFIG_CMA_SIZE_SEL_PERCENTAGE is not set
# CONFIG_CMA_SIZE_SEL_MIN is not set
# CONFIG_CMA_SIZE_SEL_MAX is not set
	CONFIG_CMA_ALIGNMENT=8
#这个配置项可能决定了你最大能分派的内存大小,即使分配size很大,有可能返回的size只有极限大小
	CONFIG_CMA_AREAS=70

```
## 3. compile kernel
after `make menuconfig` then `make bzImage -j8`
bzImage show in ${kernel_path}/arch/x86/boot/bzImage
** you can reference artical to build a qemu environment**==>[QEMU+GDB调试Linux内核总结](https://blog.csdn.net/M120674/article/details/118856793?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118856793-blog-80967912.pc_relevant_blogantidownloadv1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-118856793-blog-80967912.pc_relevant_blogantidownloadv1&utm_relevant_index=1)

## 4. pass cmdline to set cma size
	cma=nn[MG]	[ARM,KNL]
			Sets the size of kernel global memory area for contiguous
			memory allocations. For more information, see
			include/linux/dma-contiguous.h
	my qemu cmdline:
```
DIR=`pwd`
qemu-system-x86_64 \
     -m 5G\
    -enable-kvm \
    -append "nokaslr cma=1000M console=ttyS0" \
    -kernel bzImage \
    -initrd ${DIR}/initramfs.img \
    -nographic  \
#    -S -s \

```

## 5. test module
reference kernel module helper for testing CMA==> [simple kernel module as the helper to test CMA](https://lwn.net/Articles/485193/)

## 6. check memory alloction station
`cat /proc/meminfo |grep Cma`

邰佳俊 (tel:18020521131)
