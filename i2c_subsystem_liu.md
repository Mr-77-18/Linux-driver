## i2c子系统分析
i2c子系统的分析我们依然采用分析gpio子系统那一套方法：从上到下，从下到上相结合的方式。

本文分析的实例是开发板上一个传感器：ap3216c。它挂在开发板的i2c2上。

那么从上到下时，我们从ap3216c的驱动作为入口，看该驱动是怎么与i2c交互的。

```c
/* 设备树匹配列表 */
static const struct of_device_id ap3216c_of_match[] = {
	{.compatible = "alientek,ap3216c"},
	{ /* Sentinel */ }
};

/* i2c驱动结构体 */	
static struct i2c_driver ap3216c_driver = {
	.probe = ap3216c_probe,
	.remove = ap3216c_remove,
	.driver = {
			.owner = THIS_MODULE,
		   	.name = "ap3216c",
		   	.of_match_table = ap3216c_of_match, 
		   },
	.id_table = ap3216c_id,
};
	
`````

没错，这个驱动就是i2c驱动模型。那么我们重点看probe()函数。
```c
static int ap3216c_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
	printk("in probe\r\n");
	/* 1、构建设备号 */
	if (ap3216cdev.major) {
		ap3216cdev.devid = MKDEV(ap3216cdev.major, 0);
		register_chrdev_region(ap3216cdev.devid, AP3216C_CNT, AP3216C_NAME);
	} else {
		alloc_chrdev_region(&ap3216cdev.devid, 0, AP3216C_CNT, AP3216C_NAME);
		ap3216cdev.major = MAJOR(ap3216cdev.devid);
	}

	printk("the major is %d\r\n" , ap3216cdev.major);

	/* 2、注册设备 */
	cdev_init(&ap3216cdev.cdev, &ap3216c_ops);
	cdev_add(&ap3216cdev.cdev, ap3216cdev.devid, AP3216C_CNT);

	/* 3、创建类 */
	ap3216cdev.class = class_create(THIS_MODULE, AP3216C_NAME);
	if (IS_ERR(ap3216cdev.class)) {
		return PTR_ERR(ap3216cdev.class);
	}

	/* 4、创建设备 */
	ap3216cdev.device = device_create(ap3216cdev.class, NULL, ap3216cdev.devid, NULL, AP3216C_NAME);
	if (IS_ERR(ap3216cdev.device)) {
		return PTR_ERR(ap3216cdev.device);
	}

	ap3216cdev.private_data = client;

	return 0;
}
`````

我们看到这个probe函数里面是注册了一个字符设备，这里没有涉及到具体的i2c操作函数。那么我们得看注册的字符设备的操作函数。因为字符设备需要实例化file_operations函数表，那么里面的函数应该会涉及到i2c操作。

我们看到这里cdev_init(&ap3216cdev.cdev , &ap3216c_ops),没错，这里面注册了ap3216c_ops这个file_operations，我们看看这个ops具体包含那些函数，然后挑一个去分析一下。
```c
static const struct file_operations ap3216c_ops = {
.owner = THIS_MODULE,
.open = ap3216c_open,
.read = ap3216c_read,
.release = ap3216c_release,
};

`````
ap3216c_ops提供了read函数，而没有提供write函数，这个其实也是可以理解。因为ap3216c是一个传感器，我们只需要从传感器里面读取数据就可以了。

那么我们具体看一下ap3216c_read()在干什么。
```c
static ssize_t ap3216c_read(struct file *filp, char __user *buf, size_t cnt, loff_t *off)
{
	short data[3];
	long err = 0;

	struct ap3216c_dev *dev = (struct ap3216c_dev *)filp->private_data;
	
	//这个函数从传感器读取数据
	ap3216c_readdata(dev);

	//读取到的数据保存在了struct ap3216c_dev数据结构下
	data[0] = dev->ir;
	data[1] = dev->als;
	data[2] = dev->ps;
	err = copy_to_user(buf, data, sizeof(data));
	return 0;
}
`````

我们可以看到ap3216c_read()又调用了ap3216c_readdata()来读取数据。我们继续跟进去看。
```c
void ap3216c_readdata(struct ap3216c_dev *dev)
{
	unsigned char i =0;
    unsigned char buf[6];
	
	/* 循环读取所有传感器数据 */
    for(i = 0; i < 6; i++)	
    {
        buf[i] = ap3216c_read_reg(dev, AP3216C_IRDATALOW + i);	
    }

    if(buf[0] & 0X80) 	/* IR_OF位为1,则数据无效 */
		dev->ir = 0;					
	else 				/* 读取IR传感器的数据   		*/
		dev->ir = ((unsigned short)buf[1] << 2) | (buf[0] & 0X03); 			
	
	dev->als = ((unsigned short)buf[3] << 8) | buf[2];	/* 读取ALS传感器的数据 			 */  
	
    if(buf[4] & 0x40)	/* IR_OF位为1,则数据无效 			*/
		dev->ps = 0;    													
	else 				/* 读取PS传感器的数据    */
		dev->ps = ((unsigned short)(buf[5] & 0X3F) << 4) | (buf[4] & 0X0F); 
}
`````

没错，这里面调用ap3216c_read_reg()函数去读取传感器的数据。我们看传入的参数，第一个参数是**dev** ，这个应该是作为数据存储的地方；第二个参数AP3216C_IRDATALOW对应的值是

```c
static unsigned char ap3216c_read_reg(struct ap3216c_dev *dev, u8 reg)
{
	u8 data = 0;

	ap3216c_read_regs(dev, reg, &data, 1);
	return data;
}
`````

```c
static int ap3216c_read_regs(struct ap3216c_dev *dev, u8 reg, void *val, int len)
{
	int ret;
	struct i2c_msg msg[2];
	struct i2c_client *client = (struct i2c_client *)dev->private_data;

	/* msg[0]为发送要读取的首地址 */
	msg[0].addr = client->addr;			/* ap3216c地址 */
	msg[0].flags = 0;					/* 标记为发送数据 */
	msg[0].buf = &reg;					/* 读取的首地址 */
	msg[0].len = 1;						/* reg长度*/

	/* msg[1]读取数据 */
	msg[1].addr = client->addr;			/* ap3216c地址 */
	msg[1].flags = I2C_M_RD;			/* 标记为读取数据*/
	msg[1].buf = val;					/* 读取数据缓冲区 */
	msg[1].len = len;					/* 要读取的数据长度*/

	ret = i2c_transfer(client->adapter, msg, 2);
	if(ret == 2) {
		ret = 0;
	} else {
		printk("i2c rd failed=%d reg=%06x len=%d\n",ret, reg, len);
		ret = -EREMOTEIO;
	}
	return ret;
}

`````
我么可以看到这里面准备了要读取数据的地址，以及读取数据缓冲区。最后调用i2c_transfer()函数，这个函数属于i2c子系统的核心部分了，至此我们进入到了i2c核心层。

我们传给i2c_transfer的第一个参数是一个adapter,第二个参数包含了要读取数据的地址，以及读取后数据存放地址。最后一个表示msg的大小。

这里肯定会产生疑问对不对，好好的怎么突然冒出个adapter，这个是什么玩意，要读取数据的地址等信息不是给到了吗，为什么还需要一个adapter?这是因为一个SOC中包含多个i2c控制器，一个设备会被挂在不同的i2c上，比如这里的传感器是挂在了i2c2上，而开发板中还包含了i2c1,i2c3...，所以这个adapter相当于指定了是哪一个i2c。有了这个信息之后，就可以操作挂在ap3216c这个传感器的i2c了。

**那么看到这里，应该还是会产生一个疑问吧，反正我看到这里就会想到这个adapter是从哪里来的，也就是怎么产生的，又是由谁传给这个驱动的** 
1. 怎么产生的:i2c驱动
2. 谁传给驱动的:driver_register()

我们先看第一个问题，怎么产生的，我们看到adapter是保存在了dev的private_data当中，而这个private_data在ap3216c_probe函数的最后被赋予成struct i2c_client,这个结构是由其它人传过来的，也就是第二个问题中的那个"人",这里我们先不深究是谁传过来的，只需要知道这个adapter保存在了struct i2c_client结构体里面。

接下来我们从下往上看，看过gpio子系统分析的应该都知道这里的下指的是dts，那么我们从这里的i2c2开始看起，因为传感器挂接到了i2c2上。
```c
			i2c2: i2c@021a4000 {
				#address-cells = <1>;
				#size-cells = <0>;
				compatible = "fsl,imx6ul-i2c", "fsl,imx21-i2c";
				reg = <0x021a4000 0x4000>;
				interrupts = <GIC_SPI 37 IRQ_TYPE_LEVEL_HIGH>;
				clocks = <&clks IMX6UL_CLK_I2C2>;
				status = "disabled";
			};

`````

我们依靠字符串"fsl,imx21-i2c"查找对应的驱动。
```c
static struct platform_driver i2c_imx_driver = {
	.probe = i2c_imx_probe,
	.remove = i2c_imx_remove,
	.driver	= {
		.name = DRIVER_NAME,
		.owner = THIS_MODULE,
		.of_match_table = i2c_imx_dt_ids,
		.pm = IMX_I2C_PM,
	},
	.id_table	= imx_i2c_devtype,
};
`````

按照传统，继续分析probe函数.由于probe函数比较长，这里只给出关键部分，这里的关键部分是我们关心的struct i2c_client以及struct i2c_adapter的内容.
```c
static int i2c_imx_probe(struct platform_device *pdev)
{
	const struct of_device_id *of_id = of_match_device(i2c_imx_dt_ids,
							   &pdev->dev);
	struct imx_i2c_struct *i2c_imx;

	i2c_imx->adapter.owner		= THIS_MODULE;
	i2c_imx->adapter.algo		= &i2c_imx_algo;
	i2c_imx->adapter.dev.parent	= &pdev->dev;
	i2c_imx->adapter.nr		= pdev->id;
	i2c_imx->adapter.dev.of_node	= pdev->dev.of_node;
	i2c_imx->base			= base;
	/* Add I2C adapter */
	ret = i2c_add_numbered_adapter(&i2c_imx->adapter);
	if (ret < 0) {
		dev_err(&pdev->dev, "registration failed\n");
		goto clk_disable;
	}

	...
}
`````

可以看到其中的关键函数是i2c_add_numbered_adapter(),它的参数就是struct i2c_adapter,这个adapter被设置成为了i2c_imx_algo。我们等下再看具体的i2c_imx_algo。先看一下i2c_add_numbered_adapter()干了什么。
```c
int i2c_add_numbered_adapter(struct i2c_adapter *adap)
{
	if (adap->nr == -1) /* -1 means dynamically assign bus id */
		return i2c_add_adapter(adap);

	return __i2c_add_numbered_adapter(adap);
}
`````

没错，这个函数里面就是注册了struct i2c_adapter。到目前位置，我们解决了第一个问题，也就是怎么产生的。这里可以看到是i2c驱动初始化并注册了这个adapter。

我们最后看一下这个i2c_imx_algo的内容：
```c
static struct i2c_algorithm i2c_imx_algo = {
	.master_xfer	= i2c_imx_xfer,
	.functionality	= i2c_imx_func,
};

`````

不知道你看到这个master_xfer有没有觉得很熟悉。因为这个就是i2c_transfer最终调用的函数。现在是不是连接起来了。

最后只剩下第二个问题，是谁传给驱动的，答案是driver_register()
```c
driver_register(drv) [core.c]
  bus_add_driver(drv) [bus.c]
    if (drv->bus->p->drivers_autoprobe)
      driver_attach(dev)[dd.c]
        bus_for_each_dev(dev->bus, NULL, drv,__driver_attach)
        __driver_attach(dev, drv) [dd.c]
          driver_match_device(drv, dev) [base.h]
            drv-bus->match ? drv->bus-amatch(dev, drv) : 1
            if false, return;
          driver_probe_device(drv, dev) [dd.c]
            really_probe(dev, drv) [dd.c]
              dev-driver = drv;
              if (dev-bus->probe)
                dev->bus->probe(dev);
              else if (drv->probe)
                drv-aprobe(dev);
              probe_failed:
                dev->-driver = NULL;
`````

我们可以看到在调用probe函数的时候，将包含了adapter的dev传给了probe函数。
