@Date   : 2023年12月14日\
@Author : Tlx

在实际的驱动开发中，一般I2C主机控制器驱动由半导体厂家编写好，而驱动设备一般由设备器件的厂家编写好了，我们只需要提供设备信息即可。\
比如说I2C设备的话提供设备连接到了哪个I2C接口上，I2C的速度是多少等等，相当于将设备信息从设备驱动中分离出来，驱动使用标准方法去获取设备信息(比如从设备树中获取到信息)，然后根据获取到的设备信息来初始化设备。这样就相当于驱动只负责驱动，设备只负责设备，我们的主要目标就是匹配两者。\
即linux通过总线把驱动和设备给联系起来，当向系统注册驱动的时候，总线会尝试匹配对应的设备。同样，当向系统注册一个设备的时候，总线会尝试匹配对应的驱动。

### platform总线
Linux系统内核使用bus_type结构体表示总线，其中platform总线是bus_type的一个具体实例（在SOC中有些外设是没有总线的，为了使用驱动-总线-设备模型而提出platform虚拟总线）

```c++
struct bus_type platform_bus_type = {
.name = "platform",
.dev_groups = platform_dev_groups,
.match = platform_match,        // 匹配函数
.uevent = platform_uevent,
.pm = &platform_dev_pm_ops,
};
```
platform_driver结构体表示platform驱动
```c++
struct platform_driver {
    int (*probe)(struct platform_device *);  // 当驱动与设备匹配成功以后probe就会执行
    int (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
 };
 ```

 ### 编写使用dts的platfrom下的led驱动
 #### 1, 验证初始环境
 先构建好基本的驱动框架，搭建好Makefile，编译一下看有没有问题。（关于LED的dts节点注册问题放在最后的附录中）
```c++
// 驱动.c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of_gpio.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/irq.h>
#include <linux/wait.h>
#include <linux/poll.h>
#include <linux/fs.h>
#include <linux/fcntl.h>
#include <linux/platform_device.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

#define LEDDEV_CNT 1              
#define LEDDEV_NAME "dtsplatled"
#define LEDOFF 0
#define LEDON 1

// /* leddev设备结构体 */
struct leddev_dev{
    dev_t devid;            /* 设备号 */
    struct cdev cdev;       /* cdev */
    struct class *class;    /* 类 */
    struct device *device;  /* 设备 */
};

struct leddev_dev leddev;   /* led设备 */



static int __init led_init(void)
{
    return 0;
}

static void __exit led_exit(void)
{

}

module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("tlx");
 ```
 对应的Makefile如下
 ```Makefile
 KERNELDIR := /home/tlx/linux/linux-imx

CURRENT_PATH := $(shell pwd)
obj-m := platform_led.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
 ```
2, 先写出框架的主体，比如platform驱动结构体，match匹配函数，probe函数\
另外，修改led_init中的返回值，注册platform驱动
```c++
/* 移除函数 */
static int led_remove(struct platform_device *dev)
{
    return 0;
}

/* 当驱动与设备匹配后就会执行此函数 */
static int led_probe(struct platform_device * dev)
{
    return 0;
}

/* 匹配列表 */
static const struct of_device_id led_of_match[] = {
    { .compatible = "atkalpha-gpioled"},
    { /* Sentinel */}
};

/* platform驱动结构体 */
static struct platform_driver led_driver = {
    .driver = {
        .name = "imx6ull-led", /* 驱动名字，用于和设备匹配 */
        .of_match_table = led_of_match, /* 设备树匹配表 */
    },

    .probe = led_probe,
    .remove = led_remove,
};


static int __init led_init(void)
{
    return platform_driver_register(&led_driver);
}
```

3， 开始完善probe函数
```c++
/* 当驱动与设备匹配后就会执行此函数 */
static int led_probe(struct platform_device * dev)
{
    printk("led driver and device was matched\r\n");

    /* 1.设置设备号 */
    if (leddev.major){
        leddev.devid = MKDEV(leddev.major, 0);
        register_chrdev_region(leddev.devid, LEDDEV_CNT,LEDDEV_NAME);
    }else{ /* 自动分配 */
        alloc_chrdev_region(&leddev.devid,0,LEDDEV_CNT,LEDDEV_NAME);
        leddev.major = MAJOR(leddev.devid);
    }

    printk("2\r\n");
    /* 2. 注册设备 */
    cdev_init(&leddev.cdev, &led_fops); /* 文件操作集 */
    cdev_add(&leddev.cdev, leddev.devid,LEDDEV_CNT);

    /* 3. 创建类 */
    leddev.class = class_create(THIS_MODULE, LEDDEV_NAME);     
    if (IS_ERR(leddev.class))
    {
        return PTR_ERR(leddev.class);
    }
    printk("3\r\n");
    /* 4. 创建设备 */
    leddev.device = device_create(leddev.class, NULL, leddev.devid, NULL, LEDDEV_NAME);
    if(IS_ERR(leddev.device)){
        return PTR_ERR(leddev.device);
    }
    printk("4\r\n");
    printk("the device name is %s\n",leddev.device->kobj.name);

    /* 5. 初始化IO */
    leddev.node = of_find_node_by_path("/gpioled"); /* 获取对应的dts描述节点信息 */
    if (leddev.node == NULL){
        printk("gpioled node nost find! \r\n");
        return -EINVAL;
    }

    leddev.led0 = of_get_named_gpio(leddev.node, "led_gpio",0); /* 获取对应dts节点内的gpio描述 */
    if (leddev.led0 < 0){
        printk("can't get led_gpio\r\n");
        return -EINVAL;
    }

    gpio_request(leddev.led0, "led0"); /* 这里这个名字可以随便取来着 */
    gpio_direction_output(leddev.led0,1); /* 设置为输出，默认高电平 */
    return 0;
}
```
4, 添加led的文件操作集函数
```c++
/* led文件操作集函数部分 */
/* led转换功能 */
void led0_switch(u8 sta)
{
    if (sta == LEDON)
        gpio_set_value(leddev.led0,0);
    else if (sta == LEDOFF)
    {
        gpio_set_value(leddev.led0,1);
    }
}

/* 打开设备 */
static int led_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &leddev; /* 设置私有数据 */ 
    return 0;
}

/* 向设备写数据 */
static ssize_t led_write(struct file *filp, const char __user *buf, \
        size_t cnt, loff_t *offt)
{
    int retvalue;
    unsigned char databuf[2];
    unsigned char ledstat;

    /* 驱动这里是内核空间，需要把数据从用户空间拷贝过来 */
    retvalue = copy_from_user(databuf, buf, cnt);
    if (retvalue < 0)
    {
        printk("kernel write failed\r\n");
        return -EFAULT;
    }

    ledstat = databuf[0];
    if (ledstat == LEDON){
        led0_switch(LEDON);
    }else if(ledstat == LEDOFF)
        led0_switch(LEDOFF);

    return 0;
}
/* 设备操作函数 */
static struct file_operations led_fops = {
    .owner = THIS_MODULE,
    .open = led_open,
    .write = led_write,
};
/* led文件操作集函数部分 */
```
5， 编写应用程序，使用arm-linux-gnueabihf-gcc 编译，这里要注意的是，把可执行文件传送到开发板后，要给可执行文件添加权限，不然会提示权限不足的错误
```c++
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "stdlib.h"
#include "string.h"

#define LEDOFF 1
#define LEDON 0

/* 主函数 */
int main(int argc, char* argv[])
{
    int fd, retvalue;
    char *filename;
    unsigned char databuf[1];

    if(argc !=3)
    {
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];

    /* 打开led驱动 */
    fd = open(filename, O_RDWR);
    if(fd < 0)
    {
        printf("file %s open failed!\r\n", argv[1]);
        return -1;
    }

    databuf[0] = atoi(argv[2]); /* 要执行的操作 */

    /* 向/ded驱动 */
    retvalue = write(fd, databuf, sizeof(databuf));
    if(retvalue < 0)
    {
        printf("LED Control Failed!\r\n");
        close(fd);
        return -1;
    }

    retvalue = close(fd);
    if(retvalue < 0)
    {
        printf("file %s close failed!\r\n", argv[1]);
        return -1;
    }

    return 0;
}
```


### 踩坑点
#### 驱动的匹配过程
在驱动的编写过程中，我们可以注意到这样一个结构体
```c++
/* platform驱动结构体 */
static struct platform_driver led_driver = {
    .driver = {
        .name = "gpioled",         /* 驱动名字，用于和设备匹配 */
        .of_match_table = led_of_match, /* 设备树匹配表 */
    },

    .probe = led_probe,
    .remove = led_remove,
};
```

其中，这个.name用于设置驱动名字（如果of_match_table无法匹配的话这里的这个name会作为最后一道检查机制去匹配设备树中的同名设备）

这里是内核中的platform_match函数
```c++
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```

然后，在我们刚才编写的驱动初始化函数中，是有这样的一行
```c++
static int __init led_init(void)
{
    printk("hello\n");
    return platform_driver_register(&led_driver);
}
```

这个register函数会分配一个driver对象，设置的名字就是之前的.name，也就是说我们这个可以随便设置，只要保证这个.name或者match函数中的compatible中有一个能匹配上就行
```c++
/* 匹配列表 */
static const struct of_device_id led_of_match[] = {
    { .compatible = "tlxled" },
    { /* Sentinel */}
};

/* 这里是设备树根节点的内容 */
	gpioled {
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "tlxled";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_led>;
		led_gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;
		status = "okay";
	};
```