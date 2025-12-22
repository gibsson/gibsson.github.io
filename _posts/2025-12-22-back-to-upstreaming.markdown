---
layout: post
title:  "Back to upstreaming!"
date:   2025-12-22 22:02:14 +0100
categories: mainline
---
For the last few years, I haven't had the chance to do much upstreaming as part of my job.

It took some time to get back on the horse but I wanted to share my thoughts on the process and how it involved since I last did it.

# What to upstream?

The first step was to decide what to upstream. As my current work is around Qualcomm Android BSP, there isn't much to upstream there unfortunately.

So I went back to the hardware I had in stock and picked the [Tungsten510](https://www.ezurio.com/product/tungsten510-smarc) and [Tungsten700](https://www.ezurio.com/product/tungsten700-smarc) SMARC modules which are based on the [MediaTek Genio processors](https://www.ezurio.com/product/tungsten510-smarc).

<p align="center">
<img src="{{ site.baseurl }}/img/tungsten510.png" width="400">
</p>

The idea was to get a powerful platform that would allow testing many aspects of the kernel. That is what the Genio SoCs offer as they are high-end SoC (up to 8 cores, Mali GPU, USB3.0, GbE, PCIe, MIPI-DSI, HDMI etc...) and are well supported upstream thanks to the [work from Collabora](https://www.collabora.com/news-and-blog/news-and-events/collabora-mediatek-pushing-boundaries-on-the-latest-iot-boards-and-chromebooks.html).

# Upstreaming process

Once the device was selected, the next step was to get the board up and running with mainline Linux.

In this case it was fairly easy as I did a port of the platform for the [Android (RITA) release from Baylibre](https://lairdcp.github.io/guides/android-14-rita-tungsten/Android-rita14-release-for-Tungsten-Platforms.html) which was based on Android kernel 6.12 + Collbora upstream patches. This blog post won't be about the porting effort but the [mediatek-next](https://github.com/gibsson/linux-next/commits/mediatek-next/) branch on github shows my progress.

## First submission

Now that the board is booting on `mediatek-next` branch, it was rebased on top of `master` which was `v6.18-rc1` at the time.

As my previous device tree contribution was in 2018, I was rusty and not necessarily aware of process changes. Therefore I went ahead with my (old) way:
```
$ git format-patch --cover-letter -o patches_v1
$ ./scripts/get_maintainer.pl patches_v1/000*
$ vim patches_v1/000*
```

*Spoiler*: don't use any of the above commands as they are deprecated/outdated :sweat_smile:

First feedback I received after sending the series was: *"Just use b4!"*.

## b4 tool

What is `b4`? It's a tool created to make both developers and maintainers lives easier. And now that I've used it I confirm that it is true to its promise. Moreover the documentation is awesome:
- [b4.docs.kernel.org](https://b4.docs.kernel.org/)

Note that the `b4` binary that comes with your OS might be outdated, so I recommend using `pip` to get the latest:
```
$ pip install b4
```

But what does it do really? Anything needed for a good contribution:
- Keeps track of your changes
- Helps running `checkpatch` on pathces: `b4 prep --check`
- Takes care of CC'ing the right persons: `b4 prep --auto-to-cc`
- Ease the writing of your cover letter: `b4 prep --edit-cover`
- Retrieves the review tags from mailing list: `b4 trailers -u`

Once all of the above options have been used to ensure the series looks good, you can simply send (or even do a dry-run):
```
$ b4 send
```

At this stage, the whole patch management process was ok, now to the next tool.

## DTS schema validation

That one I knew about after reading [Tips and Tricks for Validating Devicetree sources](https://www.linaro.org/blog/tips-and-tricks-for-validating-devicetree-sources-with-the-devicetree-schema/) from Linaro when the article came out. But that doesn't mean it was done properly at first :wink:

The article is well written and explains everything so I won't cover the basics. But I got tricked by forcing the use of my distro dt-schema package (2022.08) whereas the kernel checks for version > 2023.9. Long story short, bypassing that check hides so many errors.

Fortunately, the bot reporting the issues does give a clue:
```
If you already ran DT checks and didn't see these error(s), then
make sure dt-schema is up to date:
  pip3 install dtschema --upgrade
```

After the upgrade I was able to replicate the issues reported by the bot and fix them. To be noted that the errors are very useful, it checks the bindings, the node names and so on, this tool will now be in my todo list for each device tree made.

Here is what it looks like on the latest version of my patch:
```
 make CHECK_DTBS=y mediatek/mt8370-tungsten-smarc.dtb
  DTC [C] arch/arm64/boot/dts/mediatek/mt8370-tungsten-smarc.dtb
/linux/arch/arm64/boot/dts/mediatek/mt8370-tungsten-smarc.dtb: pmic (mediatek,mt6359): '#sound-dai-cells' does not match any of the regexes: '^pinctrl-[0-9]+$'
	from schema $id: http://devicetree.org/schemas/mfd/mediatek,mt6397.yaml#
/linux/arch/arm64/boot/dts/mediatek/mt8370-tungsten-smarc.dtb: scp@10720000 (mediatek,mt8188-scp-dual): reg-names: ['cfg'] is too short
	from schema $id: http://devicetree.org/schemas/remoteproc/mtk,scp.yaml#
/linux/arch/arm64/boot/dts/mediatek/mt8370-tungsten-smarc.dtb: scp@10720000 (mediatek,mt8188-scp-dual): reg: [[0, 275906560, 0, 917504]] is too short
	from schema $id: http://devicetree.org/schemas/remoteproc/mtk,scp.yaml#
/linux/arch/arm64/boot/dts/mediatek/mt8370-tungsten-smarc.dtb: scp@10720000 (mediatek,mt8188-scp-dual): reg-names: ['cfg'] is too short
	from schema $id: http://devicetree.org/schemas/remoteproc/mtk,scp.yaml#
```

But wait, there are still errors!?! True, but those are from `mt8188.dtsi` and also present in other platforms, to be fixed separately.

# Result

After learning to use the above tools and taking all the other feedback into account I'm happy with latest submission:
- [[PATCH v5 0/5] Add support for Ezurio MediaTek platforms](https://lore.kernel.org/all/20251203-review-v5-0-b26d5512c6af@gmail.com/)

As of this writing, the series hasn't been merged but all the corrections requested have been made so I'm hopeful it will get in.

Until then, I am using this platform to experiment with the latest kernels as most features are working:
- MIPI-DSI display support
- GPU Panfrost open-souce driver
- SDIO: eMMC, SD & Wi-Fi/BT (MT7921S)
- Gigabit Ethernet
- USB 3.0 ports
- PCIe gen3

![kmscube]({{ site.baseurl }}/img/tungsten510_kmscube.png)

Next blog post will be showing more in-depth testing/benchmarking.

Also, updates will be provided from the missing features will be supported:
- HDMI
- I2S Audio
- Video codecs (h.264/h.265)
- Camera support

Let me know if you have any questions :wink:
