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
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/asoc1.png">
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
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/asoc2.png">
</p>

以上几张图片在网上也很多，但是很少把这两部分连接起来的，没有一个全局的视图。本文整理全局视图如下。
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/asoc3.png">
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
devm_snd_soc_register_card()
	->snd_soc_register_card()
	   ->snd_soc_instantiate_card()
				->snd_card_new()
				->soc_probe_link_dais()
					->soc_new_pcm()
						->snd_pcm_new()
						->snd_pcm_set_ops()
				->snd_card_register()
`````
其实从这里也可以看出来ASOC是对ALSA的进一步封装因为ALSA框架内的函数在这里都是被封装起来的。

其次在开分析代码之前，还有一个我认为是最重要的，就是每一个函数都干了什么，网上很多都是直接用文字大概描述函数功能，但是我认为最好的方式是建立函数与图像之间的关系。即一个函数执行之后，会在整个框架中多一个什么结构体或者改变。这个是很多文章都没有的。

下面我们先建立这些关系，让脑子里面有一个针对整个ASOC框架的动态过程。

1. ALSA框架内

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

2. ASOC框架内
最重要的函数莫过于devm_snd_soc_register_card()
<p align="center">
<img src="https://raw.githubusercontent.com/Mr-77-18/Don-t-want-to-learn/main/image/g5.gif">
</p>

有了对这些函数的感性认识，我们才不会在各种函数之中迷离。这些动态过程只需要有一个大体的印象，知道都是在干什么，在整个ASOC视图中处在那一块田地就够了。这在后面我们分析代码的时候会比较有帮助。

---
#### 代码分析
然后就是驱动框架分析了:我们知道，ASOC驱动框架分为Machine , Platform , Codec三部分。其中Machine是关键，所以我们先从Machine开始分析

我们这里的分析实例是firefly的rk3399j开发板

dts部分
```c
  rt5640-sound {
    compatible = "simple-audio-card";
    simple-audio-card,name = "rockchip,rt5640-codec";                                                                                                                                        
    simple-audio-card,format = "i2s";
    simple-audio-card,mclk-fs = <256>;
    simple-audio-card,widgets =
      "Microphone", "Mic Jack",
      "Headphone", "Headphone Jack";
    simple-audio-card,routing =
      "Mic Jack", "MICBIAS1",
      "IN1P", "Mic Jack",
      "Headphone Jack", "HPOL",
      "Headphone Jack", "HPOR";

    simple-audio-card,cpu {
      sound-dai = <&i2s1>;
    };  

    simple-audio-card,codec {
      sound-dai = <&rt5640>;
    };  
  };  

`````

rt5640-sound对应的就是ASOC中的machine部分，我们知道machine的作用就是用来连接platform和codec的，那么我们可以看到在rt5640-sound节点之中包含了两个dai:i2s1 , rt5640。没错，它们就是整个machine关联的platform和codec。我们这里不具体看plaftorm和codec，我们只研究machine。怎么研究呢，当然是根据compatible属性找到对应的处理驱动。可以找到该驱动在sound/soc/generic/simple-card.c.
```c
static struct platform_driver asoc_simple_card = {
	.driver = {
		.name = "asoc-simple-card",
		.of_match_table = asoc_simple_of_match,
	},
	.probe = asoc_simple_card_probe,
	.remove = asoc_simple_card_remove,
};

`````

当匹配成功之后，会调用.probe函数，我们具体看一下它的工作。
```c
static int asoc_simple_card_probe(struct platform_device *pdev)
{
  /*从设备树中取得想要的信息，主要就是dai_link,也就是整个machine关联着那些platfom和codec,这些信息会被保存在snd_soc_dai_link当中*/
  ...

	/*注册一个struct snd_soc_card。priv->snd_card->dai_link保存了dai_link*/
	ret = devm_snd_soc_register_card(&pdev->dev, &priv->snd_card);
	if (ret >= 0)
		return ret;

err:
  ...
}
`````

这里面最总的函数就是最后的devm_snd_soc_register_card()，我们已经在前面看见过整个函数的动态过程了，没错，它就是用来填满ASOC这副美丽的画卷的。
```c
int devm_snd_soc_register_card(struct device *dev, struct snd_soc_card *card)
{
	ptr = devres_alloc(devm_card_release, sizeof(*ptr), GFP_KERNEL);
	if (!ptr)
		return -ENOMEM;

	//注册设备
	ret = snd_soc_register_card(card);
	if (ret == 0) {
		*ptr = card;
		devres_add(dev, ptr);
	} else {
		devres_free(ptr);
	}

	return ret;
}
`````

这里面调用了关键函数snd_soc_register_card()
```c
int snd_soc_register_card(struct snd_soc_card *card)
{
	/*检查struct snd_soc_dai_link结构体，看是否符合要求*/
	for (i = 0; i < card->num_links; i++) {
    ...
	}

	/*初始化snd_soc_card结构体*/
	dev_set_drvdata(card->dev, card);

	snd_soc_initialize_card_lists(card);

	/*分配snd_soc_pcm_runtime结构体*/
	card->rtd = devm_kzalloc(card->dev,
				 sizeof(struct snd_soc_pcm_runtime) *
				 (card->num_links + card->num_aux_devs),
				 GFP_KERNEL);
	if (card->rtd == NULL)
		return -ENOMEM;

    /*对snd_soc_pcm_runtime中的一些成员做初始化,主要就是把保存在snd_soc_card->dai_link当中的关于plaform和codec的信息保存到snd_soc_pcm_runtime当中*/
  ...
  for (i = 0; i < card->num_links; i++) {
    ...
  }
  ...

	//这个是重点
	ret = snd_soc_instantiate_card(card);
	if (ret != 0)
		return ret;

  ...
}

`````

这里最后调用了关键函数snd_soc_instantiate_card()
```c
static int snd_soc_instantiate_card(struct snd_soc_card *card)
{
	/* bind DAIs */
	/*soc_bind_dai_link根据num_links的值，进行DAIs的bind工作*/
	for (i = 0; i < card->num_links; i++) {
		ret = soc_bind_dai_link(card, i);
    ...
	}

	/* bind aux_devs too */
	for (i = 0; i < card->num_aux_devs; i++) {
		ret = soc_bind_aux_dev(card, i);
    ...
	}

	/* initialize the register cache for each available codec */
	list_for_each_entry(codec, &codec_list, list) {
		ret = snd_soc_init_codec_cache(codec);
    ...
	}

	/* card bind complete so register a sound card */
	/*snd_card_new创建一个struct snd_card结构体*/	
	ret = snd_card_new(card->dev, SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1,
			card->owner, 0, &card->snd_card);
  ...

	/*创建声卡的一些其它部件，如dapm_widgets等*/
	if (card->dapm_widgets)
		snd_soc_dapm_new_controls(&card->dapm, card->dapm_widgets,
					  card->num_dapm_widgets);

	if (card->of_dapm_widgets)
		snd_soc_dapm_new_controls(&card->dapm, card->of_dapm_widgets,
					  card->num_of_dapm_widgets);

	/* initialise the sound card only once */
	/*调用各个子部件的probe函数.创建PCM、Control设备等、调用到platform->driver->pcm_new的函数*/
	if (card->probe) {
		ret = card->probe(card);
    ...
	}

	/* probe all components used by DAI links on this card */
	for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
			order++) {
		for (i = 0; i < card->num_links; i++) {
        ...
			  ret = soc_probe_link_components(card, i, order);
        ...
			}
		}
	}

	/* probe all DAI links on this card */
	for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
			order++) {
		for (i = 0; i < card->num_links; i++) {
        ...
			  ret = soc_probe_link_dais(card, i, order);
        ...
			}
		}
	}

	for (i = 0; i < card->num_aux_devs; i++) {
		ret = soc_probe_aux_dev(card, i);
    ...
		}
	}

	/* dapm和dai widget做相应操作 */
  ...

	/* snd_card_register */
	ret = snd_card_register(card->snd_card);

  ...

	return ret;
}

`````

整个函数做了很多重要工作，主要就是封装了ALSA框架内的函数，就是在ASOC这张画卷上画上了属于ASLA的那一部分(就是整个视图的左半部分)。
1. 首先是snd_card_new()
2. 然后是soc_probe_link_dais()
3. 最后是snd_card_register()

devm_snd_soc_register_card()->snd_soc_register_card()->snd_soc_instantiate_card()->snd_card_new()
```c
int snd_card_new(struct device *parent, int idx, const char *xid,
		    struct module *module, int extra_size,
		    struct snd_card **card_ret)
{
  ...
  /*分配snd_card结构体*/
	card = kzalloc(sizeof(*card) + extra_size, GFP_KERNEL);
  ...

  /*snd_card的一些初始化*/
  ...

  /*创建一个snd_device，具体的设备是snd_control*/
	err = snd_ctl_create(card);
	
  /*在/proc下创建对应的文件*/
	err = snd_info_card_create(card);

  ...
}

`````

devm_snd_soc_register_card()->snd_soc_register_card()->snd_soc_instantiate_card()->snd_probe_link_dais()
```c
static int soc_probe_link_dais(struct snd_soc_card *card, int num, int order)
{
	/* probe the CODEC DAI */
	for (i = 0; i < rtd->num_codecs; i++) {
		ret = soc_probe_dai(rtd->codec_dais[i], order);
		if (ret)
			return ret;
	}
  ...

	/* do machine specific initialization */
	if (dai_link->init) {
		ret = dai_link->init(rtd);
    ...
		}
	}
  ...

	if (cpu_dai->driver->compress_dai) {
		/*create compress_device"*/
		ret = soc_new_compress(rtd, num);
    ...
		}
	} else {

		if (!dai_link->params) {
			/* create the pcm */
			ret = soc_new_pcm(rtd, num);
      ...
			}
		} else {
      ...
		}
	}

	return 0;
}

`````

devm_snd_soc_register_card()->snd_soc_register_card()->snd_soc_instantiate_card()->snd_card_register()
```c
int snd_card_register(struct snd_card *card)
{
	if (!card->registered) {
    /*基础函数*/
		err = device_add(&card->card_dev);
	}

  /*调用snd_card下面所有的snd_device->ops.dev_register()*/
	if ((err = snd_device_register_all(card)) < 0)
		return err;

  ...
}
`````

devm_snd_soc_register_card()->snd_soc_register_card()->snd_soc_instantiate_card()->snd_probe_link_dais()->soc_new_pcm()
```c
int soc_new_pcm(struct snd_soc_pcm_runtime *rtd, int num)
{ 
  ...
	/* create the PCM */
  ...
	ret = snd_pcm_new(rtd->card->snd_card, new_name, num, playback,
			capture, &pcm);

  ...

  /*初始化pcm的一些内容*/
	if (rtd->dai_link->no_pcm) {
		if (playback)
			pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream->private_data = rtd;
		if (capture)
			pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream->private_data = rtd;
		goto out;
	}

	/* ASoC PCM operations */
	/*这里设置snd_soc_pcm_runtime中的ops  */
	if (rtd->dai_link->dynamic) {
		rtd->ops.open		= dpcm_fe_dai_open;
		rtd->ops.hw_params	= dpcm_fe_dai_hw_params;
		rtd->ops.prepare	= dpcm_fe_dai_prepare;
		rtd->ops.trigger	= dpcm_fe_dai_trigger;
		rtd->ops.hw_free	= dpcm_fe_dai_hw_free;
		rtd->ops.close		= dpcm_fe_dai_close;
		rtd->ops.pointer	= soc_pcm_pointer;
		rtd->ops.ioctl		= soc_pcm_ioctl;
	} else {
		rtd->ops.open		= soc_pcm_open;
		rtd->ops.hw_params	= soc_pcm_hw_params;
		rtd->ops.prepare	= soc_pcm_prepare;
		rtd->ops.trigger	= soc_pcm_trigger;
		rtd->ops.hw_free	= soc_pcm_hw_free;
		rtd->ops.close		= soc_pcm_close;
		rtd->ops.pointer	= soc_pcm_pointer;
		rtd->ops.ioctl		= soc_pcm_ioctl;
	}

	if (platform->driver->ops) {
		rtd->ops.ack		= platform->driver->ops->ack;
		rtd->ops.copy		= platform->driver->ops->copy;
		rtd->ops.silence	= platform->driver->ops->silence;
		rtd->ops.page		= platform->driver->ops->page;
		rtd->ops.mmap		= platform->driver->ops->mmap;
	}

	/*这里将snd_pcm.substream.ops设置成为snd_soc_pcm_runtime中ops  */
	if (playback)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, &rtd->ops);

	if (capture)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_CAPTURE, &rtd->ops);

	if (platform->driver->pcm_new) {
		/*实现dma内存申请和dma初始化相关工作*/
		ret = platform->driver->pcm_new(rtd);
		}
	}

  ...
}
`````

devm_snd_soc_register_card()->snd_soc_register_card()->snd_soc_instantiate_card()->snd_probe_link_dais()->soc_new_pcm()->snd_pcm_new()
```c
int snd_pcm_new(struct snd_card *card, const char *id, int device,
		int playback_count, int capture_count, struct snd_pcm **rpcm)
{
	return _snd_pcm_new(card, id, device, playback_count, capture_count,
			false, rpcm);
}
`````
```c
static int _snd_pcm_new(struct snd_card *card, const char *id, int device,
		int playback_count, int capture_count, bool internal,
		struct snd_pcm **rpcm)
{
	/*这里面创建了ops,最终会成为pcm里面的ops*/
	static struct snd_device_ops ops = {
		.dev_free = snd_pcm_dev_free,
		.dev_register =	snd_pcm_dev_register,
		.dev_disconnect = snd_pcm_dev_disconnect,
	};

  /*snd_pcm的一些初始化*/
	pcm->card = card;
	pcm->device = device;
	pcm->internal = internal;
	mutex_init(&pcm->open_mutex);
	init_waitqueue_head(&pcm->open_wait);
	INIT_LIST_HEAD(&pcm->list);

  /*创建snd_pcm的snd_pcm_substream,分为playback , capture*/
	if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_PLAYBACK, playback_count)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
	if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_CAPTURE, capture_count)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
  /*加入到snd_card当中去*/
	if ((err = snd_device_new(card, SNDRV_DEV_PCM, pcm, &ops)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
	if (rpcm)
		*rpcm = pcm;
	return 0;
}


`````

基本的核心函数就是这样，看各位能否跟前面的动图结合起来。

---

分析一下write的过程

首先我们要找到整个主入口，ASOC本质是字符设备驱动，主程序号是116，次设备号是随机分配的。应用层open(/dev/snd/pcmC1D0p)的时候其实是调用了chrdev_open(),看下面代码段就可以理解了。
```c
char_dev.c
/*
 * Called every time a character special file is opened
 * 应用层open每一个字符设备都会执行这个函数，也就是这个函数是所以字符设备的入口函数
 */
static int chrdev_open(struct inode *inode, struct file *filp)
{
fops = fops_get(p->ops);//获得file_operations *fops
replace_fops(filp, fops);//替换file_operations
ret = filp->f_op->open(inode, filp);//调用到具体节点对应的open函数。Asoc就是snd_open
}
 
sound.c:
static const struct file_operations snd_fops =
{//顶层 file_operations。实现转换作用
	.owner =	THIS_MODULE,
	.open =		snd_open,
	.llseek =	noop_llseek,
};
 
//alsa实际是个字符设备驱动 主设备号是#define CONFIG_SND_MAJOR	116	
static int __init alsa_sound_init(void) {
	if (register_chrdev(major, "alsa", &snd_fops)) {
		pr_err("ALSA core: unable to register native major device number %d\n", major);
		return -EIO;
	}
}
`````

没错，最终是调用了snd_open函数，这个函数干了什么呢，我们继续看。
```c
static int snd_open(struct inode *inode, struct file *file)
{
	/*没错，找到保存在snd_minors当中的snd_minor结构体，因为其中保存了争对这个snd_device的file_operations,这个file_operations在snd_pcm_dev_register()当中被设置*/
	mptr = snd_minors[minor];
	
	/*然后替换掉file->f_op,这样open传回来的描述符中的file中保存的就是这个f_ops,后续在调用write什么的时候都是调用到mptr->f_ops这里面的函数*/
	new_fops = fops_get(mptr->f_ops);
	replace_fops(file, new_fops);

	if (file->f_op->open)
		err = file->f_op->open(inode, file);
	return err;
}
`````

我们知道snd_soc_instantiate_card()中会调用snd_card_register()触发调用snd_device.ops->dev_register(),那么对于pcm来说，在snd_pcm_new()->_snd_pcm_new()中有：
```c
static int _snd_pcm_new(struct snd_card *card, const char *id, int device,
		int playback_count, int capture_count, bool internal,
		struct snd_pcm **rpcm)
{
	/*这里面创建了ops,最终会成为pcm里面的ops*/
	static struct snd_device_ops ops = {
		.dev_free = snd_pcm_dev_free,
		.dev_register =	snd_pcm_dev_register,
		.dev_disconnect = snd_pcm_dev_disconnect,
	};

  /*加入到snd_card当中去*/
	snd_device_new(card, SNDRV_DEV_PCM, pcm, &ops)
`````

可以看到，snd_device.ops->dev_register被赋值为snd_pcm_dev_register.我们看这个函数干了什么
```c
static int snd_pcm_dev_register(struct snd_device *device)
{
    ...
		/* register pcm */
		err = snd_register_device(devtype, pcm->card, pcm->device,
					  &snd_pcm_f_ops[cidx], pcm,
					  &pcm->streams[cidx].dev);
    ...
}

int snd_register_device(int type, struct snd_card *card, int dev,
			const struct file_operations *f_ops,
			void *private_data, struct device *device)
{
  ...
	preg->f_ops = f_ops;

  ...
	snd_minors[minor] = preg;
  ...
}

`````

没错，这里往snd_minors当中加入了snd_minor用来表示这个snd_device.其中的file_operation就是传进来的snd_pcm_f_ops[].
```c
const struct file_operations snd_pcm_f_ops[2] = {
	{
		.owner =		THIS_MODULE,
		.write =		snd_pcm_write,
		.write_iter =		snd_pcm_writev,
		.open =			snd_pcm_playback_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_playback_poll,
		.unlocked_ioctl =	snd_pcm_playback_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	},
	{
		.owner =		THIS_MODULE,
		.read =			snd_pcm_read,
		.read_iter =		snd_pcm_readv,
		.open =			snd_pcm_capture_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_capture_poll,
		.unlocked_ioctl =	snd_pcm_capture_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	}
}
`````

根据是playback还是capture选择合适的file_operations.好的，那么到这里我们已经通过open(dev/snd/pcmXXX)找到了最终的入口操作函数表file_operations。所以在调用write()的时候最终调用的是snd_pcm_f_ops[0].write();

```c
static ssize_t snd_pcm_write(struct file *file, const char __user *buf,
			     size_t count, loff_t * offset)
{
  ...
	result = snd_pcm_lib_write(substream, buf, count);
  ...
}
`````

```c
snd_pcm_sframes_t snd_pcm_lib_write(struct snd_pcm_substream *substream, const void __user *buf, snd_pcm_uframes_t size)
{
  ...
	return snd_pcm_lib_write1(substream, (unsigned long)buf, size, nonblock,
				  snd_pcm_lib_write_transfer);
}
`````
```c
static snd_pcm_sframes_t snd_pcm_lib_write1(struct snd_pcm_substream *substream, 
					    unsigned long data,
					    snd_pcm_uframes_t size,
					    int nonblock,
					    transfer_f transfer)
{
      ...
			err = snd_pcm_start(substream);
      ...
}
`````

```c
static struct action_ops snd_pcm_action_start = {
	.pre_action = snd_pcm_pre_start,
	.do_action = snd_pcm_do_start,
	.undo_action = snd_pcm_undo_start,
	.post_action = snd_pcm_post_start
};

int snd_pcm_start(struct snd_pcm_substream *substream)
{
	return snd_pcm_action(&snd_pcm_action_start, substream,
			      SNDRV_PCM_STATE_RUNNING);
}
`````

```c
static int snd_pcm_action(struct action_ops *ops,
			  struct snd_pcm_substream *substream,
			  int state)
{
    ...
		return snd_pcm_action_single(ops, substream, state);
    ...
}
`````

```c
static int snd_pcm_action_single(struct action_ops *ops,
				 struct snd_pcm_substream *substream,
				 int state)
{
	int res;
	
	res = ops->pre_action(substream, state);
	if (res < 0)
		return res;
	res = ops->do_action(substream, state);
	if (res == 0)
		ops->post_action(substream, state);
	else if (ops->undo_action)
		ops->undo_action(substream, state);
	return res;
}
`````
这里面的action_ops是在snd_pcm_action()当中的第一个参数传过来的，所以调用的ops->do_action是snd_pcm_do_start()
```c
static int snd_pcm_do_start(struct snd_pcm_substream *substream, int state)
{
	if (substream->runtime->trigger_master != substream)
		return 0;
	return substream->ops->trigger(substream, SNDRV_PCM_TRIGGER_START);
}

`````
没错，这里调用了substream->ops->trigger(),如果substream是playback来说，对应的是什么，在哪里设置了这个substream->ops你知道吗。答案是snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, &rtd->ops)。这个在soc_pcm_new当中被调用。没错这个ops就是struct snd_soc_pcm_runtime当中的ops
```c
int soc_new_pcm(struct snd_soc_pcm_runtime *rtd, int num)
{ 
	/* ASoC PCM operations */
	/*这里设置snd_soc_pcm_runtime中的ops  */
	  rtd->ops.open		= soc_pcm_open;
		rtd->ops.hw_params	= soc_pcm_hw_params;
		rtd->ops.prepare	= soc_pcm_prepare;
		rtd->ops.trigger	= soc_pcm_trigger;
		rtd->ops.hw_free	= soc_pcm_hw_free;
		rtd->ops.close		= soc_pcm_close;
		rtd->ops.pointer	= soc_pcm_pointer;
		rtd->ops.ioctl		= soc_pcm_ioctl;
	/*这里将snd_pcm.substream.ops设置成为snd_soc_pcm_runtime中ops  */
	if (playback)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, &rtd->ops);

	if (capture)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_CAPTURE, &rtd->ops);
}
`````

没错，对应到的是soc_pcm_trigger()
```c
static int soc_pcm_trigger(struct snd_pcm_substream *substream, int cmd)
{
	for (i = 0; i < rtd->num_codecs; i++) {
			ret = codec_dai->driver->ops->trigger(substream,
							      cmd, codec_dai);
	}

	if (platform->driver->ops && platform->driver->ops->trigger) {
		ret = platform->driver->ops->trigger(substream, cmd);
	}

	if (cpu_dai->driver->ops && cpu_dai->driver->ops->trigger) {
		ret = cpu_dai->driver->ops->trigger(substream, cmd, cpu_dai);
	}

	if (rtd->dai_link->ops && rtd->dai_link->ops->trigger) {
		ret = rtd->dai_link->ops->trigger(substream, cmd);
	}
}


`````

没错，这里面最终回调platform和codec对应的操作函数。看到这里，你应该会感到兴奋。因为你似乎有了那么一点头绪，不回觉得ASOC是一个庞然大物，至少你在这个迷茫的世界里面找到了一条清晰的道理（write的过程).依靠这条路继续探索ASOC吧，加油年轻人
