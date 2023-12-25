## gpio子系统分析
gpio子系统是linux内核为了简化驱动开发者操作gpio而设计出来的子系统。驱动开发人员可以很方便的使用封装好的接口进行操作。

本文分析gpio子系统采用两种模式相结合。一个是从上往下，一个是从下往上。这里的上指的是用户接口层。下指的是dts

首先我们从上往下来看：
```c
	...
	ret = gpio_direction_output(3 , 1);//这样写的驱动是不通用的
	...
`````

没错，这个gpio_direction_output()就是linux内核gpio子系统提供的函数。这个函数是设置gpio1_IO03这个pin为高电平。

那么我们就以它没线索，一点点探究gpio子系统。

首先跟进这个函数去看：
```c
static inline int gpio_direction_output(unsigned gpio, int value)
{
	return gpiod_direction_output_raw(gpio_to_desc(gpio), value);
}
`````

继续跟进,这里只是简单的的调用了gpiod_direction_output_raw();

```c
int gpiod_direction_output_raw(struct gpio_desc *desc, int value)
{
	if (!desc || !desc->chip) {
		pr_warn("%s: invalid GPIO\n", __func__);
		return -EINVAL;
	}
	return _gpiod_direction_output_raw(desc, value);
}
`````

继续跟进:
```c
static int _gpiod_direction_output_raw(struct gpio_desc *desc, int value)
{
	struct gpio_chip	*chip;
	int			status = -EINVAL;

	/* GPIOs used for IRQs shall not be set as output */
	if (test_bit(FLAG_USED_AS_IRQ, &desc->flags)) {
		gpiod_err(desc,
			  "%s: tried to set a GPIO tied to an IRQ as output\n",
			  __func__);
		return -EIO;
	}

	/* Open drain pin should not be driven to 1 */
	if (value && test_bit(FLAG_OPEN_DRAIN,  &desc->flags))
		return gpiod_direction_input(desc);

	/* Open source pin should not be driven to 0 */
	if (!value && test_bit(FLAG_OPEN_SOURCE,  &desc->flags))
		return gpiod_direction_input(desc);

	chip = desc->chip;
	if (!chip->set || !chip->direction_output) {
		gpiod_warn(desc,
		       "%s: missing set() or direction_output() operations\n",
		       __func__);
		return -EIO;
	}

	//chip,这是linux内核gpio子系统核心部分和gpio驱动的那个接口层
	status = chip->direction_output(chip, gpio_chip_hwgpio(desc), value);
	if (status == 0)
		set_bit(FLAG_IS_OUT, &desc->flags);
	trace_gpio_value(desc_to_gpio(desc), 0, value);
	trace_gpio_direction(desc_to_gpio(desc), 0, status);
	return status;
}

`````

我们只看重点部分：
```c
status = chip->direction_output();
`````

这里面是指针调用了一个函数去继续往下走。这种模式在linux内核种非常常见。这个时候gpio_direction_output()这个函数算是走到底了，因为具体的chip->direction_output()我们是没有办法从上往下继续深究的。这也可以linux内核分层的体现，代码通过各种回调函数实现对下层细节，或者说对下层的封装。例如各种ops操作函数。

那么接下来我们从下网上去分析，重点放在gpio_chip这个结构体。从下网上需要从dts开始。这里我们以我们实验开发板的gpio1为例子进行说明。
```c
			gpio1: gpio@0209c000 {
				compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
				reg = <0x0209c000 0x4000>;
				interrupts = <GIC_SPI 66 IRQ_TYPE_LEVEL_HIGH>,
					     <GIC_SPI 67 IRQ_TYPE_LEVEL_HIGH>;
				gpio-controller;
				#gpio-cells = <2>;
				interrupt-controller;
				#interrupt-cells = <2>;
			};

`````

我们依照字符串"fsl,imx35-gpio"进行查找对应的驱动。
```c
static struct platform_driver mxc_gpio_driver = {
	.driver		= {
		.name	= "gpio-mxc",
		.of_match_table = mxc_gpio_dt_ids,
	},
	.probe		= mxc_gpio_probe,
	.id_table	= mxc_gpio_devtype,
};

`````

没错，这是一个platform驱动，我们继续看probe函数。
```c
static int mxc_gpio_probe(struct platform_device *pdev)
{
	struct device_node *np = pdev->dev.of_node;
	struct mxc_gpio_port *port;
	struct resource *iores;
	int irq_base;
	int err;

	mxc_gpio_get_hw(pdev);

	port = devm_kzalloc(&pdev->dev, sizeof(*port), GFP_KERNEL);
	if (!port)
		return -ENOMEM;

	//从dts拿内存资源
	iores = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	//帮你映射好
	port->base = devm_ioremap_resource(&pdev->dev, iores);
	if (IS_ERR(port->base))
		return PTR_ERR(port->base);

	//中断相关
	port->irq_high = platform_get_irq(pdev, 1);
	port->irq = platform_get_irq(pdev, 0);
	if (port->irq < 0)
		return port->irq;

	/* disable the interrupt and clear the status */
	writel(0, port->base + GPIO_IMR);
	writel(~0, port->base + GPIO_ISR);

	if (mxc_gpio_hwtype == IMX21_GPIO) {
		/*
		 * Setup one handler for all GPIO interrupts. Actually setting
		 * the handler is needed only once, but doing it for every port
		 * is more robust and easier.
		 */
		irq_set_chained_handler(port->irq, mx2_gpio_irq_handler);
	} else {
		/* setup one handler for each entry */
		irq_set_chained_handler(port->irq, mx3_gpio_irq_handler);
		irq_set_handler_data(port->irq, port);
		if (port->irq_high > 0) {
			/* setup handler for GPIO 16 to 31 */
			irq_set_chained_handler(port->irq_high,
						mx3_gpio_irq_handler);
			irq_set_handler_data(port->irq_high, port);
		}
	}

	err = bgpio_init(&port->bgc, &pdev->dev, 4,
			 port->base + GPIO_PSR,
			 port->base + GPIO_DR, NULL,
			 port->base + GPIO_GDIR, NULL, 0);
	if (err)
		goto out_bgio;

	port->bgc.gc.to_irq = mxc_gpio_to_irq;
	port->bgc.gc.base = (pdev->id < 0) ? of_alias_get_id(np, "gpio") * 32 :
					     pdev->id * 32;

	//这里面肯定初始化了gpio_chip结构体，里面包含了操作函数
	err = gpiochip_add(&port->bgc.gc);
	if (err)
		goto out_bgpio_remove;

	irq_base = irq_alloc_descs(-1, 0, 32, numa_node_id());
	if (irq_base < 0) {
		err = irq_base;
		goto out_gpiochip_remove;
	}

	port->domain = irq_domain_add_legacy(np, 32, irq_base, 0,
					     &irq_domain_simple_ops, NULL);
	if (!port->domain) {
		err = -ENODEV;
		goto out_irqdesc_free;
	}

	/* gpio-mxc can be a generic irq chip */
	mxc_gpio_init_gc(port, irq_base);

	//加到了一个全局链表里面，作为整个gpio控制器的管理，里面肯定包含控制函数，*ops一个函数列表，用于操作这个gpio
	list_add_tail(&port->node, &mxc_gpio_ports);

	return 0;

out_irqdesc_free:
	irq_free_descs(irq_base, 32);
out_gpiochip_remove:
	gpiochip_remove(&port->bgc.gc);
out_bgpio_remove:
	bgpio_remove(&port->bgc);
out_bgio:
	dev_info(&pdev->dev, "%s failed with errno %d\n", __func__, err);
	return err;
}


`````

我们只关注与gpio_chip有关的部分。我们可以在代码中可以看到，后面调用了gpiochip_add()函数注册了一个gpio_chip结构体，那么到这里你是否恍然大悟，因为好像跟从上往下走的那条线对接上了。那么我们看这个结构体是否有对应的chip->direction_output()函数呢，按道理来说，应该会被初始化。在代码中我们跟进bgpio_init()函数去看，因为第一个参数就是包含了gpio_chip结构体。
```c

int bgpio_init(struct bgpio_chip *bgc, struct device *dev,
	       unsigned long sz, void __iomem *dat, void __iomem *set,
	       void __iomem *clr, void __iomem *dirout, void __iomem *dirin,
	       unsigned long flags)
{
	int ret;

	if (!is_power_of_2(sz))
		return -EINVAL;

	bgc->bits = sz * 8;
	if (bgc->bits > BITS_PER_LONG)
		return -EINVAL;

	spin_lock_init(&bgc->lock);
	bgc->gc.dev = dev;
	bgc->gc.label = dev_name(dev);
	bgc->gc.base = -1;
	bgc->gc.ngpio = bgc->bits;
	bgc->gc.request = bgpio_request;

	ret = bgpio_setup_io(bgc, dat, set, clr);
	if (ret)
		return ret;

	ret = bgpio_setup_accessors(dev, bgc, flags & BGPIOF_BIG_ENDIAN,
				    flags & BGPIOF_BIG_ENDIAN_BYTE_ORDER);
	if (ret)
		return ret;

		//这里设置了回调函数
	ret = bgpio_setup_direction(bgc, dirout, dirin);
	if (ret)
		return ret;

	bgc->data = bgc->read_reg(bgc->reg_dat);
	if (bgc->gc.set == bgpio_set_set &&
			!(flags & BGPIOF_UNREADABLE_REG_SET))
		bgc->data = bgc->read_reg(bgc->reg_set);
	if (bgc->reg_dir && !(flags & BGPIOF_UNREADABLE_REG_DIR))
		bgc->dir = bgc->read_reg(bgc->reg_dir);

	return ret;
}
`````

我们继续跟进bgpio_setup_direction()函数去看。
```c
static int bgpio_setup_direction(struct bgpio_chip *bgc,
				 void __iomem *dirout,
				 void __iomem *dirin)
{
	if (dirout && dirin) {
		return -EINVAL;
	} else if (dirout) {
		bgc->reg_dir = dirout;
		bgc->gc.direction_output = bgpio_dir_out;
		bgc->gc.direction_input = bgpio_dir_in;
	} else if (dirin) {
		bgc->reg_dir = dirin;
		bgc->gc.direction_output = bgpio_dir_out_inv;
		bgc->gc.direction_input = bgpio_dir_in_inv;
	} else {
		bgc->gc.direction_output = bgpio_simple_dir_out;
		bgc->gc.direction_input = bgpio_simple_dir_in;
	}

	return 0;
}
`````

不知道你看到没有，反正我看到这个函数的时候很兴奋，因为确实在这里对gpio_chip->direction_output进行了赋值，这里赋值为bgpio_dir_out_inv()函数。没错，完全对接上了，形成闭环了。

这部分只是gpio子系统的冰山一角，我将继续往下一点一点剥开它的全貌。一起期待。
