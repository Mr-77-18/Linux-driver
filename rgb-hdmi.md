## RGB转HDMI实验
物理连接：
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/wuli.png">
</p>

---

修改dts：
1.	LCD配置
2. sii902x配置

---

1. LCD配置
在根节点"/"下加入如下节点
```c
&lcdif {
  pinctrl-names = "default";
  pinctrl-0 = <&pinctrl_lcdif_dat
         &pinctrl_lcdif_ctrl>;
  display = <&display0>;
  status = "okay";

        display0: display {
                bits-per-pixel = <16>;
                bus-width = <24>;

                display-timings {
                        native-mode = <&timing0>;
                        timing0: timing0 {
                        clock-frequency = <51200000>;
                        hactive = <1024>;
                        vactive = <600>;
                        hfront-porch = <160>;
                        hback-porch = <140>;
                        hsync-len = <20>;
                        vback-porch = <20>;
                        vfront-porch = <12>;
                        vsync-len = <3>;

                        hsync-active = <0>;
                        vsync-active = <0>;
                        de-active = <1>;
      /* rgb to hdmi: pixelclk-ative should be set to 1 */
                        pixelclk-active = <1>;
                        };
                };
        };
};
`````
可以看出io复用一共有两个：pinctrl_lcdif_dat , pinctrl_lcdif_ctrl。在iomuxc节点下加入这两个节点的描述：
```c
  pinctrl_lcdif_dat: lcdifdatgrp {
      fsl,pins = <
        MX6UL_PAD_LCD_DATA00__LCDIF_DATA00  0x49
        MX6UL_PAD_LCD_DATA01__LCDIF_DATA01  0x49
        MX6UL_PAD_LCD_DATA02__LCDIF_DATA02  0x49
        MX6UL_PAD_LCD_DATA03__LCDIF_DATA03  0x49
        MX6UL_PAD_LCD_DATA04__LCDIF_DATA04  0x49                                                                                                                                                                  
        MX6UL_PAD_LCD_DATA05__LCDIF_DATA05  0x49
        MX6UL_PAD_LCD_DATA06__LCDIF_DATA06  0x49
        MX6UL_PAD_LCD_DATA07__LCDIF_DATA07  0x51
        MX6UL_PAD_LCD_DATA08__LCDIF_DATA08  0x49
        MX6UL_PAD_LCD_DATA09__LCDIF_DATA09  0x49
        MX6UL_PAD_LCD_DATA10__LCDIF_DATA10  0x49
        MX6UL_PAD_LCD_DATA11__LCDIF_DATA11  0x49
        MX6UL_PAD_LCD_DATA12__LCDIF_DATA12  0x49
        MX6UL_PAD_LCD_DATA13__LCDIF_DATA13  0x49
        MX6UL_PAD_LCD_DATA14__LCDIF_DATA14  0x49
        MX6UL_PAD_LCD_DATA15__LCDIF_DATA15  0x51
        MX6UL_PAD_LCD_DATA16__LCDIF_DATA16  0x49
        MX6UL_PAD_LCD_DATA17__LCDIF_DATA17  0x49
        MX6UL_PAD_LCD_DATA18__LCDIF_DATA18  0x49
        MX6UL_PAD_LCD_DATA19__LCDIF_DATA19  0x49
        MX6UL_PAD_LCD_DATA20__LCDIF_DATA20  0x49
        MX6UL_PAD_LCD_DATA21__LCDIF_DATA21  0x49
        MX6UL_PAD_LCD_DATA22__LCDIF_DATA22  0x49
        MX6UL_PAD_LCD_DATA23__LCDIF_DATA23  0x51
      >;
    };

  pinctrl_lcdif_ctrl: lcdifctrlgrp {
      fsl,pins = <
        MX6UL_PAD_LCD_CLK__LCDIF_CLK      0x49
        MX6UL_PAD_LCD_ENABLE__LCDIF_ENABLE  0x49
        MX6UL_PAD_LCD_HSYNC__LCDIF_HSYNC    0x49
        MX6UL_PAD_LCD_VSYNC__LCDIF_VSYNC    0x49
      >;
    };
`````
到这里关于lcd的配置就配置完成了。

---

2.sii902x配置
sii902x是在板子上属于i2c设备，挂在了开发板的i2c2上，所以在i2c2节点下增加节点如下：
```c
  //Senhong Liu:RGB->HDMI
  sii902x: sii902x@39 {
    compatible = "SiI,sii902x";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_sii902x
                &ts_reset_hdmi_pin>;
    interrupt-parent = <&gpio1>;
    interrupts = <9 IRQ_TYPE_EDGE_FALLING>;
    irq-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
    mode_str = "1280x720M@60";
    bits-per-pixel = <16>;
    resets = <&sii902x_reset>;
    reg = <0x39>;
    status = "okay";
  };  

`````

然后在iomuxc节点下面加入如下节点：
```c
   /* Senhong Liu SII902X  INT*/
    pinctrl_sii902x: hdmigrp-1 {
      fsl,pins = <
        MX6UL_PAD_GPIO1_IO09__GPIO1_IO09 0x11
      >;
    };
`````

还要在iomuxc_SNVS中加入如下节点：
```c
    ts_reset_hdmi_pin: ts_reset_hdmi_mux {
      fsl,pins = <
        MX6ULL_PAD_SNVS_TAMPER9__GPIO5_IO09 0x49
      >;
    };
`````

最后还需要在根节点”\"下加入如下节点：
```c
  //Senhong Liu:RGB->HDMI
  sii902x_reset: sii902x-reset {
    compatible = "gpio-reset";
    reset-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;
    reset-delay-us = <100000>;
    #reset-cells = <0>;
    status = "okay";
  };
`````

至此所有dts的修改完成

**要注意看io复用的内容是否有冲突，需要把有冲突的部分注释掉** 

---

最后就是加入驱动部分了，lcd部分驱动linux内核默认自带的。sii902x的驱动需要自己加上去。

打开.config文件，修改如下内容：
```
 ++ CONFIG_FB_MXS_SII902X=y
`````

至此修改完成。
