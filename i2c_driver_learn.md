# I2C设备驱动：
##	1.分为总线驱动和设备驱动

###		1.1总线驱动： 抽象成了i2c_adapter这个数据结构
>		struct i2c_adapter{
>		...
>		const struct i2c_algorithm *algo;	//总线访问算法
>		...
>		}
i2c_algorithm中：
>		int (*master_xfer)(struct i2c_adapter *adap,struct i2c_msg *msgs, int num); //i2c适配器的传输函数,完成设备与i2c之间的通信

看看调用master_xfer的全过程：
首先是自己写的write_regs中调用i2c_transfer:
>		int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num){
>			if (in_atomic() || irqs_disabled()) {
>			ret = i2c_trylock_adapter(adap);
>			
>			...
>			ret = __i2c_transfer(adap, msgs, num);
>			i2c_unlock_adapter(adap);
>			...
>		}
继续查看__i2c_transfer：
>		int __i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num){
>			for (ret = 0, try = 0; try <= adap->retries; try++) {
>				ret = adap->algo->master_xfer(adap, msgs, num);		//这里调用了master_xfer，传输函数
>				if (ret != -EAGAIN)
>						break;
>				if (time_after(jiffies, orig_jiffies + adap->timeout))
>				       break;
>			}
>		}
最终在这里调用了master_xfer

还有很多static_key_false(...)的叫做静态键的函数，当静态键的状态为static_key_false时，相关的代码段将被编译进内核，但在运行时将被跳过，从而减少了不必要的执行开销。


之后i2c核心层调用i2c_add_adapter向系统注册adapter。i2c_adapter是一个核心模块，负责管理所有注册到系统的i2c总线适配器和设备，提供通信方式。
查看i2c_add_adapter函数：
>		int i2c_add_adapter(struct i2c_adapter *adapter){
>		...
>		if (dev->of_node) {
>		       id = of_alias_get_id(dev->of_node, "i2c");
>		       if (id >= 0) {
>					adapter->nr = id;
>					return __i2c_add_numbered_adapter(adapter);
>				}
>		}
>		...
>		return i2c_register_adapter(adapter);
>		}
先看看__i2c_add_numbered_adapter(adapter) , 注册一个已知id的i2c_adapter：
>		static int __i2c_add_numbered_adapter(struct i2c_adapter *adap)
>		{
>		int id;
>		
>		 mutex_lock(&core_lock);
>		id = idr_alloc(&i2c_adapter_idr, adap, adap->nr, adap->nr + 1,
>		           GFP_KERNEL);
>		mutex_unlock(&core_lock);
>		if (id < 0)
>		     return id == -ENOSPC ? -EBUSY : id;
>		
>		return i2c_register_adapter(adap);
>		}
最后也是调用了i2c_register_adapter:
>		static int i2c_register_adapter(struct i2c_adapter *adap)
>		{
>		...
>			dev_set_name(&adap->dev, "i2c-%d", adap->nr);
>		  	adap->dev.bus = &i2c_bus_type;
>		 	adap->dev.type = &i2c_adapter_type;
>		 	res = device_register(&adap->dev);
>		...
>		exit_recovery:
>		...
>		if (adap->nr < __i2c_first_dynamic_bus_num)
>         i2c_scan_static_board_info(adap);
>		...
>		}
这里将adapter注册到了i2c_bus_type上，之后调用了i2c_scan_static_board_info(adap)，进去看看是干什么的。

static void i2c_scan_static_board_info(struct i2c_adapter *adapter)，获取了i2c_devinfo，之后调用了i2c_new_device:
>		struct i2c_client *i2c_new_device(struct i2c_adapter *adap, struct i2c_board_info const *info)
>		{
>		...
>		client->dev.parent = &client->adapter->dev;
>		  client->dev.bus = &i2c_bus_type;
>		  client->dev.type = &i2c_client_type;
>		  client->dev.of_node = info->of_node;
>		  client->dev.fwnode = info->fwnode;
>		
>		  i2c_dev_set_name(adap, client);
>		  status = device_register(&client->dev);
>		...
>		}
在这里还注册了一个client设备到i2c总线上，类型为i2c_client_type。

i2c的设备驱动和总线驱动就这样联系起来了？应该是吧。还有很多细节没有深挖





###		1.2设备驱动：关注i2c_client /  i2c_driver
>		i2c_client就是真实的i2c物理设备 , i2c_driver是i2c驱动 ， 跟普通的字符设备匹配类似
>		
>		struct i2c_client {
>			unsigned short flags; /* 标志 */
>			unsigned short addr; /* 芯片地址， 7 位，存在低 7 位*/
>			......
>			char name[I2C_NAME_SIZE]; /* 名字 */
>			struct i2c_adapter *adapter; /* 对应的 I2C 适配器 */
>			struct device dev; /* 设备结构体 */
>			int irq; /* 中断 */
>			struct list_head detected;
>			......
>		};

>		struct i2c_driver {	//实现设备与总线的挂接
>			unsigned int class;
>			int (*attach_adapter)(struct i2c_adapter *) __deprecated;
>			int (*probe)(struct i2c_client *, const struct i2c_device_id *);	//与platfrom驱动的probe一样，匹配成功后就会执行，用于初始化
>			...
>			struct device_driver driver;	//设备驱动，用于与设备树match
>			const struct i2c_device_id *id_table;	//传统的id 不使用设备树的情况下可以match
>			......
>		};


##	2.设备和驱动匹配过程
首先注册adapter
>		int i2c_add_adapter(struct i2c_adapter *adapter)
>		int i2c_add_numbered_adapter(struct i2c_adapter *adap)
>		void i2c_del_adapter(struct i2c_adapter * adap)
注册driver
>		int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
>		int i2c_add_driver (struct i2c_driver *driver)
>		void i2c_del_driver(struct i2c_driver *driver)
在i2c_bus_type定义.match匹配方法：
与platfrom驱动匹配方式的类似
>		if (of_driver_match_device(dev, drv))	//设备树设备和驱动的匹配 匹配of_device_id的compatible
>		if (acpi_driver_match_device(dev, drv))	//acpi
>		if (driver->id_table)	return i2c_match_id(driver->id_table, client) != NULL;	//上面说的 直接匹配i2c_device_id的.name
之前在platform驱动学习中发现，只要.of_match_table匹配的上of_device_id的.compatible字段，驱动的.name字段即使不正确也能够匹配上，但是.name字段不能为空，因为注册的时候需要用到.name字段，否则驱动注册失败，也就无法匹配。

driver.of_match_table是匹配dts中的设备节点的（device_node，platfrom中有个resource是device_node的属性），匹配顺序依次是 compatible -> type -> name ，中一个就匹配成功 。 
>		是否强制选择(override) -> of_device_id匹配dts结点 -> 比较platform_device_id -> 最后才匹配driver.name 和 platform_device.name




##	3.ap3216c

>		以ap3216c传感器为例：
>		在dts中找到pinctrl_i2c1：设置IO复用

>		在i2c1节点内添加ap3216c属性，compatible和reg<0x1e>
>		ap3216c@1e {
>			compatible = "alientek,ap3216c";
>			reg = <0x1e>;
>		};
make dtbs , 用新的dtb打开，可以在sys/bus/i2c/devices下看到0-001e

modprobe时显示ap3216c busy，原因是linux内核中有一个ap3216驱动了，在Device Driver -> character driver -> Lite-On.. 取消掉再编译zImage
>		make zImage -j8

>		depmod
>		modprobe ap3216c.ko
>		./ap3216cApp /dev/ap3216c
>		可以看到一直在循环输出， 此时用手电筒对板子进行照射有变化





