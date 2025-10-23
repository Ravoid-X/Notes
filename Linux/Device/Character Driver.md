## 设备号
每个设备都由一个设备号（dev_t）唯一标识，它由主设备号和次设备号组成
### 主设备号
标识设备所属的驱动程序。当用户对一个设备文件进行操作时，内核会根据其主设备号，将请求分派给正确的驱动程序。
### 次设备号
1. 由驱动程序内部使用，用于区分它所管理的多个同类型设备实例。
2. 例如，一个多串口卡的驱动程序可能会用不同的次设备号来标识卡上的每一个物理端口。
## 注册接口
现代内核推荐使用动态方式注册设备号，以避免与系统中已存在的设备发生冲突。
### 分配设备号区域
1. 使用 alloc_chrdev_region() 向内核申请一个或多个设备号
2. 通常将 firstminor 设为0，count 设为所需设备数量，内核会自动选择一个未被使用的主设备号，并将第一个设备号存入 dev 所指向的变量中。
```
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, const char *name);
```
### 初始化字符设备结构
使用 struct cdev 结构体来代表一个字符设备。它将设备号与驱动程序的文件操作函数集连接起来
```
//初始化 cdev 结构体，并将其与 file_operations 结构体关联
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```
### 添加字符设备
初始化完成后，调用 cdev_add() 函数将设备正式注册到内核中，使其对系统可见
```
//成功返回，用户空间程序就可以通过对应的设备文件来访问它
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```
### 删除
调用相应的清理函数 cdev_del() 和 unregister_chrdev_region() 来释放这些资源
### 示例
```
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/kernel.h>

#define DEVICE_NAME "skeleton_dev"
#define DEVICE_COUNT 1

static dev_t dev_num;
static struct cdev my_cdev;

static int __init skeleton_init(void) {
    int ret;
    // 1. 分配设备号
    ret = alloc_chrdev_region(&dev_num, 0, DEVICE_COUNT, DEVICE_NAME);
    if (ret < 0) {
        pr_err("Failed to allocate device number\n");
        return ret;
    }
    pr_info("Allocated device number: Major=%d, Minor=%d\n", MAJOR(dev_num), MINOR(dev_num));
    // 2. 初始化 cdev 结构
    cdev_init(&my_cdev, NULL); // fops 暂时为 NULL
    my_cdev.owner = THIS_MODULE;
    // 3. 添加 cdev 到内核
    ret = cdev_add(&my_cdev, dev_num, DEVICE_COUNT);
    if (ret < 0) {
        pr_err("Failed to add cdev\n");
        unregister_chrdev_region(dev_num, DEVICE_COUNT);
        return ret;
    }
    pr_info("Skeleton device driver loaded\n");
    return 0;
}

static void __exit skeleton_exit(void) {
    // 卸载与加载顺序相反
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, DEVICE_COUNT);
    pr_info("Skeleton device driver unloaded\n");
}

module_init(skeleton_init);
module_exit(skeleton_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Expert Team");
MODULE_DESCRIPTION("A skeleton character device driver");
```
## file_operations
字符驱动程序的核心，是一个包含函数指针的结构体，定义了驱动程序如何响应对设备文件的各种系统调用。
### 结构
```
static const struct file_operations my_fops = {
  .owner   = THIS_MODULE,
  .open    = my_open,
  .release = my_release,
  .read    = my_read,
  .write   = my_write,
    /*... 其他操作... */
};
```
## 核心驱动功能
### open
1. 当进程首次打开设备文件时，VFS 会调用驱动的 open 方法。
2. 通常用于执行设备相关的初始化、检查设备状态、分配私有数据结构等。
3. 至关重要的任务是调用 try_module_get(THIS_MODULE) 来增加模块的引用计数，这可以防止模块在被使用时被意外卸载。   
### release
1. 当最后一个持有设备文件描述符的进程关闭它时，release 方法被调用。
2. 负责执行清理工作，如释放 open 方法中分配的资源，并调用 module_put(THIS_MODULE) 来减少模块的引用计数。
### read & write
驱动程序数据交互的核心，函数原型定义了与用户空间交互的标准接口
```
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
```
1. struct file *filp：代表一个打开的文件，驱动可以用它来存储私有数据（filp->private_data）。
2. char __user *buf：指向用户空间的缓冲区。__user 是一个特殊的宏，用于静态分析工具（如 sparse）检查，提醒开发者这是一个不能直接解引用的用户空间地址。
3. size_t len：用户期望读或写的字节数。
4. loff_t *off：指向文件偏移量的指针，驱动需要根据它来读写正确的位置，并更新它。
### copy_to_user()
从内核空间拷贝 n 字节数据到用户空间
```
copy_to_user(void __user *to, const void *from, unsigned long n);
```
### copy_from_user()
从用户空间拷贝 n 字节数据到内核空间
```
copy_from_user(void *to, const void __user *from, unsigned long n);
```
### copy 传输验证
1. 地址验证：调用 access_ok() 宏来检查用户提供的地址范围是否完全位于当前进程的用户地址空间内。
2. 异常处理\
（1）实际的数据拷贝操作是在一个特殊的、受内核异常处理程序监控的上下文中执行的。\
（2）如果在拷贝过程中发生页面错误，例如因为用户提供了一个无效的地址，内核的缺页异常处理程序会检查到错误发生在这一特殊区域。\
（3）此时，它会中止拷贝操作，并让 copy_*_user() 函数返回一个非零值，表示有多少字节未能成功拷贝。
## “回显”字符驱动程序
任何写入该设备的数据都会被存储在一个内核缓冲区中，当从设备读取时，这些数据会被返回给用户。
```
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/kernel.h>
#include <linux/uaccess.h> // for copy_to_user/copy_from_user

#define DEVICE_NAME "echo_dev"
#define DEVICE_COUNT 1
#define BUFFER_SIZE 256

static dev_t dev_num;
static struct cdev my_cdev;
static char device_buffer;
static int buffer_len = 0;

static int echo_open(struct inode *inode, struct file *filp) {
    try_module_get(THIS_MODULE);
    pr_info("Echo device opened\n");
    return 0;
}
static int echo_release(struct inode *inode, struct file *filp) {
    module_put(THIS_MODULE);
    pr_info("Echo device closed\n");
    return 0;
}
static ssize_t echo_read(struct file *filp, char __user *buf, size_t len, loff_t *off) {
    int bytes_to_read;
    if (*off >= buffer_len) {
        return 0; // End of file
    }
    bytes_to_read = min(len, (size_t)(buffer_len - *off));
    if (copy_to_user(buf, device_buffer + *off, bytes_to_read)!= 0) {
        return -EFAULT;
    }
    *off += bytes_to_read;
    pr_info("Read %d bytes from device\n", bytes_to_read);
    return bytes_to_read;
}
static ssize_t echo_write(struct file *filp, const char __user *buf, size_t len, loff_t *off) {
    int bytes_to_write;
    bytes_to_write = min(len, (size_t)BUFFER_SIZE);
    if (copy_from_user(device_buffer, buf, bytes_to_write)!= 0) {
        return -EFAULT;
    }
    buffer_len = bytes_to_write;
    *off = 0; // Reset offset for subsequent reads
    pr_info("Wrote %d bytes to device\n", bytes_to_write);
    return bytes_to_write;
}
static const struct file_operations echo_fops = {
  .owner   = THIS_MODULE,
  .open    = echo_open,
  .release = echo_release,
  .read    = echo_read,
  .write   = echo_write,
};
static int __init echo_init(void) {
    int ret;
    ret = alloc_chrdev_region(&dev_num, 0, DEVICE_COUNT, DEVICE_NAME);
    if (ret < 0) return ret;
    cdev_init(&my_cdev, &echo_fops);
    ret = cdev_add(&my_cdev, dev_num, DEVICE_COUNT);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, DEVICE_COUNT);
        return ret;
    }
    pr_info("Echo device driver loaded. Major=%d\n", MAJOR(dev_num));
    return 0;
}
static void __exit echo_exit(void) {
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, DEVICE_COUNT);
    pr_info("Echo device driver unloaded\n");
}

module_init(echo_init);
module_exit(echo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Expert Team");
MODULE_DESCRIPTION("A simple echo character device driver");
```
## 编译
为外部内核模块编写 Makefile，有特定的格式，利用了内核的 kbuild 系统 
### `obj-m := my_driver.o`
1. 最核心的一行。告诉 kbuild 系统，需要将 my_driver.o 编译成一个可加载模块（.ko文件）
2. 如果模块由多个源文件组成，则使用 `my_driver-y := file1.o file2.o` 来指定其依赖的-目标文件 。
### `make -C <kernel_dir> M=$PWD modules`
编译外部模块的标准命令
1. `-C <kernel_dir>`：指示make命令首先切换到指定的内核源码树目录。这个目录通常是 `/lib/modules/$(shell uname -r)/build`，它包含了编译模块所需的头文件和配置信息。   
2. `M=$PWD`：告诉 kbuild 系统外部模块的源代码位于当前目录（`$PWD`）。   
3. modules：是 kbuild 系统中的一个目标，用于编译在 obj-m 中指定的模块。
## 创建设备节点
编译并加载驱动后，还需要在 /dev 目录下创建一个设备文件，用户程序才能通过它与驱动交互
### 手动方法
需要从 dmesg 的输出中找到驱动加载时分配的主设备号
```
//c 表示字符设备，<major_number> 是驱动的主设备号，0 是次设备号 
sudo mknod /dev/echo_dev c <major_number> 0
```
### 自动方法
1. 现代 Linux 系统使用 udev 守护进程来自动管理 /dev 目录
2. 在驱动初始化时调用 class_create() 和 device_create() 来完成。
3. 模块加载时，这些函数会在 sysfs 中创建相应的条目，udev 会监测到这些变化，并根据预设的规则自动创建具有正确名称和权限的设备文件