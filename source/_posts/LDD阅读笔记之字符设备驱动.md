---
title: "LDD阅读笔记之字符设备驱动"
category: LDD
tags: [Linux,Driver]
date: 2015-01-04 9:23
comments: true
---

# 主要开发流程介绍

module_init宏和module_exit宏

    当模块装载时需要调用module_init宏指定的函数，
    卸载时需要调用 module_exit宏指定的函数

以下是简单的init流程：

- 初始化设备
- 初始化file_operation
- 获取字符设备号
- 注册字符设备

当卸载模块时，需要释放申请的设备号。

<!--more-->
# 主设备号和次设备号

对字符设备的访问是通过文件系统内的设备名称进行的。那些名称被称为特殊 文件、设备文件，或者简单称为文件系统树的节点，他们通常位于/dev目录。

通常而言，主设备号表示设备对应的驱动程序。例如，/dev/null和/dev/zero 由驱动程序1管理，而虚拟控制台和串口终端由驱动程序4管理。

>现代的Linux内核允许多个驱动程序共享主设备号，但我们看到的仍然按照”一个主设备号对应一个驱动程序“的原则组织。次设备号的作用被加强了，一个主设备号加一个次设备号可以对应一个驱动程序。

>/proc/devices 可以查看注册的主设备号；/proc/modules 可以查看正在使用模块的进程数两个文件中的module名字不同，devices中的是用户设置的name，modules中的是名字module.ko中的module。

# 设备编号的内部表达

在内核中dev_t类型（在linux/types.h中定义）用来保存设备编号——包括主设备号和次设备号。我们的代码不应该对设备编号的组织做任何假定，而应该始终使用linux/kdev_t.h中定义的宏。

```c
MAJOR(dev_t dev);
MINOR(dev_t dev);
```
相反，如果要将主设备号和次设备号转换成dev_t类型，则使用：

```c
MKDEV(int major, int minor);
```
# 分配和释放设备编号

在建立一个字符设备之前，我们的驱动程序首先要做的事情就是获得一个或者多个设备编号。完成该工作的必要函数是 *register_chrdev_region*，该函数在linux/fd.h中声明：

```c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
```

其中 *first* 是要分配的设备编号范围的起始值。first的次设备号经常被置为0，但对该函数不是必须的。*count* 是所请求的连续设备编号的个数。 *name* 是和该编号范围关联的设备名称，它将出现在/proc/devices和sysfs中。

如果我们知道可用的设备编号，则 *register_chrdev_region* 会工作很好。但是 我们进程不知道将要用哪些主设备号；应此提供了以下函数

```c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, 
                            unsigned int count, char *name);
```

*dev* 用于输出参数，成功调用后保存已分配范围的第一个编号。 *firstminor* 应该是 要使用的被请求的第一个次设备号，通常是0。 *count* 和 *name* 参数和 *register_chrdev_region* 相同。

无论使用哪种方法分配设备号，都应该在不再使用它们时释放这些设备编号。

```c
void unregister_chrdev_region(dev_t first, unsigned int count);
```

通常我们在清除模块中调用 *unregister_chrdev_region* 函数。

# 一些重要的数据结构

## 文件操作

迄今为止，我们已经为自己保留了一些设备编号，但尚未将任何驱动程序的操作连接到这些 编号。file_operations结构就是用来建立这种连接的。

```c   
struct file_operations {
    struct module *owner;
     loff_t(*llseek) (struct file *, loff_t, int);
     ssize_t(*read) (struct file *, char __user *, size_t, loff_t *);
     ssize_t(*aio_read) (struct kiocb *, char __user *, size_t, loff_t);
     ssize_t(*write) (struct file *, const char __user *, size_t, loff_t *);
     ssize_t(*aio_write) (struct kiocb *, const char __user *, size_t,
                  loff_t);
    int (*readdir) (struct file *, void *, filldir_t);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    int (*ioctl) (struct inode *, struct file *, unsigned int,
              unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, struct dentry *, int datasync);
    int (*aio_fsync) (struct kiocb *, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
     ssize_t(*readv) (struct file *, const struct iovec *, unsigned long,
              loff_t *);
     ssize_t(*writev) (struct file *, const struct iovec *, unsigned long,
               loff_t *);
     ssize_t(*sendfile) (struct file *, loff_t *, size_t, read_actor_t,
                 void __user *);
     ssize_t(*sendpage) (struct file *, struct page *, int, size_t,
                 loff_t *, int);
    unsigned long (*get_unmapped_area) (struct file *, unsigned long,
                        unsigned long, unsigned long,
                        unsigned long);
};

struct file_operations scull_fops = {
    .owner = THIS_MODULE,
    .llseek = scull_llseek,
}; //C99 syntax
```

## file结构

在linux/fs.h中定义的struct file是设备驱动程序所使用的第二个最重要的数据结构。

>注意：file结构和用户空间程序中的FILE没有任何关联。FILE是C库中的定义的结构， 而struct file是一个内核结构，不会出现在用户程序中（文件描述符应该是指向该结构体）。

```c
struct file {
    mode_t f_mode;      //文件模式
    loff_t f_pos;       //当前读写位置
    unsigned int f_flags; //文件标志，如O_RDONLY、O_NONBLOCK和O_SYNC。
    struct file_operations *f_op;   //与文件相关的操作。
    void *private_data;     //open系统调用在调用驱动程序的open方法前将这个
                            //指针置为NULL。
    struct dentry *f_dentry;//文件对应的目录项(dentry)结构。
                            //filp->f_dentry->d_inode
    ……
}
```

## inode结构

内核用inode结构在内部表示文件，因此它和file结构不同，后者表示打开的文件描述符。对 单个文件，可能会有多个表示打开的文件描述符的file结构，但它们都指向单个inode结构。

```c
struct inode {
    dev_t i_rdev;       //对表示设备文件的inode结构，
                        //该字段包含了真正的设备编号
    struct cdev *i_cdev;//表示字符设备的内核内部结构。
    ……
```

*i_rdev* 的类型在2.5开发系列版本中发生了变化，为了鼓励编写可移植性更强的代码， 内核开发者增加了两个新的宏。

```c
unsigned int iminor(struct inode *inode);       //获取次设备号
unsigned int imajor(struct inode *inode);       //获取主设备号
```

# 字符设备的注册

内核内部使用struct cdev结构来表示字符设备。在内核调用设备的操作之前，必须分配 并注册一个或者多个上述结构。为此，我们的代码需要包含linux/cdev.h，其中定义 了这个结构以及与其相关的一些辅助函数。   

```c
struct cdev {
    struct kobject kobj;
    struct module *owner;
    const struct file_operations *ops;
    struct list_head list;
    dev_t dev;
    unsigned int count;
};
```
>有一个老的机制可以避免使用cdev结构，但是新代码应该使用新技术。

```c
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops;
```

我们可以将cdev结构嵌入到自己的设备特定结构中（有点类似派生的C版本）。 如果没有通过 *cdev_alloc* 申请，则我们需要用下面的代码来初始化已分配的结构：

```c       
void  cdev_init(struct cdev *cdev, 
                    struct file_operations *fops);
```

还有一个字段需要初始化，和file_operations一样，struct cdev也有一个所有者字段， 应设置为THIS_MODULE。

在设置完cdev结构后，最后的步骤是告诉内核该结构的信息：

```c
int cdev_add(struct cdev *dev, dev_t num, 
            unsigned int count);
```

*num* 是该设备对应的第一个设备编号，count是应该和该设备关联的设备编号数量。 只要 *cdev_add* 成功返回，我们的设备就要开始工作了，它的操作会被内核调用。 要从系统移除一个字符设备，做如下调用：

```c
void cdev_del(struct cdev *dev);
```

在将cdev通过 *cdev_del* 移除后，就不应该再访问cdev结构了。

## Scull中的设备注册

```c
static void scull_setup_cdev(struct scull_dev *dev, int index)
{
    int err, devno = MKDEV(scull_major, scull_minor+index);
    cdev_init(&dev->cdev, &scull_fops);
    dev->cdev.owner = THIS_MODULE;
    dev->cdev.ops = &scull_fops;  // this expression is redundancy ?
    err = cdev_add(&dev->cdev, devno, 1);
        if(err){
            printk(KERN_NOTICE "Error %d adding scull%d", err, index);
        }
    }
```

因为cdev结构被内嵌到了strcut scull_dev中，因此必须调用cdev_init来执行该结构的初始化。

## 早期办法

```c
int register_chrdev(unsigned int major, const char *name,
                        struct file_operations *fops);
```

对 *register_chrdev* 的调用将为 **给定的主设备号注册0～255作为次设备号**，并为 每个设备建立一个对应的默认cdev结构。使用这一接口的驱动程序必须能够处理所有 256个次设备号上的 *open* 调用。对应的移除函数：

```c       
int unregister_chrdev(unsigned int major, const char *name);
```
# open和release

## open方法

*open* 方法提供给驱动程序以初始化的能力（和module_init的不同），open应完成如下 工作：

- 检查设备特定的错误（如设备未就绪或类似硬件问题）
- 如果设备首次打开，对其进行初始化。
- 如有必要，更新f_op指针。
- 分配并填写置于filp->private_data里的数据结构。

*open* 方法的原型如下：

```c
int (*open) (struct inode *inode, struct file *filp);
```

inode参数在i_cdev字段中包含了我们需要的信息，即我们先前设定的cdev结构。我们通常需要 包含它的scull_dev结构，内核黑客为我们提供了此类技巧，它通过定义在linux/kernel.h 中的container_of宏实现：

```c
container_of(pointer, container_type, container_field);

struct scull_dev *dev;
dev = container_of(inode->i_cdev, struct scull_dev, cdev); //cdev是成员名
flip->private_data = dev;
```

另一个确定要打开的设备的方法是：检查保存在inode中的次设备号。如果使用了 *register_chrdev* 注册设备，则必须使用该技术。 经过简化的 *scull_open* 代码：

```c        
int scull_open(struct inode *inode, struct file *filp)
{
    struct scutll_dev *dev;
    dev = container_of(inode->i_cdev, struct scull_dev, cdev);
    filp->private_data = dev;
    if((filp->f_flags&O_ACCMODE)==O_WRONLY){
        scull_trim(dev);
    }
    return 0;
}
```

由于我们没有维护scull的打开计数，只维护模块的使用计数，因此也就没有类似"首次打开时初始化设备"这类动作。

## release方法

*release* 方法和 *open* 相反，有时这个方法被称为 *device_close*。

- 释放由 *open* 分配的、保存在filp->private_data中的所有内容。
- 在最后一次关闭操作时关闭设备。

>注： 后面scull_open为每种设备都替换了不同的filp->f_op，所以不同的设备由 不同的函数关闭

当关闭一个设备文件的次数比打开它的次数多时，系统中会发生什么？ 答案很简单：并不是每个close系统调用都会引起对release方法的调用。只有 真正释放设备数据结构的 *close* 调用才会调用这个方法。内核对每个file 结构维护其被使用多少次的计数器。无论 *fork* 还是 *dup* 都不会创建新 的数据结构（仅由open创建），他们只是增加已有结构中的计数。只有file结 构的计数归零是，close系统调用才会执行release方法，这只在删除这个结 构时才会发生。**保证了一次open只会看到一次release调用**。

>注意：flush方法在应用程序每次调用close时都会调用。

# read和write

*read* 和 *write* 方法完成的任务是相似的，亦即，拷贝数据到应用程序空间， 或者反过来。

```C
ssize_t read(struct file *filp, char __user *buff,
            size_t count, loff_t *offp);
ssize_t write(struct file *filp, const char __user *buff,
            size_t count, loff_t *offp);
```

需要说明的是read和write方法的buff参数是用户空间的指针。因此，代码不能直接 引用其中内容。原因如下：

- 随着驱动程序所运行的架构不同或者内核配置不同，在内核模式运行时，用户
  空间的指针可能是无效的。
- 即使该指针在内核空间中代表相同的东西，但用户空间的内存是分页的，而在系统
  调用被调用时，涉及到的内存可能根本不在RAM中。对用户空间的内存的直接引用
  将导致页错误，而这对内核代码来说是不允许发生的事情。其结果可能是“oops”。
- 我们讨论的指针可能由用户程序提供，而该程序可能存在缺陷或者是个恶意程序。

为确保安全，需要使用下面几个函数（由linux/uaccess.h中定义）

```c
unsigned long copy_to_user(void __user *to,
                            const void *from,
                            unsigned long count);
unsigned long copy_from_user(void *to,
                             const void __user *from,
                             unsigned long count);
```

当内核空间运行的代码访问用户空间的时候必须多加小心，因为被寻址的用户页面 可能不存在当前内存中，于是虚拟内存子系统经该进程转入休眠，知道该页面被加载 到期望位置。对驱动开发人员来说，这带来的结果就是任何访问用户空间的函数都是 必须可重入的（异步信号安全），必须能和其他驱动程序函数并发执行，更特别的是 必须处于能够合法休眠的状态。 

![The arguments to read][1]

# readv和writev

Unix系统很早就已支持两个可选的系统调用：readv和writev。 如果驱动程序没有提供用于处理向量操作的方法，readv和writev会通过对read和 write方法的多次调用来实现。但在很多情况下，直接在驱动程序中实现readv和writev 可以获得更高的效率。 

```c
ssize_t (*readv) (struct file *filp, const struct iovec *iov,
                    unsigned long count, loff_t *ppos);
ssize_t (*writev) (struct file *filp, const struct iovec *iov,
                    unsigned long count, loff_t *ppos);
```

iovec结构定义在linux/uio.h中：

```c
struct iovec {
    void __user *iov_base;
    __kernel_size_t iov_len;
}
```

每个iovec结构都描述了一个用于传输的数据块。函数中的count参数指明要操作多少 个iovec结构。 正确而有效率的操作经常需要驱动程序做一些更为巧妙的事情。 例如，磁带驱动程序的writev就应将所有iovec结构的内容作为磁带上的单个记录写入。 如果忽略他们，内核会通过read和write模拟它们。 

# 实例

```c
    /*
     * =====================================================================================
     *
     *       Filename:  scull.c
     *
     *    Description:  this is a first driver from ldd3
     *                  ignore mutithread race, data overflow,
     *                  just a simple example.
     *
     *        Version:  1.0
     *        Created:  11/29/2014 09:00:04 PM
     *       Revision:  none
     *       Compiler:  gcc
     *
     *         Author:  Xu Jianjun(firemiles), 
     *   Organization:  
     *
     * =====================================================================================
     */
    #include <linux/module.h>
    #include <linux/init.h>
    #include <linux/fs.h>
    #include <linux/cdev.h>
    #include <linux/uaccess.h>
    #include <linux/slab.h>

    MODULE_LICENSE("GPL");

    static long scull_ioctl (struct file *file, unsigned int cmd, unsigned long n);
    static ssize_t scull_write (struct file *file, const char __user *buff, 
                            size_t count, loff_t *f_ops );
    static ssize_t scull_read (struct file *file, char __user *buff, 
                            size_t count, loff_t *f_ops );
    static loff_t scull_llseek (struct file *file, loff_t f_ops, int count);
    static int scull_release (struct inode *inode, struct file *file );
    static int scull_open (struct inode *inode, struct file *file );

    static int scull_major = 0;
    static int scull_minor = 0;

    struct scull_dev {
        char *data;
        size_t maxlen; //buffer length
        size_t len;    //data length
        struct  cdev cdev;
    }scull_dev1;


    struct file_operations scull_fops = {
        .owner  =   THIS_MODULE,
        .llseek =   scull_llseek,
        .read   =   scull_read,
        .write  =   scull_write,
        .unlocked_ioctl  =   scull_ioctl, //new api
        .open   =   scull_open,
        .release=   scull_release,
    };

    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull open
     *  Description:  
     * =====================================================================================
     */
    static int scull_open (struct inode *inode, struct file *filp )
    {
        struct scull_dev *dev;
        dev = container_of(inode->i_cdev, struct scull_dev, cdev);
        filp->private_data = dev;
        if(dev->data == NULL){
            dev->data = (char *)kmalloc(1024, GFP_KERNEL);
            dev->maxlen = 1024;
            dev->len = 0;
        }
        return 0;
    }       /* -----  end of function scull open  ----- */


    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_close
     *  Description:  
     * =====================================================================================
     */
    static int scull_release (struct inode *inode, struct file *filp )
    {
        return 0;
    }       /* -----  end of function scull_close  ----- */



    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_llseek
     *  Description:  
     * =====================================================================================
     */
    static loff_t scull_llseek (struct file *file, loff_t f_ops, int count)
    {
        return 0;
    }       /* -----  end of function scull_llseek  ----- */

    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_read
     *  Description:  
     * =====================================================================================
     */
    static ssize_t scull_read (struct file *filp, char __user *buff, size_t count, loff_t *f_ops )
    {
        int num;
        struct scull_dev *dev = filp->private_data;
        copy_to_user(buff, dev->data, dev->len); // ignore buff overflow;
        num = dev->len;
        dev->len = 0;
        return num;
    }       /* -----  end of function scull_read  ----- */


    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_write
     *  Description:  
     * =====================================================================================
     */
    static ssize_t scull_write (struct file *filp, const char __user *buff, size_t count, loff_t *f_ops )
    {
        struct scull_dev *dev = filp->private_data;
        copy_from_user(dev->data, buff, count); // ignore data overflow; 
        dev->len = count;
        return count;
    }       /* -----  end of function scull_write  ----- */


    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_ioctl
     *  Description:  
     * =====================================================================================
     */
    static long scull_ioctl (struct file *file, unsigned int cmd, unsigned long n)
    {
        return 0;
    }       /* -----  end of function scull_ioctl  ----- */


    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_setup_cdev
     *  Description:  
     * =====================================================================================
     */
    static void scull_setup_cdev (struct scull_dev *dev, int index)
    {
        int err;
        dev_t devnum;
        char name[16];
        sprintf(name,"scull1");
        if(scull_major){
            devnum = MKDEV(scull_major, scull_minor+index);
            err = register_chrdev_region(devnum, 1, name);
        }else{
            err = alloc_chrdev_region(&devnum, scull_minor+index, 1, name);
            scull_major = MAJOR(devnum);
        }
        if(err<0){
            printk(KERN_WARNING "scull1: can't get major %d\n", scull_major);
            return;
        }
        cdev_init(&dev->cdev, &scull_fops);
        dev->cdev.owner = THIS_MODULE;
        dev->cdev.ops = &scull_fops; //nessary?
        err = cdev_add(&dev->cdev, devnum, 1);
        if(err){
            printk(KERN_NOTICE "Error %d adding scull%d", err, index);
        }
    }       /* -----  end of function scull_setup_cdev  ----- */

    /*  
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_init
     *  Description:  
     * =====================================================================================
     */
    static int __init scull_init (void)
    {
        scull_setup_cdev(&scull_dev1, 1);
        printk(KERN_ALERT "scull init\n");
        return 0;
    }       /* -----  end of function scull_init  ----- */


    /* 
     * ===  FUNCTION  ======================================================================
     *         Name:  scull_exit
     *  Description:  
     * =====================================================================================
     */
    static void __exit scull_exit (void)
    {
        kfree(scull_dev1.data);
        unregister_chrdev_region(scull_dev1.cdev.dev, 1);
        cdev_del(&scull_dev1.cdev);
        printk(KERN_ALERT "scull exit\n");
    }       /* -----  end of function scull_exit  ----- */
    /* this macro told to compiler.
     * that the two function are init function and exit function
     * 
     */

    module_init(scull_init);
    module_exit(scull_exit);
```





[1]: images/LDD/The_arguments_to_read.png
