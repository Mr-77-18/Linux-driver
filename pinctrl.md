## pinctrl子系统
本文讲述我对于pinctrl子系统的简单理解。在讲解之前，我希望大家进入下面这样一个场景，带着问题跟我一起去探究pinctrl子系统的原理。

场景：

实现led的点灯（万能led)

有设备树描述如下：
```c
led{
	#address_cells = <1>;
	#size_cells = <1>;
	compatible = "atkalphs-led";
	pinctrl-names = "defaule";
	pinctrl-0 = <&pinctrl_gpio_leds>;
	led-gpio = <&gpio 3 GPIO_ACTIVE_HIGH>;
	status = "okay";

...

&iomux {
	compatible = "fsl,imx6ul-iomuxc";
	...
	
	&pinctrl_gpio_leds: gpio-leds{
		fsl,pins = < MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0x17059>;
	};

	...
}
`````
从上面的Dts可以看出，led接到了gpio1的IO03这个Pin上面。功能复用是GPIO，那么我们知道要正确操作led之前，必须先设置pin的复用功能：即把gpio1的IO03这条Pin的功能设置成GPIO。那么现在抛出一个问题：这个设置动作通常是由谁设置的，在什么时候设置的，这个需要设置的寄存器信息以及数据在哪呢？带着这些问题，我们一步一步往下走。

既然我们是探索pinctrl子系统，相必大家都知道了，这个pin的设置就是由内核里面的pinctrl子系统去设置的。那么我们来看一下这个pinctrl子系统在这个场景下的入口是在哪里：

从dts中iomux这个词大概知道是用来处理Io复用的，那么我们不妨看一下这个节点是被如何处理的。怎么找呢，淡然是利用compatible属性中的值去找啦。最终我们找到这样一个文件：
```c
//drivers/pinctrl/freescale/pinctrl-imx6ul.c
static struct of_device_id imx6ul_pinctrl_of_match[] = {
  { .compatible = "fsl*mx6ul-iomuxc", .data = &imx6ul_pinctrl_info, },
  { .compatible = "fsl*mx6ull-iomuxc-snvs", .data = &imx6ull_snvs_pinctrl_info, },
  { /* sentinel */  }
};

static struct platform_driver imx6ul_pinctrl_driver = {
  .driver = {
    .name = "imx6ul-pinctrl",
    .owner = THIS_MODULE,
    .of_match_table = of_match_ptr(imx6ul_pinctrl_of_match),
  
},
  .probe = imx6ul_pinctrl_probe,
  .remove = imx_pinctrl_remove,

};
`````
没错，有一个plaform_driver来处理这个节点。那么我们继续看它是怎么被处理的。
```c

static int imx6ul_pinctrl_probe(struct platform_device *pdev)
{
  const struct of_device_id *match;
  struct imx_pinctrl_soc_info *pinctrl_info;

  match = of_match_device(imx6ul_pinctrl_of_match, &pdev->dev);

  if (!match)
    return -ENODEV;

  pinctrl_info = (struct imx_pinctrl_soc_info *) match->data;

  return imx_pinctrl_probe(pdev, pinctrl_info);

}


`````

这里面调用了imx_pinctrl_probe()对pin进行处理
```c
int imx_pinctrl_probe(struct platform_device *pdev , struct imx_pinctrl_soc_info* info){
  ...
  ret = imx_pinctrl_probe_dt(pdev , info);
  ...
  ipctl-pctl = pinctrl_register(imx_pinctrl_desc , &pdev->dev , ipctl);
}
`````

继续跟进
```c
static int imx_pinctrl_probe_dt(struct platform_device *pdev,
        struct imx_pinctrl_soc_info *info)
{
  struct device_node *np = pdev->dev.of_node;
  struct device_node *child;
  u32 nfuncs = 0;
  u32 i = 0;

  if (!np)
    return -ENODEV;

  nfuncs = of_get_child_count(np);
  if (nfuncs <= 0) {
    dev_err(&pdev->dev, "no functions defined\n");
    return -EINVAL;
  }

  info->nfunctions = nfuncs;
  info->functions = devm_kzalloc(&pdev->dev, nfuncs * sizeof(struct imx_pmx_func),
          GFP_KERNEL);
  if (!info->functions)
    return -ENOMEM;

  info->ngroups = 0;
  for_each_child_of_node(np, child)
    info->ngroups += of_get_child_count(child);
  info->groups = devm_kzalloc(&pdev->dev, info->ngroups * sizeof(struct imx_pin_group),
          GFP_KERNEL);
  if (!info->groups)
    return -ENOMEM;

  for_each_child_of_node(np, child)
    imx_pinctrl_parse_functions(child, info, i++);//在这里被设置

  return 0;
}
`````

继续跟进

```c

static int imx_pinctrl_parse_functions(struct device_node *np,
               struct imx_pinctrl_soc_info *info,
               u32 index)
{ 
  struct device_node *child;
  struct imx_pmx_func *func;
  struct imx_pin_group *grp;
  u32 i = 0;
  
  dev_dbg(info->dev, "parse function(%d): %s\n", index, np->name);
  
  func = &info->functions[index];
  
  /* Initialise function */
  func->name = np->name;
  func->num_groups = of_get_child_count(np);
  if (func->num_groups == 0) {
    dev_err(info->dev, "no groups defined in %s\n", np->full_name);
    return -EINVAL;
  }
  func->groups = devm_kzalloc(info->dev, 
      func->num_groups * sizeof(char *), GFP_KERNEL);
  
  for_each_child_of_node(np, child) {
    func->groups[i] = child->name;
    grp = &info->groups[info->grp_index++];
    imx_pinctrl_parse_groups(child, grp, info, i++);
  }
  
  return 0;
}


`````

继续跟进
```c

static int imx_pinctrl_parse_groups(struct device_node *np,
            struct imx_pin_group *grp,
            struct imx_pinctrl_soc_info *info,
            u32 index)
{
  int size, pin_size;
  const __be32 *list;
  int i;
  u32 config;

  dev_dbg(info->dev, "group(%d): %s\n", index, np->name);

  if (info->flags & SHARE_MUX_CONF_REG)
    pin_size = SHARE_FSL_PIN_SIZE;
  else
    pin_size = FSL_PIN_SIZE;
  /* Initialise group */
  grp->name = np->name;

  /*
   * the binding format is fsl,pins = <PIN_FUNC_ID CONFIG ...>,
   * do sanity check and calculate pins number
   */
  list = of_get_property(np, "fsl,pins", &size);
  if (!list) {
    dev_err(info->dev, "no fsl,pins property in node %s\n", np->full_name);
    return -EINVAL;
  }

  /* we do not check return since it's safe node passed down */
  if (!size || size % pin_size) {
    dev_err(info->dev, "Invalid fsl,pins property in node %s\n", np->full_name);
    return -EINVAL;
  }

  grp->npins = size / pin_size;
  grp->pins = devm_kzalloc(info->dev, grp->npins * sizeof(struct imx_pin),
        GFP_KERNEL);
  grp->pin_ids = devm_kzalloc(info->dev, grp->npins * sizeof(unsigned int),
        GFP_KERNEL);
  if (!grp->pins || ! grp->pin_ids)
    return -ENOMEM;

  for (i = 0; i < grp->npins; i++) {
    u32 mux_reg = be32_to_cpu(*list++);
    u32 conf_reg;
    unsigned int pin_id;
    struct imx_pin_reg *pin_reg;
    struct imx_pin *pin = &grp->pins[i];

    if (!(info->flags & ZERO_OFFSET_VALID) && !mux_reg)
      mux_reg = -1;

    if (info->flags & SHARE_MUX_CONF_REG) {
      conf_reg = mux_reg;
    } else {
      conf_reg = be32_to_cpu(*list++);
      if (!(info->flags & ZERO_OFFSET_VALID) && !conf_reg)
        conf_reg = -1;
    }

    //这里存储dts里面的内容
    pin_id = (mux_reg != -1) ? mux_reg / 4 : conf_reg / 4;
    pin_reg = &info->pin_regs[pin_id];
    pin->pin = pin_id;
    grp->pin_ids[i] = pin_id;
    pin_reg->mux_reg = mux_reg;
    pin_reg->conf_reg = conf_reg;
    pin->input_reg = be32_to_cpu(*list++);
    pin->mux_mode = be32_to_cpu(*list++);
    pin->input_val = be32_to_cpu(*list++);

    /* SION bit is in mux register */
    config = be32_to_cpu(*list++);
    if (config & IMX_PAD_SION)
      pin->mux_mode |= IOMUXC_CONFIG_SION;
    pin->config = config & ~IMX_PAD_SION;

    dev_dbg(info->dev, "%s: 0x%x 0x%08lx", info->pins[pin_id].name,
        pin->mux_mode, pin->config);
  }

  return 0;
}
`````
