# ramoops

## I. 介绍
Ramoops是一个oops/panic记录器，在系统崩溃之前将其日志写入RAM。它通过在循环缓冲区中记录oopses和panics来工作。
Ramoops需要一个具有持久RAM的系统，以便重新启动后该区域的内容能够存活下来。

## II. Ramoops概念
Ramoops使用一个预定义的内存区域来存储dump。内存区域的开始、大小和类型使用3个变量进行设置：
* `mem_address`定义起始地址
* `mem_size`定义大小，以2的幂次对齐
* `mem_type`定义类型，默认为`pgprot_writecombine`

通常，应使用`mem_type=0`的默认值，因为该值将`pstore`映射设置为`pgprot_writecombine`。设置`mem_type=1`将尝试使用
`pgprot_noncached`，它只在某些平台上工作，这是因为`pstore`依赖于原子操作。至少在ARM上，`pgprot_noncached`会导致内存
被映射为强顺序的，并且强顺序内存上的原子操作是实现定义的，并且不会在许多ARM（如OMAP）上工作。

内存区域被划分为以`record_size`定义的数量个块，每个oops/panic写一个块的信息

将`dump_oops`设置为1将会使oops/panic都输出，设置为0，只有panic输出

模块使用计数器记录多个输出，在重新启动时被重置

Ramoops还支持对持久内存区域的软件ECC保护。当使用硬件重置使机器恢复使用时(看门狗触发)，这可能很有用。在这种情况下，
RAM可能有些损坏，但通常是可恢复的。

## III. 参数设置
设置Ramoops参数可以采用两种不同的方式：
### 使用模块参数(上文描述的变量名称)
为了快速调试，您还可以在引导期间保留部分内存，然后使用为ramoops保留的内存。
例如，假设一台内存大于128MB的机器，下面的内核命令行将告诉内核只使用前128MB的内存，并将ECC保护的Ramoops区域置于
128MB边界处：
```txt
"mem=128M ramoops.mem_address=0x8000000 ramoops.ecc=1"
```
### 使用platform设备并设置pltform数据
可以通过平台数据设置参数，例如：
```c
#include <linux/pstore_ram.h> [...]
static struct ramoops_platform_data ramoops_data = { .mem_size               = <...>,
        .mem_address            = <...>,
        .mem_type               = <...>,
        .record_size            = <...>,
        .dump_oops              = <...>,
        .ecc                    = <...>, };
static struct platform_device ramoops_dev = { .name = "ramoops",
        .dev = { .platform_data = &ramoops_data, }, };
[... inside a function ...] int ret;
ret = platform_device_register(&ramoops_dev); if (ret) { printk(KERN_ERR "unable to register platform device\n");
	return ret; }
```

您可以指定RAM内存或外围设备的内存。但是，在指定RAM时，请确保在体系结构代码的早期发出memblock_reserve()来保留内存，例如：
```c
#include <linux/memblock.h>
memblock_reserve(ramoops_data.mem_address, ramoops_data.mem_size);
```
## IV. 输出格式
输出以一个头开始，当前定义为“===”，后跟一个时间戳和一行新行

## V. 读数据
可以从pstore文件系统读取输出的内容。这些文件的格式是“dmesg-ramoops-n”，其中n是内存中的记录编号。要从RAM中删除存储的记录，
只需取消链接相应的pstore文件。

## VI. 持久函数跟踪
持久性函数跟踪对于调试软件或硬件相关挂起可能很有用。函数调用链日志存储在“ftrace ramoops”文件中。以下是用法示例：
```shell
 # mount -t debugfs debugfs /sys/kernel/debug/
 # echo 1 > /sys/kernel/debug/pstore/record_ftrace
 # reboot -f
 [...]
 # mount -t pstore pstore /mnt/
 # tail /mnt/ftrace-ramoops 
 0 ffffffff8101ea64  ffffffff8101bcda  native_apic_mem_read <- disconnect_bsp_APIC+0x6a/0xc0 
 0 ffffffff8101ea44  ffffffff8101bcf6  native_apic_mem_write <- disconnect_bsp_APIC+0x86/0xc0 
 0 ffffffff81020084  ffffffff8101a4b5  hpet_disable <- native_machine_shutdown+0x75/0x90 
 0 ffffffff81005f94  ffffffff8101a4bb  iommu_shutdown_noop <- native_machine_shutdown+0x7b/0x90 
 0 ffffffff8101a6a1  ffffffff8101a437  native_machine_emergency_restart <- native_machine_restart+0x37/0x40 
 0 ffffffff811f9876  ffffffff8101a73a  acpi_reboot <- native_machine_emergency_restart+0xaa/0x1e0 
 0 ffffffff8101a514  ffffffff8101a772  mach_reboot_fixups <- native_machine_emergency_restart+0xe2/0x1e0 
 0 ffffffff811d9c54  ffffffff8101a7a0  __const_udelay <- native_machine_emergency_restart+0x110/0x1e0 
 0 ffffffff811d9c34  ffffffff811d9c80  __delay <- __const_udelay+0x30/0x40 
 0 ffffffff811d9d14  ffffffff811d9c3f  delay_tsc <- __delay+0xf/0x20 
```
