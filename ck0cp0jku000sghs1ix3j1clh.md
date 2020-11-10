## Getting Started with Nvidia Jetson Nano, TensorFlow and Smalltalk

On all my previous posts (like [this one](https://dev.to/martinezpeck/docker-swarm-cloud-on-a-arm64-diy-sbc-cluster-running-a-smalltalk-webapp-9l1)) you can see [VASmalltalk](https://www.instantiations.com/products/vasmalltalk/) running on any Raspberry Pi, on Rock64 and even on Nvidia Jetson TX2:

%[https://twitter.com/martinezpeck/status/1141775466384306182]

In addition, you can also see [previous posts](https://dev.to/martinezpeck/object-detection-with-tensorflow-and-smalltalk-15p7) where I show how to use TensorFlow from Smalltalk to recognize objects in images.

Last week, at [ESUG 2019](https://esug.github.io/2019-Conference/conf2019.html), I demoed a VA Smalltalk and TensorFlow project on an Nvidia Jetson Nano provided by [Instantiations](https://twitter.com/instantiations).

In this post, I will show you how to get started with the Jetson Nano, how to run VASmalltalk and finally how to use the TensorFlow wrapper to take advantage of the 128 GPU cores.

### What do you need before starting

Below is the whole list of supplies I gathered:

- [NVIDIA Jetson Nano Developer Kit](https://www.amazon.com/NVIDIA-Jetson-Nano-Developer-Kit/dp/B07PZHBDKT/ref=sr_1_3?crid=3TX2GTYZE1PQO&keywords=jetson+nano&qid=1567444057&s=gateway&sprefix=jetson+nano%2Caps%2C347&sr=8-3)
- MicroSD card: I got a [Samsung 128GB U3](https://www.amazon.com/Samsung-MicroSDXC-Adapter-MB-ME128GA-AM/dp/B06XWZWYVP/ref=pd_bxgy_147_img_3/144-0517921-1924023?_encoding=UTF8&pd_rd_i=B06XWZWYVP&pd_rd_r=3351843b-8299-4a77-bbb3-d1dd44b03579&pd_rd_w=2dh8W&pd_rd_wg=eLEPI&pf_rd_p=a2006322-0bc0-4db9-a08e-d168c18ce6f0&pf_rd_r=KFRNVE50K798SYHVYKX3&psc=1&refRID=KFRNVE50K798SYHVYKX3)
- Power supply: you can choose either a limited USB or a much more powerful DC switching power supply. I got [the latter](https://www.amazon.com/SMAKN-Switching-Supply-Adapter-100-240/dp/B01N4HYWAM/ref=pd_bxgy_147_img_2/144-0517921-1924023?_encoding=UTF8&pd_rd_i=B01N4HYWAM&pd_rd_r=3351843b-8299-4a77-bbb3-d1dd44b03579&pd_rd_w=2dh8W&pd_rd_wg=eLEPI&pf_rd_p=a2006322-0bc0-4db9-a08e-d168c18ce6f0&pf_rd_r=KFRNVE50K798SYHVYKX3&psc=1&refRID=KFRNVE50K798SYHVYKX3). 
- Case (optional): I like metal cases so I got [one](https://www.amazon.com/Geekworm-NVIDIA-Enclosure-Control-Developer/dp/B07RRRX121/ref=sr_1_4?keywords=jetson+nano+case&qid=1565123115&s=electronics&sr=1-4) with Power & Reset Control Switch.
- Fan (optional): I could only find [one fan that would fit on the Nano](https://www.amazon.com/Noctua-NF-A4x20-5V-PWM-Premium-Quality/dp/B071FNHVXN/ref=sr_1_3?keywords=noctua+nfa4x20+5v+pwm&qid=1567445024&s=gateway&sr=8-3) but it was very expensive. 
- Wireless Module (optional): the board does not come with built-in WiFi or Bluetooth so I decided to buy [this module](https://www.amazon.com/Waveshare-AC8265-Wireless-Supports-Bluetooth/dp/B07SGDRG34/ref=sr_1_2_sspa?keywords=jetson+nano+wifi&qid=1565122735&s=electronics&sr=1-2-spons&psc=1&spLa=ZW5jcnlwdGVkUXVhbGlmaWVyPUEzRDhMM1hZQURCMDdVJmVuY3J5cHRlZElkPUEwNTAyNjMzMk1FWU5aNEhVNkJKNSZlbmNyeXB0ZWRBZElkPUEwMzAyMDcxM0JIS1VGSE43TTFPVyZ3aWRnZXROYW1lPXNwX2F0ZiZhY3Rpb249Y2xpY2tSZWRpcmVjdCZkb05vdExvZ0NsaWNrPXRydWU=). 

### Assembling the Nano and related hardware

For this step, I first followed [this short guide](https://towardsdatascience.com/getting-started-with-nvidia-jetson-nano-and-installing-tensorflow-gpu-ad4a3da8ed26) but finally moved to [this one which was super detailed](https://blog.hackster.io/getting-started-with-the-nvidia-jetson-nano-developer-kit-43aa7c298797). I won’t repeat everything written there but instead I will add my own bits below.

I started by formatting the SD. For this, I always use “SD Card Formatter” program. Downloading the operating system image and flashing the SD was easy… But the first downside is that for the first boot you NEED an external monitor, keyboard and mouse. No way to do it headless :( After the first boot, you can indeed enable SSH and VNC, but not for the first time.

The next step was to assemble the Wifi and Bluetooth. It was not a walk in the park but not that difficult either. You need to disassemble the Nano a bit, connect some wires, etc:

%[https://twitter.com/martinezpeck/status/1165271816224563200]

Something bad is that the board is configured to start by default with a USB power supply. In my case, I ordered a DC instead as it’s better if you want to take the maximum power. But…to tell the Nano whether to use USB or DC, you must change the jumper J48. But guess what? The development kit (100USD) does NOT even bring a single jumper. So you got your Nano and your DC power supply and you are dead, you can’t boot it. Seriously? (BTW, before you ask, no, I didn’t have a female-to-female cable with me that day as a workaround for the jumper)

The other complicated part was to assemble the case and the fan. For that, I needed to [carefully watch this video](https://www.youtube.com/watch?v=SpUB6h4Akp4&t=365s) a few times. Once it was built, it felt really solid and nice. BTW the case did come with a jumper for J48 which was really nice since that meant I could use the DC power supply.

The fan itself was also complicated. The Noctua NF-A4x20 5V PWM I bought wouldn’t fit easily. The NA-AV3 silicone anti-vibration mounts would not get through the holes of the Nano. And the screws for the fan provided by the case were too short. So I had to buy some other extra screws that were long enough.

When I was ready to try the fan, I powered it and nothing happened. I thought I did something wrong and I had to re-open the case a few times… painful process. I almost gave up, when I found [some help over the internet](https://devtalk.nvidia.com/default/topic/1049589/jetson-nano/fan-not-working/). Believe it or not, you must run a console command in order to start the fan: `sudo jetson_clocks`. After that, it started working.

%[https://twitter.com/martinezpeck/status/1168873407116644352]

### Setting it to run headless

While in most boards and operating systems this is easy, on the Nano this is a challenging part. The SSH part is easy and you almost don’t need to do anything in particular. But for VNC… OMG…. I followed all the recommendations provided in [this guide](https://blog.hackster.io/getting-started-with-the-nvidia-jetson-nano-developer-kit-43aa7c298797). In my case, I could never get the `xrdp` working… when it tries to connect from my Mac, it simply crashes…

As for VNC, after all the workarounds/corrections mentioned there, I was able to connect but the resolution was too bad (640×480). I spent quite some time googling until I found a workaround mentioned [here](https://devtalk.nvidia.com/default/topic/995621/jetson-tx1/jetson-tx1-desktop-sharing-resolution-problem-without-real-monitor/1). Basically, I did `sudo vim /etc/X11/xorg.conf` and I added these lines:

```
Section "Screen"
   Identifier    "Default Screen"
   Monitor        "Configured Monitor"
   Device        "Default Device"
   SubSection "Display"
       Depth    24
       Virtual 1280 800
   EndSubSection
EndSection
```

 In other words, I needed to change the size of the Virtual Display that is used if no monitor is connected (by default it was 640×480).

After rebooting, I was finally able to get a decent resolution with VNC.

- ![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-02-at-4.16.38-PM.png?fit=748%2C468&ssl=1)

### Installing VASmalltalk dependencies

This part was easy and I basically followed the bash script of a [previous post](https://dev.to/martinezpeck/getting-started-with-vasmalltalk-raspberry-pi-and-other-devices-25g3-temp-slug-2060664):

```bash
# Install VA Dependencies for running headfull and VA Environments tool
sudo apt-get install --assume-yes --no-install-recommends \
  libc6 \
  locales \
  xterm \
  libxm4 \
  xfonts-base \
  xfonts-75dpi \
  xfonts-100dpi
  
# Only necessary if we are using OpenSSL from Smalltalk
sudo apt-get install --assume-yes --no-install-recommends \
  libssl-dev 
  
# Generate locales
sudo su
echo en_US.ISO-8859-1 ISO-8859-1 >> /etc/locale.gen
echo en_US.ISO-8859-15 ISO-8859-15 >> /etc/locale.gen
locale-gen
exit
```

### Installing TensorFlow and VASmalltalk wrapper

The first thing you must do is to either build TensorFlow from scratch for Nvidia Jetson Nano with CUDA support or try to get a pre-build binary from somewhere.  I am getting the latter using the following bash script:

```bash
mkdir tensorflow
cd tensorflow
wget https://dl.photoprism.org/tensorflow/nvidia-jetson/libtensorflow-nvidia-jetson-nano-1.14.0.tar.gz
tar xvzf libtensorflow-nvidia-jetson-nano-1.14.0.tar.gz
cd lib
ln -s libtensorflow_framework.so libtensorflow_framework.so.1
```

The symbolic link is a workaround because in the shared libraries that I downloaded, `libtensorflow.so` would depend on `libtensorflow_framework.so.1` but the library that was shipped was `libtensorflow_framework.so` and so I made a symlink.

To install VASmalltalk and the TensorFlow wrapper, I followed [the instructions from the Github repository](https://github.com/vasmalltalk/tensorflow-vast/blob/master/README.md#installation). The only detail is that ARM64 VM will be shipped in the upcoming 9.2 ECAP 3….so send me a private message and I will send it to you until the release is public.

For the `.ini` file I added:

```ini
TENSORFLOW_LIB=/home/mpeck/Instantiations/tensorflow/lib/libtensorflow.so
```

The last bit is that TensorFlow needs help so that `libtensorflow` can find `libtensorflow_framework`. So what I did is to export `LD_LIBRARY_PATH` before starting the VASmalltalk image. Another possibility is moving the shared libraries to `/usr/lib` or `/usr/local/lib`. It’s up to you.

```bash
cd ~/Instantiations/VastEcap3_b437_7b7fc914f16f_linux/raspberryPi64/
export LD_LIBRARY_PATH=/home/mariano/Instantiations/libtensorflow-nvidia-jetson-nano-1.14.0.2/lib:$LD_LIBRARY_PATH
./abt64.sh
```

And all tests were green:

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-04-at-11.45.48-AM.png?fit=748%2C459&ssl=1)

### Confirming we are using GPU

By default, if a GPU is present (and the shared library was compiled with GPU support), TensorFlow will use GPU over CPU. From Smalltalk we can confirm this by checking the available `TFDevice` by inspecting the result of `(TFSession on: TFGraph create) devices`

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-04-at-11.54.56-AM.png?w=748&ssl=1)

You can then run a simple test like `TensorFlowCAPITest >> testAddControlInput` and see the log printed into the `xterm`. You should see that a GPU device is being used:

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-04-at-12.07.47-PM.png?w=748&ssl=1)

### **Using 128 GPU cores, TensorFlow and VASmalltalk to detect Kölsch beers with #esug19 pictures**

OK. So we have TensorFlow running, all our tests passing and we are sure we are using GPU. The obvious next step is to run some real-world demo.

In a [previous post](https://dev.to/martinezpeck/object-detection-with-tensorflow-and-smalltalk-15p7) you saw some examples of Object Detection. During the ESUG 2019 Conference I wanted to show this demo but instead of recognizing random objects on random images, I showed how to detect “beers” (Kölsch! we were at Cologne, Germany!) on the real pictures people uploaded to Twitter #esug19 hashtag.

%[https://twitter.com/martinezpeck/status/1167438196038348801]

The code for that was fairly easy:

```smalltalk
ObjectDetectionZoo new
    imageFiles: OrderedCollection new;
    addImageFile: '/home/mariano/Instantiations/tensorflow/esug2019/beer1.png';
    graphFile: '/home/mariano/Instantiations/tensorflow/frozen_inference_graph-faster_resnet50.pb';
    labelsFile: 'examples/objectDetectionZoo/mscoco_label_map.pbtxt';
    prepareImageInput;
    prepareSession;
    predict;
    openPictureWithBoundingBoxesAndLabel
```

And here are the results:

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-04-at-2.52.37-PM.png?fit=748%2C690&ssl=1)

### Conclusions

The Nvidia Jetson Nano does offer good hardware at a reasonable price. However, it’s much harder to setup than most of the boards out there. It’s the first board that takes me sooooo much time to get fully working.

But the worst part, in my opinion, is the state of Linux Tegra. Crashes everywhere and almost impossible to setup something as simple as VNC. I would really like to see a better/newer OS for the Nano.

Once all your painful setup is done, it works well and it provides nice GPU capabilities. We now have everything in place to start experimenting with it.

PS: Thanks Maxi Tabacman and Gera Richarte for doing a review of this post!