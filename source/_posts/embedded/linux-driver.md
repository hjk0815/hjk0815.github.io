---
title: linux driver        
data: 2026-05-31 10:50:21
tags: [driver]             
categories: [embedded]              
description: linux driver  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---


# linux驱动模型

## 字符设备驱动模型

![字符设备驱动模型](/image/embedded/character-driven-model.png)

### 驱动初始化

驱动初始化中涉及到一个设备描述结构的概念。在任何一种驱动模型中,设备都会用内核中的一种结构来描述,这种结构称为设备描述结构。字符设备在内核中使用  struct cdev这种结构来描述

```c
struct cdev  
{  
  struct kobject kobj;  
  struct module *owner;  
  const struct file_operations *ops; //设备操作集  
  struct list_head list;  
  dev_t dev; //设备号  
  unsigned int count; //设备数  
};
```
count表明该类型设备的数目,如有两个串口,则count的值为2。  
dev是设备号,包含有主设备号和次设备号的信息。主设备号用于区分设备的类型,次设备号用于标记相同类型的设备的不同个体。如串口1和串口2使用同一驱动程序,则其主设备号相同,但次设备号不同。Linux内核中使用 dev_t 类型来定义设备号,dev_t 这种类型其实质为32位的 unsigned int ,其中高12位为主设备号,低20位为次设备号。  
  - 知道主设备号与次设备号,可通过 dev_t dev = MKDEV (主设备号,次设备号)  获得设备号; 
  - 从设备号分解出主设备号:主设备号 = MAJOR(dev_t dev)  
  - 从设备号分解出次设备号:次设备号 = MINOR(dev_t dev)
主设备号是一个重要的资源,可以通过静态申请和动态分配为设备分配一个主设备号:  
  - 静态申请:开发者自己选择一个数字作为主设备号,然后通过函数  register_chrdev_region 向内核申请使用。这种方法的缺点是如果申请使用的设备号已经被内核中的其它驱动使用了,则申请失败。 
  - 动态分配:使用 alloc_chrdev_region 由内核分配一个可用的主设备号。因为内核知道哪些号已经被使用了,所以不会导致分配到已经被使用的号。
既然设备号是一种资源,则设备驱动在退出后都应该释放该资源。使用  unregister_chrdev_region  函数释放这些设备号。  
ops是操作函数集。 file_operations 是一个很重要的结构,该结构的成员基本都是函数指针,并且是一些文件操作的函数的指针  
```c
struct file_operations 
{  
  struct module *owner;
  loff_t(*llseek) (struct file *, loff_t, int);  
  ssize_t(*read) (struct file *, char __user *, size_t, loff_t  *);  
  ssize_t(*aio_read) (struct kiocb *, char __user *, size_t,  loff_t);  
  ssize_t(*write) (struct file *, const char __user *, size_t,  loff_t *);  
  ssize_t(*aio_write) (struct kiocb *, const char __user *,  size_t, loff_t);  
  int (*readdir) (struct file *, void *, filldir_t);  
  unsigned int (*poll) (struct file *, struct  poll_table_struct *);  
  int (*ioctl) (struct inode *, struct file *, unsigned int,  unsigned long);  
  int (*mmap) (struct file *, struct vm_area_struct *);  
  int (*open) (struct inode *, struct file *);  
  int (*flush) (struct file *);  
  int (*release) (struct inode *, struct file *);  
  int (*fsync) (struct file *, struct dentry *, int datasync);  
  int (*aio_fsync) (struct kiocb *, int datasync);  
  int (*fasync) (int, struct file *, int);  
  int (*lock) (struct file *, int, struct file_lock *);  
  ssize_t(*readv) (struct file *, const struct iovec *,  unsigned long, loff_t *);  
  ssize_t(*writev) (struct file *, const struct iovec *,  unsigned long, loff_t *);  
  ssize_t(*sendfile) (struct file *, loff_t *, size_t,  read_actor_t, void __user *);  
  ssize_t(*sendpage) (struct file *, struct page *, int,  size_t, loff_t *, int);  
  unsigned long (*get_unmapped_area) (struct file *, unsigned  long,unsigned long, unsigned long,unsigned long);
}；
```
struct file_operations 是一个函数指针的集合,定义能在设备上进行的操作。结构中的函数指针指向驱动中的函数,这些函数实现一个针对设备的操作, 对于不支持的操作则设置函数指针为 NULL。例如:  
```c
struct file_operations dev_fops = {  
  .llseek = NULL,  
  .read = dev_read,  
  .write = dev_write,  
  .ioctl = dev_ioctl,  
  .open = dev_open,  
  .release = dev_release,  
};
```
该结构体表示应用程序能够对设备文件使用函数 read() , write() ,等,但不能使用函数 llseek() 。当执行到 read() 函数时,内核根据该结构体转移到驱动程序中的  dev_read 函数去执行。  

驱动初始化有四大步骤:  
- 分配 cdev 变量的定义可以采用静态和动态两种办法:  
  - 静态分配: struct cdev mdev;  
  - 动态分配: struct cdev *pdev = cdev_alloc();
- 初始化 struct cdev 的初始化使用cdev_init函数来完成。  
  - 原型: cdev_init(struct cdev *cdev, const struct file_operations  *fops)  
  - 参数:  
    - cdev:待初始化的cdev结构  
    - fops:设备对应的操作函数集
- 注册  字符设备的注册使用cdev_add函数来完成。  
  - 原型: cdev_add(struct cdev *p, dev_t dev, unsigned count)  
  - 参数:  
    - p:待添加到内核的字符设备结构  
    - dev:设备号  
    - count:该类设备的设备个数
- 硬件初始化  根据相应硬件的数据手册完成初始化。

### 实现设备操作

```c
int (*open)(struct inode *, struct file *) //打开设备,响应open系统调用  
int (*release)(struct inode *, struct file *);//关闭设备,响应close系统调用  
loff_t (*llseek)(struct file *, loff_t, int) //重定位读写指针,响应lseek系统调用  
ssize_t (*read)(struct file *, char __user *, size_t, loff_t *)  //从设备读取数据,响应read系统调用  
ssize_t (*write)(struct file *, const char __user *, size_t,  loff_t *) //向设备写入数据,响应write系统调用
```
以上几个函数涉及到了 struct inode 和 struct file 这两种结构体  
在Linux系统中,每一个打开的文件,在内核中都会关联一个 struct file 结构体,它由内核在打开文件时创建,在文件关闭后释放。该结构体的重要成员有:  

```c
loff_t f_pos /*文件读写指针*/  
struct file_operations *f_op /*该文件所对应的操作*/
```
每一个存在于文件系统里面的文件都会关联一个inode 结构,该结构主要用来记录文件物理上的信息。因此,它和代表打开文件的file结构是不同的。一个文件没有被打开时不会关联file结构,但是却会关联一个inode 结构。该结构体重要的成员有:  
```c
dev_t i_rdev /*设备号*/
```
一个设备支持的函数操作又称为设备方法。  
open设备方法是驱动程序用来为以后的操作完成初始化准备工作的。在大部分驱动程序中,open完成如下工作: 标明次设备号 ,启动设备。  
release设备方法的作用与open相反,这个设备方法有时也称为close,它完成的工作是关闭设备。  
read设备方法通常完成两件事情: 从设备中读取数据(属于硬件访问类操作) ,将读取到的数据返回给应用程序。  
```c
ssize_t (*read)(struct file *filp,char __user *buff,size_t  count,loff_t *offp)
```
- filp: 与字符设备文件关联的file结构指针, 由内核创建。  
- buff: 从设备读取到的数据,需要保存到的位置。由read系统调用提供该参数。
- count:请求传输的数据量,由read系统调用提供该参数。
- offp:文件的读写位置,由内核从file结构中取出后,传递进来

要注意的是,buff参数是来源于用户空间的指针,这类指针都不能被内核代码直接引用,必须使用专门的函数:  
```c
int copy_to_user(void __user *to, const void *from, int n)  
int copy_from_user(void *to, const void __user *from, int  n)
```
- 其中 copy_to_user() 用于将内核数据传送给用户空间; 
- copy_from_user() 用于将用户空间的数据传送给内核空间。

write设备方法通常完成两件事情: 从应用程序提供的地址中取出数据 ,将数据写入设备(属于硬件访问类操作)  函数原型:   
```c
ssize_t (*write)(struct file *, const char __user *,  size_t, loff_t *)
```

### 注销驱动

当我们从内核中卸载驱动程序的时候,需要使用 cdev_del 函数来完成字符设备的注销。

### 一个驱动程序范例: 

```c
#include <linux/module.h>  
#include <linux/types.h>  
#include <linux/fs.h>  
#include <linux/errno.h>  
#include <linux/init.h>  
#include <linux/cdev.h>  
#include <asm/uaccess.h>  
#include <linux/slab.h>

int dev1_registers[5];  
int dev2_registers[5];

struct cdev cdev;
dev_t devno;

/*文件打开函数*/  
int mem_open(struct inode *inode, struct file *filp)
{
    /*获取次设备号*/  
    int num = MINOR(inode->i_rdev);  
    if (num==0)  
        filp->private_data = dev1_registers;  
    else if(num == 1)  
        filp->private_data = dev2_registers;  
    else  
        return -ENODEV; //无效的次设备号  
    return 0;
}

/*文件释放函数*/  
int mem_release(struct inode *inode, struct file *filp)  
{  
    return 0;  
}

/*读函数*/  
static ssize_t mem_read(struct file *filp, char __user *buf,  size_t size, loff_t *ppos)  
{  
    unsigned long p = *ppos;  
    unsigned int count = size;  
    int ret = 0;  
    int *register_addr = filp->private_data; /*获取设备的寄存器基地址*/  
    /*判断读位置是否有效*/  
    if (p >= 5*sizeof(int))  
        return 0;  
    if (count > 5*sizeof(int) - p)  
        count = 5*sizeof(int) - p;  

    /*读数据到用户空间*/  
    if (copy_to_user(buf, register_addr+p, count))  
    {  
        ret = -EFAULT;  
    }  else  
    {  
        *ppos += count;  
        ret = count;  
    }
    return ret;
}

/*写函数*/  
static ssize_t mem_write(struct file *filp, const char __user  *buf, size_t size, loff_t *ppos)  
{  
    unsigned long p = *ppos;  
    unsigned int count = size;  
    int ret = 0;  
    int *register_addr = filp->private_data; /*获取设备的寄存器地址  */  
    
    /*分析和获取有效的写长度*/  
    if (p >= 5*sizeof(int))  
        return 0;  
    if (count > 5*sizeof(int) - p)  
        count = 5*sizeof(int) - p;  
    
    /*从用户空间写入数据*/  
    if (copy_from_user(register_addr + p, buf, count))  
        ret = -EFAULT;  
    else  
    {  
        *ppos += count;  
        ret = count;  
    }  
    return ret;  
}

/* seek文件定位函数 */  
static loff_t mem_llseek(struct file *filp, loff_t offset,  int whence)  
{  
    loff_t newpos;  
    switch(whence) {  
        case SEEK_SET:  
            newpos = offset;  
            break;  
        case SEEK_CUR:  
            newpos = filp->f_pos + offset;
            break;
        case SEEK_END:  
            newpos = 5*sizeof(int)-1 + offset;  
            break;  
        default:  
            return -EINVAL;  
    }  
    if ((newpos<0) || (newpos>5*sizeof(int)))  
        return -EINVAL;  
    filp->f_pos = newpos;  
    return newpos;  
}

/*文件操作结构体*/  
static const struct file_operations mem_fops =  
{  
    .llseek = mem_llseek,  
    .read = mem_read,  
    .write = mem_write,  
    .open = mem_open,  
    .release = mem_release,  
};

/*设备驱动模块加载函数*/  
static int memdev_init(void)  
{  
    /*初始化cdev结构*/  
    cdev_init(&cdev, &mem_fops);  
    /* 注册字符设备 */  
    alloc_chrdev_region(&devno, 0, 2, "memdev");  
    cdev_add(&cdev, devno, 2);  
}

/*模块卸载函数*/  
static void memdev_exit(void)  
{  
    cdev_del(&cdev); /*注销设备*/  
    unregister_chrdev_region(devno, 2); /*释放设备号*/  
}

MODULE_LICENSE("GPL");  
module_init(memdev_init);  
module_exit(memdev_exit);

```

## LCD驱动模型

在 Linux 系统中，LCD（液晶显示器）驱动的核心任务是：管理显存（Video Memory），并将用户空间的图像数据，通过硬件接口（如 RGB、MIPI-DSI、LVDS 等）完美地投射到物理屏幕上。

现代 Linux 中，LCD 驱动主要基于两种框架实现：老一代的 Framebuffer（显示缓冲区） 和新一代的 DRM（Direct Rendering Manager）。

写个LCD驱动入口函数,需要以下4步:  
1. 分配一个fb_info结构体: framebuffer_alloc();   
2. 设置 fb_info  
3. 设置硬件相关的操作  
4. 使能LCD,并注册 fb_info: register_framebuffer()  

需要用到的函数:  
```c
void *dma_alloc_writecombine(struct device *dev, size_t size,  dma_addr_t *handle, gfp_t gfp); //分配DMA缓存区给显存  //
// 返回值为:申请到的DMA缓冲区的虚拟地址,若为NULL,表示分配失败,则需要使用  dma_free_writecombine()释放内存,避免内存泄漏
//参数如下:  
// *dev:指针,这里填0,表示这个申请的缓冲区里没有内容  
// size:分配的地址大小(字节单位)  
// *handle:申请到的物理起始地址  
// gfp:分配出来的内存参数,标志定义在<linux/gfp.h>,常用标志如下:  
// GFP_ATOMIC 用来从中断处理和进程上下文之外的其他代码中分配内存. 从不睡眠.  
// GFP_KERNEL 内核内存的正常分配. 可能睡眠.  
// GFP_USER 用来为用户空间页来分配内存; 它可能睡眠.
```


## 总线设备驱动模型

自内核2.6版本开始,需要关注的是总线、设备和驱动这3个实体,总线将设备和驱动绑定。在Linux内核系统中注册一个设备的时候,会寻找与之对应驱动进行匹配;相反地,系统中注册一个驱动的时候,会去寻找一个对应的设备进行匹配。匹配的的工作由总线来完成。  

在Linux设备中有的是没有对应的物理总线的,但为了适配Linux的总线模型,内核针对这种没有物理总线的设备开发了一种虚拟总线——platform总线。  

将设备和驱动独立开,驱动尽可能写的通用,当来了一个类似的设备后也可以使用这个驱动,让驱动程序可以重用。这体现了Linux驱动的软件架构设计的思想。   

按照这个思路,Linux中的设备和驱动都需要挂接在一种总线上,比如i2c总线上的eeprom,eeprom作为设备,eeprom的驱动都挂接在i2c驱动上。但是在嵌入式系统中,soc系统一般都会集成独立的i2c控制器,控制器也是需要驱动的,但是再按照设备-总线-驱动模型进行设计,就会发现无法找到一个合适总线去挂接控制器设备和控制器驱动了(i2c控制器是挂接在CPU内部的总线上,而不是i2c总线),所以 Linux发明了一种虚拟总线,称为platform总线,相应的设备称为platform_device  (控制器设备),对应的驱动为platform_driver(控制器驱动),用platform总线来承载这些相对特殊的系统。  

注意:所谓的platform_device并不是与字符设备、块设备和网络设备并列的概念,而是Linux系统提供的一种附加手段,例如,在 S3C6410处理器中,把内部集成的  I2C、RTC、SPI、LCD、看门狗等控制器都归纳为platform_device,而它们本身就是字符设备。我们要记住,platform 驱动只是在字符设备驱动外套一层 platform_driver 的外壳。引入platform模型符合Linux设备模型 —— 总线、设备、驱动,设备模型中配套的sysfs节点都可以用,方便我们的开发;当然你也可以选择不用,不过就失去了一些platform带来的便利;  

设备驱动中引入platform 概念,隔离BSP和驱动。在BSP中定义platform设备和设备使用的资源、设备的具体匹配信息,而在驱动中,只需要通过API去获取资源和数据,做到了板相关代码和驱动代码的分离,使得驱动具有更好的可扩展性和跨平台性。  

![总线设备](image/embedded/bus_drv.png)

总线设备驱动模型的匹配过程：  

一边的“device”结构体和另一边的“较稳定的 drivice 代码”的联系:“device_add()”除了将“devcie”结构放到bus 的“dev 链表”之外,还会从另一边的“drv”链表中取表元即某个“driver”结构,用总线里的一个(.match)函数来作比较,看另一边的“driver”是否支持一边的“device”。若是能够支持,则接着调用软件驱动部分的“.probe”函数。"driver_register()"会将“bus_drv_dev”模型中的较稳定代码“driver”结构体放到虚拟总线的某个链表(drv 链表)中。从另一边的“dev”链表中取出每一个“device”结构用bus 中的“.match”函数来作比较,若支持则调用“.probe”函数。左右两个注册就建立起来的一种机制。在“.probe”函数中做的事件由自已决定,打印一句话,或注册一个字符设备,再或注册一个“input_dev”结构体等等都是由自已决定。强制的把一个驱动程序分为左右两边这种机制而已,可以把这套东西放在任何地方,这里的“driver”只是个结构体不要被这个名字迷惑,“device”也只是个结构体,里面放什么内容都是由自已决定的。


## 输入子系统模型

每个硬件都有一个input_dev结构体,每个软件都有一个input_handler结构体。input_dev和input_handler分别通过input_register_device(),input_register_handler()向核心层注册硬件和软件。  

```c
int input_register_device(struct input_dev *dev) //*dev:要注册  的驱动设备  
{  
    ... ...  
    list_add_tail(&dev->node, &input_dev_list); //(1)放入链表中  
    ... ...  
    list_for_each_entry(handler, &input_handler_list, node)  
    //(2)  
    input_attach_handler(dev, handler);  
    ... ...  
}
```
![输入子系统](image/embedded/input-sub-system.png)

从input_dev方向分析: input设备在增加到input_dev_list链表上之后,会查找input_handler_list事件处理链表上的handler进行匹配,这里的匹配方式与总线设备驱动模型的device和driver匹配过程很相似,所有的input_device都挂在input_dev_list上,所有类型的事件都挂在input_handler_list 上,进行“匹配相亲”。如果匹配上了,就调用input_handler的connect函数进行连接。设备就是在此时注册的  

从input_handler方向分析:将handler挂到链表input_handler_list下,然后遍历input_dev_list链表,查找并匹配输入设备对应的事件处理层,如果匹配上了,就调用connect函数进行连接,并创建input_handle结构。  

所以,不管新添加input_dev还是input_handler,都会进入input_attach_handler()判断两者id是否有支持, 若两者支持便进行连接。

## platform总线匹配规则

总线,设备,驱动。匹配规则就是当有一个新的设备挂起时,总线被唤醒,match函数被调用,用device名字去跟本总线下的所有驱动名字去比较。相反就是用驱动的名字去device链表中和所有device的名字比较。如果匹配上,才会调用驱动中的probe函数,否则不调用。至于先后顺序,鉴于个人理解,不会有影响,不管谁先谁后,bus都会完成匹配工作。谈谈对Linux设备驱动模型的认识:设备驱动模型的出现主要有三个好处,设备与驱动分离,驱动可移植性增强;设备驱动抽象结构以总线结构表示看起来更加清晰明了,谁是属于哪一条bus的;最后,就是大家最熟悉的热插拔了,设备与驱动分离,很好的奠定了热插拔机制。  

# Linux内核

## 内核中申请内存的函数

### kmalloc

```c
void *kmalloc(size_t size, gfp_t flags)
```
kmalloc是内核中最常用的一种内存分配方式,它通过调用kmem_cache_alloc函数来实现。kmalloc一次最多能申请的内存大小由include/linux/Kmalloc_size.h的内容来决定,在默认的2.6.18内核版本中,kmalloc一次最多能申请大小为131702B也就是128KB字节的连续物理内存。测试结果表明,如果试图用kmalloc函数分配大于128KB的内存,编译不能通过。  

### vmalloc

```c
void *vmalloc(unsigned long size)
```
前面几种内存分配方式都是物理连续的,能保证较低的平均访问时间。但是在某些场合中,对内存区的请求不是很频繁,较高的内存访问时间也可以接受,这是就可以分配一段线性连续,物理不连续的地址,带来的好处是一次可以分配较大块的内存。vmalloc对一次能分配的内存大小没有明确限制。出于性能考虑,应谨慎使用vmalloc函数。在测试过程中,最大能一次分配1GB的空间。

### dma_alloc_coherent

```c
void *dma_alloc_coherent(struct device *dev, size_t  size,ma_addr_t *dma_handle, gfp_t gfp)
```
DMA是一种硬件机制,允许外围设备和主存之间直接传输IO数据,而不需要CPU的参与,使用DMA机制能大幅提高与设备通信的吞吐量。DMA操作中,涉及到CPU高速缓存和对应的内存数据一致性的问题,必须保证两者的数据一致,在x86_64体系  结构中,硬件已经很好的解决了这个问题, dma_alloc_coherent和get_free_pages 函数实现差别不大,前者实际是调用alloc_pages函数来分配内存,因此一次分配内存的大小限制和后者一样。__get_free_pages分配的内存同样可以用于DMA操作。测试结果证明,dma_alloc_coherent函数一次能分配的最大内存也为4M。  

### ioremap

```c
void * ioremap (unsigned long offset, unsigned long size)
```
ioremap是一种更直接的内存“分配”方式,使用时直接指定物理起始地址和需要分配内存的大小,然后将该段物理地址映射到内核地址空间。ioremap用到的物理地址空间都是事先确定的,和上面的几种内存分配方式并不太一样,并不是分配一段新的物理内存。ioremap多用于设备驱动,可以让CPU直接访问外部设备的IO空间。ioremap能映射的内存由原有的物理内存空间决定.

## 内核空间和用户空间

对 32 位操作系统而言,它的寻址空间(虚拟地址空间,或叫线性地址空间)为 4G (2的32次方)。也就是说一个进程的最大地址空间为 4G  

操作系统的核心是内核(kernel),它独立于普通的应用程序,可以访问受保护的内存空间,也有访问底层硬件设备的所有权限。为了保证内核的安全,现在的操作系统一般都强制用户进程不能直接操作内核。具体的实现方式基本都是由操作系统将虚拟地址空间划分为两部分,一部分为内核空间,另一部分为用户空间。针对 Linux 操作系统而言,最高的 1G 字节(从虚拟地址 0xC0000000 到 0xFFFFFFFF)由内核使用,称为内核空间。而较低的 3G 字节(从虚拟地址 0x00000000 到 0xBFFFFFFF)由各个进程使用,称为用户空间  

对上面的话可以这样理解：  
- 每个进程的 4G 地址空间中，最高 1G 都是一样的，即内核空间。只有剩余的 3G 才归进程自己使用。换句话说就是，最高的 1G 的内核空间是被所有进程共享的  

![进程内存地址空间分配](image/embedded/kernel-usr-mem.png)

### 内核态和用户态

当进程运行在内核空间时就处于内核态,而进程运行在用户空间时则处于用户态。  

在内核态下,进程运行在内核地址空间中,此时 CPU 可以执行任何指令。运行的代码也不受任何的限制,可以自由地访问任何有效地址,也可以直接进行端口的访问。  

在用户态下,进程运行在用户地址空间中,被执行的代码要受到 CPU 的诸多检查,它们只能访问映射其地址空间的页表项中规定的在用户态下可访问页面的虚拟地址,且只能对任务状态段(TSS)中 I/O 许可位图(I/O Permission Bitmap)中规定的可访问端口进行直接访问。  

对于以前的 DOS 操作系统来说,是没有内核空间、用户空间以及内核态、用户态这些概念的。可以认为所有的代码都是运行在内核态的,因而,用户编写的应用程序代码可以很容易的让操作系统崩溃掉。  

对于 Linux 来说,通过区分内核空间和用户空间的设计,隔离了操作系统代码(操作系统的代码要比应用程序的代码健壮很多)与应用程序代码。即便是单个应用程序出现错误,也不会影响到操作系统的稳定性,这样其它的程序还可以正常的运行(Linux可是个多任务系统啊!)。所以,区分内核空间和用户空间本质上是要提高操作系统的稳定性及可用性。

### 用户空间与内核的通信方式

#### 使用api

```c
get_user(x,ptr) //在内核中被调用,获取用户空间指定地址的数值并保存到内核变量x中。  
put_user(x,ptr) //在内核中被调用,将内核空间的变量x的数值保存到到用户空间指定地址处。  

copy_from_user()
copy_to_user() //主要应用于设备驱动读写函数中,通过系统调用触发。
```

### 使用proc文件系统

和sysfs文件系统类似,也可以作为内核空间和用户空间交互的手段。/proc 文件系统是一种虚拟文件系统,通过他可以作为一种linux内核空间和用户空间的通信方式。与普通文件不同,这里的虚拟文件的内容都是动态创建的。使用/proc文件系统的方式很简单。调用create_proc_entry,返回一个proc_dir_entry指针,然后去填充这个指针指向的结构就好了。

### 使用sysfs文件系统 + kobject

每个在内核中注册的kobject都对应着sysfs系统中的一个目录。可以通过读取根目录下的sys目录中的文件来获得相应的信息。除了sysfs文件系统和proc文件系统之外,一些其他的虚拟文件系统也能同样达到这个效果。  

### netlink

netlink socket提供了一组类似于BSD风格的API,用于用户态和内核态的IPC。相比于其他的用户态和内核态IPC机制,netlink有几个好处:  
1. 使用自定义一种协议完成数据交换,不需要添加一个文件等。2.
2. 可以支持多点传送。
3. 支持内核先发起会话。
4. 异步通信,支持缓存机制。  

### 文件

应该说这是一种比较笨拙的做法,不过确实可以这样用。当处于内核空间的时候,直接操作文件,将想要传递的信息写入文件,然后用户空间可以读取这个文件便可以得到想要的数据了。下面是一个简单的测试程序,在内核态中,程序会向“/home/melody/str_from_kernel”文件中写入一条字符串,然后我们在用户态读取这个文件,就可以得到内核态传输过来的数据了。  

### mmap系统调用

可以将内核空间的地址映射到用户空间。

### 信号

从内核空间向进程发送信号。用户程序出现重大错误,内核发送信号杀死相应进程。  

## 应用程序中open()从用户空间到内核空间的过程

1. 应用层调用open函数,在VFS层中找到struct inode结构体,判断是字符设备还是块设备,根据设备号,可以找到对应的驱动程序。  

2. 在驱动层中,每个字符设备都有一个struct cdev结构体,这个结构体通过struct inode结构体中的i_cdev把连接起VFS层和驱动层,struct cdev结构体描述了字符设备所有信息,其中最重要的一项就是字符设备的操作函数接口  

3. struct cdev结构体中的struct file结构体记录了操作字符设备的一些函数,比如  open read write函数等。  struct file结构体其实是在VFS层的,通过struct file结构体指针指向驱动层的struct file结构体将驱动层函数和VFS层链接起来  

4. 任务完成,VFS层会给应用返回一个文件描述符(fd)。这个fd是和struct file 结构体对应的。  

## 申请大块内核内存

vmalloc

# 设备驱动

## 主设备号与次设备号

主设备号:主设备号标识设备对应的特定的驱动程序。虽然现代的linux内核允许多个驱动程序共享主设备号,但我们看待的大多数设备仍然按照“一个主设备对应一个驱动程序”的原则组织。

次设备号:次设备号由内核使用,用于确定由主设备号对应驱动程序中的各个设备。依赖于驱动程序的编写方式,我们可以通过次设备号获得一个指向内核设备的直接指针,也可将此设备号当作设备本地数组的索引。

## 字符型驱动设备创建设备文件

1. 手动创建： 
- `mknod /dev/led c 250 0`: 其中dev/led 为设备节点 ,c 代表字符设备, 250代  表主设备号, 0代表次设备号。
2. 自动创建：
- UDEV/MDEV是运行在用户态的程序,可以动态管理设备文件,包括创建和删除设备文件,运行在用户态意味着系统要运行之后,在 /etc/init.d/rcS 脚本文件中  会执行 mdev -s 自动创建设备节点。

## 字符设备的注册

```c
void cdev_init(struct cdev *cdev, struct file_operations *fops)
```
该注册函数可以将cdev结构嵌入到自己的设备特定的结构中。cdev是一个指向结构体cdev的指针,而fops是指向一个类似于f file_operations结构(可以是  file_operations结构,但不限于该结构)的指针。

```c
int register_chrdev(unsigned int major, const char *namem ,  struct file operations *fopen);
```
该注册函数是早期的注册函数,major是设备的主设备号,name是驱动程序的名称,而fops是默认的file_operations结构(这是只限于file_operations结构)。对于register_chrdev的调用将为给定的主设备号注册0-255作为次设备  号,并为每个设备建立一个对应的默认cdev结构。

## 设备文件的创建

udev,其实就是device_create()、class_create()这一套接口,所谓udev是上层用户空间程序,是基于驱动中创建使用了这两个接口而起作用的,但是udev在日常开发中几乎接触不到,只需在驱动中调用创建节点的这两个API就ok了,剩下的工作就交给udev去做了

## 字符设备和块设备的区别

Linux中I/O设备分为两类:块设备和字符设备。两种设备本身没有严格限制,但是,基于不同的功能进行了分类。  

字符设备:提供连续的数据流,应用程序可以顺序读取,通常不支持随机存取。相反,此类设备支持按字节/字符来读写数据。字符终端、串口、鼠标、键盘、摄像头、声卡和显卡等就是典型的字符设备。  

块设备:应用程序可以随机访问设备数据,程序可自行确定读取数据的位置。硬盘是典型的块设备,应用程序可以寻址磁盘上的任何位置,并由此读取数据。此外,数据的读写只能以块(通常是512B)的倍数进行。与字符设备不同,块设备并不支持基于字符的寻址。如:u盘,SD卡,磁盘等。  

# 常用函数

## ioremap

```c
void * __ioremap(unsigned long phys_addr, unsigned long size,  unsigned long flags)  
void *ioremap(unsigned long phys_addr, unsigned long size)
```
phys_addr:是要映射的物理地址  
size:是要映射的长度,单位是字节  
flags:要映射的IO空间的和权限有关的标志;  
头文件:io.h

## open

```c
int open( const char * pathname, int flags);  
int open( const char * pathname,int flags, mode_t mode);
```

## read & write & copy_to_user & copy_from_user






