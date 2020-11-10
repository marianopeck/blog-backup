## Why is Smalltalk a good fit for IoT and Edge Computing?

I was gonna start a series of posts about Smalltalk, IoT, Edge Computing, Raspberry Pi, etc. But before that, I would like to answer a question that I am asked each time I present something related with these topics. **Why is Smalltalk a good fit for IoT and edge computing? Does it have unique features over other languages? How would IoT benefit from Smalltalk? **

In this post, I try to write my personal opinion about it.

### Edge computing and hardware improvements

In the first stages of IoT, most of the devices were very hardware-limited and would act more like a PLC or similar.  There wasn’t a “real” Operating System running on the device. Instead, you would be given a language to code something for that target, compile and run it. Running Linux or Smalltalk was not a possibility back then in, what was, a very small and cheap device.

Back then, most of the “code” was just retrieving sensor data and then performing an action: open the garage door when I do X, start air conditioner if temperature is higher than Y, video record my backyard, or making my drink is always at the correct temperature. Many of those initial projects were just for fun, learning, teaching, playing, etc.

%[https://twitter.com/martinezpeck/status/1122989311975145473]

But things have changed in the recent years with the explosion of IoT, AI, ARM processors, Raspberry, and other pieces of the whole picture. One the biggest game changers I can think of, is the ability to run Linux. I think that was a before/after situation. Hardware getting better, cheaper and smaller at the same time.

Today, you can have a machine of 4 cores (\> 1GH each core), 4 GB RAM, eMMC/microSD, HDMI, USB 3.0 and GPU that fits in your hand and costs less than 50USD. These machines (Raspberry Pi 3+, Pine64, BananaPi, ODROID-N2, and others) are so good that there are are even clusters specially made from them. See [PicoCluster](https://www.picocluster.com/) and [MiniNodes](https://www.mininodes.com/) just as an example. Or my own 2-nodes cluster :)

%[https://twitter.com/martinezpeck/status/1128397334864388096]

Anyway, the point is that, given the current hardware and software status, the original “code” that was just getting sensor data and pass it to a server started to become more and more complex. Some “computation” has been moving from the server into the “edge device”. Just just like server side rendering is moving to fat JS clients :) So suddenly, you can now be running really complex business logic in a small device.

Not only is the “code” becoming real domain apps, but these small devices are also starting to be used in **industry**. One easy reason is to save costs. See this [Sony example](https://www.forbes.com/sites/parmyolson/2019/03/10/how-sony-sped-up-a-factory-with-these-tiny-35-computers/).

And there’s where I believe Smalltalk is a good fit.

%[https://twitter.com/martinezpeck/status/1130125335863975942]

### Why Smalltalk could be a good fit for IoT?

- **True OOP:** at the very least, Smalltalk would have all the benefits it also has when running on a regular computer, a server or the cloud: objects all the way down, efficiency, simplicity, live environment, dynamic typing, as well as many others. It’s not my intention here to list why Smalltalk is good or bad. There is plenty of material around that in books and webs.
- **Easy deployment:** If you have already deployed Smalltalk systems, you know how much easier it is to do it compared to other languages. Yes, another great feature of the “image”.
- **Incredible debugging experience:** why write a plain string stacktrace if you can actually dump the real stack into a binary file and materialize it later for debugging? If you are familiar with Smalltalk, you probably read about that [here](https://marianopeck.wordpress.com/2012/01/19/moving-contexts-and-debuggers-between-images-with-fuel/) or [here](http://forum.world.st/ANN-PharoLambda-a-demo-of-Pharo-running-on-AWS-Lambda-td4954910.html). If you are not, then that will crash your head. Yes, you understood correct: on the deployed system you serialize the whole stack (with its variables) at the moment the exception happens into a file. Later, on whatever machine, take that file and materialize it to get back a debugger with the original stack!
- **Remote debugging:** wait… dumping a living stack into a file for further analysis is not enough? Ok, you can have LIVE remote debugger. That is, an exception is raised in your small device, and you can get a debugger open in the development environment of your laptop. And yes, you can change code, save it, and resume the exception (which is running on the device!!!).

%[https://twitter.com/martinezpeck/status/1112072127933542400]

- **Scalability and availability:** a Smalltalk image makes it easier to deploy a system. But, to scale horizontally or provide availability you still need to do quite sysadmin work. However, Smalltalk plays really well with state of the art tools like Docker (see my [series of posts](https://hashnode.com/series/getting-started-with-docker-raspberry-pi-and-smalltalk-cjy5vkygs001jyys18bwjukyq)) and Kubernetes.
- **Maturity:** who can give you 40 years of history and continued improvements? The fact that it’s old doesn’t necessary make it outdated.
- **GPIO accessing:** Smalltalk has bindings for a few of the popular GPIO accessing libs like [wiringpi](http://wiringpi.com/) or [pigpio](http://abyz.me.uk/rpi/pigpio/). That means that from Smalltalk itself you can manage the GPIO pins as well as the known protocols like 1-Wire, I2C, SPI, etc.
- **Uniformity:** imagine that you can use the same language, IDE and tools for the device that sensors data (GPIO access), for the device doing “edge computing” (may be same same as the former) and for server-side logic?
- **Transparent object oriented persistency:** imagine if from that small device that runs your business logic you can transparently persist objects on an object database running on a remote cloud?

%[https://twitter.com/martinezpeck/status/1105224167845257216]

### Conclusion

Am I saying that Smalltalk is the best system for IoT? No. It obviously has drawbacks. Exactly the same thing for Python. There is no silver bullet. But what I want to say is that I think Smalltalk has enough unique features to be, at the very least, considered as a serious alternative when doing IoT.

There have been a few new areas over the last 40 years like web and mobile apps. Now we have IoT.  **Is this the computer revolution our Smalltalk father Alan Kay has been anticipating? I don’t know. But to me, it looks revolutionary enough to be prepared.**
