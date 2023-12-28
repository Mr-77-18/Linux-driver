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

---
