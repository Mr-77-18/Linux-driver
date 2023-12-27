## ASOC框架分析
ASOC框架高能预警，ASOC比较庞大，所以我只能尽力分析。

首先要说明的是ASOC是在ALSA基础之上提出来的，解决了一些问题，这里不细讲这些问题，有兴趣百度就行。

首先要先列出一些数据结构，以防止在后面看代码的时候搞混了，因为确实容易搞混：

ALSA中的数据结构：
```c
snd_card
snd_device
snd_pcm
snd_pcm_substream
snd_pcm_runtime
snd_pcm_osp
`````
它们的关系如下：
<p align="center">
<img src="<++>">
</p>

ASOC中加上的数据结构(这个比较好分辨),因为这里面的数据结构都是带有"soc"这个字段的。
```c
snd_soc_card
snd_soc_pcm_runtime
snd_soc_dai_link
snd_soc_ops
snd_soc_codec
snd_soc_platform
snd_soc_codec_driver
snd_soc_platform_driver
`````

关系大概如下：
<p align="center">
<img src="<++>">
</p>

以上几张图片在网上也很多，但是很少把这两部分连接起来的，没有一个全局的视图。本文整理全局视图如下。
<p align="center">
<img src="<++>">
</p>

除了这些东西，我们还需要对一些常用的api进行总结：

1.	ALSA框架内
```c
snd_card_new()
	->snd_ctl_create()
snd_pcm_new()
snd_pcm_set_ops()
snd_pcm_set_ops()
snd_card_register()
`````

2. ASOC框架内
```c
snd_soc_instantiate_card()
snd_card_create()
soc_probe_dai_link()
	->soc_new_pcm()
		->snd_pcm_new()
`````
其次在开分析代码之前，还有一个我认为是最重要的，就是每一个函数都干了什么，网上很多都是直接用文字大概描述函数功能，但是我认为最好的方式是建立函数与图像之间的关系。即一个函数执行之后，会在整个框架中多一个什么结构体或者改变。这个是很多文章都没有的。

下面我们先建立这些关系，让脑子里面有一个针对整个ASOC框架的动态过程。

**snd_card_new():** 
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/g1.gif">
</p>

**snd_pcm_new():** 
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/g2.gif">
</p>

**snd_pcm_set_ops():** 
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/g3.gif">
</p>

**snd_card_register():** 
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/g4.gif">
</p>

---

然后就是驱动框架分析了:我们知道，ASOC驱动框架分为Machine , Platform , Codec三部分。其中Machine是关键，所以我们先从Machine开始分析

我们这里的分析实例是firefly的rk3399j开发板

dts部分

---

分析一下write的过程

---


