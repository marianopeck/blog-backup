## Challenge Accepted: Build TensorFlow C Binding for Raspberry Pi in 2019

Believe me. Setting up the environment and building TensorFlow C binding for Raspberry Pi is more complicated than training a neural network that makes me rich by robo-trading assets.

### Motivation

As SBCs (Single Board Computer) get more and more powerful and cheap, the more likely we will want to run some more heavy computation on them. People like to use terms like “Edge Computing”, “Embedded HPC or ML” or similar terms.

Something quite common between all these different SBCs alternatives is the use of ARM processors plus some type of GPU.

A classical example of this heavy computation is AI (Artificial Intelligence) and ML (Machine Learning). In this area, one of the most used and accepted library is Google’s [TensorFlow](https://www.tensorflow.org/). Such library is written in Python. However, [there are also pre-build official binaries for C, Java and Go](https://www.tensorflow.org/install).

[The C API is commonly used for binding to other languages via FFI](https://www.tensorflow.org/guide/extend/bindings) (Foreign Function Interface). From my point of view, that’s a critical binary.

Currently, I am developing/testing a [VASmalltalk binding that uses the C library via FFI](https://github.com/vasmalltalk/tensorflow-vast). I tested on Linux x64, I tested on Windows and then I wanted to try in Raspberry Pi 3B+, Pine64, Nvidia Jetson Nano, etc… Why? Because I truly believe that this “embedded ML” (or whatever you call it) has value. Running machine learning algorithms in a 35USD machine seems interesting to me.

%[https://twitter.com/martinezpeck/status/1149286273661710336]

So…what happened? I simply [went to TensorFlow’s official website](https://www.tensorflow.org/install/lang_c#download) and look for the shared library. Guess what? There was none. Zero. Null. Nil. No binaries for any kind of ARM board. I was so surprised that [I asked in StackOverflow](https://stackoverflow.com/questions/56837317/how-can-i-get-a-tensorflow-c-binding-for-raspberry-pi).

I understand that there are plenty of boards out there each with different hardwares, softwares, drivers, operating systems, etc. But I was expecting at least to have it for some very common ones like Raspberry Pi and Nvidia Jetson Nano.

Anyway…this is how my journey started. I am not sure if my writings would be useful for others, but at least for my future me, I am sure they will.

The next sections are sorted in the order I look for the solutions. 

> DISCLAIMER: I am NOT a TensorFlow expert. So if you have any feedback, please share!

### Failed attempt 1: install Python version and extract the shared library from there

With some recent TensorFlow version, Raspberry Pi / Raspbian is officially supported (I think \>= 1.9). However, the only “binaries” available are Python wheels. I suspected Python would be using C underneath so I install the Python version directly on my Pi following [the official instructions using `pip`](https://www.tensorflow.org/install/pip):

```bash
pip3 install --user --upgrade tensorflow # install in $HOME
```

I then look into the intalled directory and found some shared libraries!

```bash
cd /usr/local/lib/python3.5/dist-packages/tensorflow/python
ls -lah _pywrap_tensorflow_internal.so
-rwxr-xr-x 1 root staff 154M Jul  1 09:32 _pywrap_tensorflow_internal.so
```

But guess what? `_pywrap_tensorflow_internal.so` is not the same as the shared library we need for the C binding (`libtensorflow.so.1.14.0`)

I kept looking and then [I found an installation with Docker](https://www.tensorflow.org/install/docker). But again, only possible to build binaries for Python, not for C.

After all my failed attempts, [I opened a case on Github](https://github.com/tensorflow/tensorflow/issues/30359) as a “feature request”.

### Failed attempt 2: looking for non official pre-build binaries

The obvious next step was… “OK, if Google doesn’t do it, then someone else must”. I mean…. Smalltalk is not the only one wanting to bind against the C library, right?

Long short story, I found none. I found [this one](https://github.com/PINTO0309/Tensorflow-bin/issues/10), but it was only python builds (but he said he may try providing shared libraries…so stay tunned!). I then found [this one](https://dl.photoprism.org/tensorflow/) that would work at least for Nvidia Jetson Nano (but I don’t have the Nano with me yet). I found [another one](https://github.com/lhelontra/tensorflow-on-arm/issues/69), but again only Python.

So….in conclusion, I didn’t find the shared library for the Raspberry Pi anywhere. If you are aware of something, please let me know. What was worst was that most answers were “you better compile it yourself”. That didn’t sound too bad…I mean…sure, why not? Until I checked the official size of the Linux x64 shared library and the `libtensorflow.so` was 216MB. WHATTTTTTTTTT? At that moment I thought “OK, this is not gonna be easy”.

### Abandoned attempt: build from scratch on the Pi

My next obvious step was to try to build from scratch on the Pi. For that, I based my work on [this very helpful step by step guide](https://gist.github.com/EKami/9869ae6347f68c592c5b5cd181a3b205). However, time has passed since that guide was written, TensorFlow become “a bit easier” to build on the Pi and so some instructions from it are not necessary anymore. In addition, I found my own problems that were not addressed there.

I recommend you read that guide first and then continue here. Below is what I ended up doing, which is similar to that guide.

Before getting started, some important tips I recommend:

- Have many free GBs in your Pi disk. 
- Be sure to NOT be running anything heavy on the Pi (shutdown X, VNC, docker, whatever that can use CPU or memory).
- Run the build from `ssh`.
- Use `tmux` or similar tool because the process takes hours (many hours) and so you will likely want to power off your development machine and check Pi status the next morning.
- Use heat sinks in your Pi if you don’t want to burn it. 

#### Building the builder: bazel

The first thing is that to build TensorFlow you need the bazel tool. Of course: `sudo apt-get install bazel`, right? hahahahahahah. LOL. I wish it was that simple. Once again, it looks there is no bazel package ready to install on the Pi. So you must first compile it. OK…this thing is becoming meta. I need to build the builder…what’s next? to compile the Linux kernel in which i will build the builder? …

Now…to compile either bazel or TensorFlow, in both cases, the 1GB RAM of your Pi won’t be enough. So you must increase the swap space. In the mentioned guide it mounts an external USB stick / hard disk / etc. In my case, I just increased the swap partition from the SD card to 2GB. But people recommend more….like 8GB (but I didn’t have that much free):

```bash
sudo vim /etc/dphys-swapfile # change CONF_SWAPFILE to 2000
sudo /etc/init.d/dphys-swapfile stop
sudo /etc/init.d/dphys-swapfile start
free -m # confirm we have now 2000mb
```

> IMPORTANT: Whether you success or not with bazel and TensorFlow compilation, its VERY important that you put back the original swap space size (`CONF_SWAPFILE`) when you are done. Else, you will ruin the SD lifespan.

The compilation of bazel can take a few hours. Once I finished and started to compile TensorFlow, I got a wonderful message:

> Please downgrade your bazel installation to version 0.21.0 or lower to build TensorFlow!

Are you kidding me?????? I spent hours compiling the wrong bazel version? FUC… Is there a way to know in advance which bazel version each TensorFlow version needs? I have no clue. If you know, please tell me. Anyway, I started over with the version it needed for the version of TensorFlow I wanted (1.13.1):

```bash
mkdir bazel
cd bazel 
wget https://github.com/bazelbuild/bazel/releases/download/0.21.0/bazel-0.21.0-dist.zip
unzip bazel-0.21.0-dist.zip
env BAZEL_JAVAC_OPTS="-J-Xms384m -J-Xmx1024m" \
JAVA_TOOL_OPTS="-Xmx1024m" \
EXTRA_BAZEL_ARGS="--host_javabase=@local_jdk//:jdk" \
bash ./compile.sh
sudo cp output/bazel /usr/local/bin/bazel
cd ..
rm -rf bazel*
```

The Java options for the memory (the `1024` is because the Pi 3B+ has 1GB RAM) are necessary because else compilation just fails (thanks [freedomtan](https://github.com/freedomtan) for the help). And no, it doesn’t fail with a nice “Out of Memory” but some kind of random error. [I reported that into a Github issue.](https://github.com/bazelbuild/bazel/issues/8882)

The other necessary part `--host_javabase=@local_jdk//:jdk`. I don’t even remember why…it simply wouldn’t work without that.

If you succeed on doing this, save that `bazel` binary everywhere! don’t loose it hahahaha. Again, if you know somewhere where I can find bazel pre-build binaries for the Pi, please let me know.

#### Building TensorFlow

The first steps are trivial:

```bash
git clone --recurse-submodules https://github.com/tensorflow/tensorflow.git
cd tensorflow
git checkout v1.13.1
./configure
```

The `./configure` will ask you a few questions about what support you want to add to the TensorFlow compilation you are about to do. The answers will depend on the hardware you are targeting. For Raspberry Pi I think it’s OK to simple answer false to all of them:

![](https://marianopeck.files.wordpress.com/2019/07/screen-shot-2019-07-13-at-2.30.07-pm.png?w=748)

Watching at the questions, you may get an idea what you will eventually answer for Nvidia Jetson, Parallella Board, etc. And yes, I would like to see if it works on the Parallella Board:

%[https://twitter.com/martinezpeck/status/1147132349450203136]

Finally, time to run compilation. No, don’t grab a beer, you will end drunk. No, don’t take coffee…you will drink so much caffeine that you will not be able to sleep for a whole week.

```bash
bazel --host_jvm_args=-Xmx1024m --host_jvm_args=-Xms384m build \
--config opt --verbose_failures --jobs=3 --local_resources 1024,1.0,1.0 \
--copt=-mfpu=neon-vfpv4 \
--copt=-ftree-vectorize \
--copt=-funsafe-math-optimizations \
--copt=-ftree-loop-vectorize \
--copt=-fomit-frame-pointer \
--copt=-DRASPBERRY_PI \
--host_copt=-mfpu=neon-vfpv4 \
--host_copt=-ftree-vectorize \
--host_copt=-funsafe-math-optimizations \
--host_copt=-ftree-loop-vectorize \
--host_copt=-fomit-frame-pointer \
--host_copt=-DRASPBERRY_PI \
//tensorflow/tools/lib_package:libtensorflow
```

Some interesting points about that:

- Most of the `--copt` and `--host_copt` [were not identified by me](https://github.com/tensorflow/tensorflow/issues/30359). Again, thanks [freedomtan](https://github.com/freedomtan).
- I already explain why the Java memory arguments.
- `--verbose_failures` is useful if our build fails to get some description of what went wrong.
- `--local_resources` helps specify “how much hardware resources” to use.
- For me, it was still failing to build because of low resources. So I ended up adding `--jobs=3` which minimizes the use of resources (but will take longer, obviously). [I got this from a StackOverflow](https://stackoverflow.com/questions/34382360/decrease-bazel-memory-usage).
- It’s interesting to note that building a Python Wheel or a shared library is almost the same process. The only change is that instead of `//tensorflow/tools/lib_package:libtensorflow` (for .so) you use `//tensorflow/tools/pip_package:build_pip_package` to get a Wheel. That’s why I was kindly asking those already providing Wheels, to also provide shared libraries. 
- This process will take many many hours (in my case it took more than 20). So, go to sleep and check the next morning. 

This **should** work. However, as it was taking too much time, I continued looking for other alternatives and I never really let the process to finish. So I can’t confirm it works. And now my SD doesn’t have free space and I already got a working .so (next section). If you try it and it works, let me know! Otherwise, I guess I will try again in the near future.

### Final attempt: cross-compiling

While I was waiting the compilation on the Pi to finish and suffering by watching its green led constantly turned on for hours and hours, I continued looking for more alternatives. By chance, I arrived to [an official link that showed how to cross-compile TensorFlow for the Pi.](https://www.tensorflow.org/install/source_rpi#build_from_source) (I should have seen this before! hahahahaha)

Just to understand how difficult it is to have all the environment setup ready, imagine that the cross-compile procedure is to use Docker and start off from an existing image they provide…

The procedure looked very simple: install docker and then run one shell line:

```bash
tensorflow/tools/ci_build/ci_build.sh PI \
    tensorflow/tools/ci_build/pi/build_raspberry_pi.sh
```

Cool. That sounded magical. Too good to be true. I was then ready to take advantage of all my 8 CPU cores and 16 GB RAM of my MBP. Unfortunately, the process never finished well for me. Each run would fail at a different place and the explanation was never clear. Again, [I opened a case on Github](https://github.com/tensorflow/tensorflow/issues/30764) but no response so far.

I was about to abandon all my attempts for TensorFlow on ARM / SBC. But I had one last idea: try again this cross-compilation with Docker but now on a Linux virtual machine that I had with Linux Mint 18.3. Of course, this VM was never gonna be as fast as doing it directly in my host (OSX), but it should still be much faster than doing it on the Pi.

Call it a miracle or not, but after a few hours, that DID WORK. I successfully got the `.so`, moved it into the Pi and then run my tests. Everything was working:

%[https://twitter.com/martinezpeck/status/1151100675104940032]

### Conclusion

I hope Google would officially ship C binaries at least for the most common SBC like Raspberry Pi or Jetson Nano. If not that, I hope some of those people already compiling Wheels for Raspberry could compile shared libraries too.

Finally, nevertheless, I think it was worth for me learning the low level details of a build from scratch. Why? Because there are many boards I would like to experiment with: Rpi3 but with a ARM 64 OS (Armbian, Ubuntu Server 18.04, etc), Rpi4, Pine64, Jetson Nano, etc. We can even test on an Nvidia Jetson TX2!!! And for all these cases, I won’t be able to use the cross-compile alternative out-of-the-box because that was intended for the Pi only.

%[https://twitter.com/martinezpeck/status/1141775466384306182]

I hope I could have helped someone else aside from my future me. If you have any kind of feedback please share!