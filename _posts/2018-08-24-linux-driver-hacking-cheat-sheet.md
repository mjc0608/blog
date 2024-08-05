---
layout: post
title: Linux Driver Hacking Cheat Sheet
---



#### Reference:

1. [LDD3](https://lwn.net/Kernel/LDD3/)
2. Linux Kernel

## Char Device

#### Device Number

Device Number has to parts: `major` and `minor`.

In most cases, `major` represents the driver, and `minor` represents the specific device. `dev_t` stores `major` and `minor`. Use `MAJOR(dev_t dev)` and `MINOR(dev_t dev)` micros to access `major` and `minor`, and use `MKDEV(int major, int minor)` to construct `dev_t`.

```c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
```

Here, `first` is the beginning device number of the range you want to alloc. The `minor` of `first` is often 0. `name` is the name of the device, which will appear in `/proc/devices` and `/sysfs`.

```c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
```

Sometimes we do not know which number we want to use. Through this API the kernel can alloc a range of numbers for us. `dev` will hold the first number in the allocated range. `firstminor` is often `0`.

```c
#define MINORMASK ((1U << MINORBITS) - 1)
```

The micro `MINORMASK` can let us easily register or alloc all minors in a major.

```c
void unregister_chrdev_region(dev_t first, unsigned int count);
```

Unregister the number range.

#### File Operations

```c
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    /* ... */
};
```

The `file_operations`struct defines the file operations of what the kernel should do while userspace applications access the file.

container_of(pointer, container_type, container_field);

`container_of` is a micro based on some interesting feature of the C language and used to get the ‘parent’ struct of something. For example:

```c
/* Let's assume this is something we define in our driver */
struct efes_dev {
    struct efes_data *data;
    struct semaphore efes_lock;
    unsigned long efes_id;
    struct cdev cdev;
};
/* and in the code */
int efes_open(struct inode *inode, struct file *filp)
{
    struct efes_dev *dev;
    /* This line of code get the efes_dev struct! */
    dev = container_of(inode->i_cdev, struct efes_dev, cdev);
    /* Then we insert this data to filp, since only open and release has inode */
    filp->private_data = dev;
    /* do whatever we need here */
    /* ... */
}
```

#### Char Device Registration

```c
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops; /* my_fops is defined as the previous section */
```

This allocates a `cdev` struct and returns a pointer. However, sometimes we can allocate the struct ourselves, and use:

```c
void cdev_init(struct cdev *cdev, struct file_operations *fops);
```

to initialize it.

After initializing the `cdev` struct, it is important to set the `owner` field as `THIS_MODULE`, e.g.:

```c
my_cdev->owner = THIS_MODULE;
```

After finishing the above steps, we need to tell the kernel that we are ready by:

```c
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
```

Here `num` is the device number, and `count` is the number of devices (if any) that you want to register. Usually, `count` is `1`. Note that the name of the device is not need when calling this function, since the kernel can go find it through the `dev_t`, which is already registered before.

And unregistering it by:

```c
void cdev_del(struct cdev *dev);
```

There is an old way to register the device by using:

```c
int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);
```

Here the `major` is the major number, and the `name` is the name of the device. You do not need to create a `dev_t` for the device. This API creates `256` minors (0~255) for the major, and creates a cdev struct for each. The driver is supposed to handle all `open` calls on all 256 minors. If using this API, you need to unregister the char device with:

```c
int unregister_chrdev(unsigned int major, const char *name);
```

#### Access from Userspace

Till now the driver is not visiable in `/dev`. The user may use `mknod` system call to add it.

```bash
mknod /dev/$device c $major $minor
# here device is the name of the device, and c means char device
# need root permission
```

