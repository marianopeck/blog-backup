## Accessing Raspberry Pi GPIOs with Smalltalk

As I commented in an [earlier post](https://martinezpeck.hashnode.dev/getting-started-with-raspberry-pi-and-smalltalk-cjyhi7727005m5hs1360k0tdr), one of the great features [VASmalltalk](https://twitter.com/instantiations?lang=es) has in the context of IoT, is a wrapper of the C library [pigpio](http://abyz.me.uk/rpi/pigpio/) which allows us to manage GPIOs as well as their associated protocols like 1-Wire, I2C, etc.

In this post we will see the basic setup for getting this to work.

%[https://twitter.com/martinezpeck/status/1129434671467704322]

### Setting up pigpio library

The first step is to install the [pigpio](http://abyz.me.uk/rpi/pigpio/) C library. The easiest way is to install it with a packager manager if the library is available:

```bash
sudo apt-get update
sudo apt-get install pigpio
```

If you want a different version than the one installed by the packager manager or the package is not available, then you can compile it yourself:

```bash
rm master.zip
sudo rm -rf pigpio-master
wget https://github.com/joan2937/pigpio/archive/master.zip
unzip master.zip
cd pigpio-master
make
sudo make install
```

To verify the library is installed correctly, you can execute `pigpiod -v` and that should print the installed version.

### Setting up VASmalltalk 

In [this post](https://martinezpeck.hashnode.dev/getting-started-with-raspberry-pi-and-smalltalk-cjyhi7727005m5hs1360k0tdr) I showed how to get a VASmalltalk ECAP release with ARM support. The instructions are as easy as uncompress the download zip into a desired folder.

The next step is to edit `/raspberryPi/abt32.ini`:

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-05-at-11.14.12-am.png?w=748)

to include the following fields:

```ini
RaspberryGpio=libpigpio.so
RaspberryGpioDaemon=libpigpiod_if2.so
RaspberryGpioUltrasonicDaemon=libpigpioultrasonic.so
```

under the section `[PlatformLibrary Name Mappings]`.

Then, fire up the VASmalltalk image by doing:

```bash
cd raspberryPi/
./abt32.sh
```

Once inside VASmalltalk, go to `Tools` -> `Load/Unload Features…` and load the feature `VA: VAStGoodies.com Tools`.

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-05-at-11.21.26-am.png?w=748)

Then `Tools` -> `Browse Configurations Maps`, right click on the left pane (list of maps) and select `Import from VAStGoodies.com`. That will import the selected version into the connected ENVY manager. 

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-05-at-11.33.39-am.png?w=748)

After that, on the same Configurations Map Browser, you must go to `RaspberryHardwareInterfaceCore` and `RaspberryHardwareInterfaceTest`, select the version you imported, right click and then `Load With Required Maps`. 

![](https://i0.wp.com/marianopeck.blog/wp-content/uploads/2019/09/Screen-Shot-2019-09-04-at-10.57.27-AM.png?resize=768%2C389&ssl=1)

You are done! You have installed pigpio library and the VASmalltalk wrapper. Let’s use it!

### Some GPIO utilities to help you started

Before starting with VASmalltalk, let me show you some Linux utilities that are very useful when you are working with GPIO.

One of the commands is `pinout` which comes with Raspbian. It shows you everything you need to know about your Pi!! Hardware information as well as a layout of the pins:

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-05-at-11.47.46-am.png?w=581&h=770)

And yes, do believe the command line output and visit [https://pinout.xyz](https://pinout.xyz/). It is tremendously useful. A must have.

The other tool is `gpio`. This allows you to see the status of every pin and even pull up / down them right from there. Example below shows how to read all pins and then pull up BCM pin 17 (physical pin 11).

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-05-at-12.03.10-pm.png?w=748)

I don’t want to enter into the details in this post, but as you can see, each pin could have 3 different numbers: physical (the number on board), BCM and wPi (wiring Pi). And they also have a name. So…whenever you are connecting something you must be sure which “mode” they refer too. The number alone is not enough.

### Managing GPIOs from Smalltalk! 

In this post, we will see the most basic scenario of a GPIO which basically means pulling it up or down. When up, it outputs 3.3v (in the case of a Raspberry Pi), when down, 0. This is enough for you to play with LEDs, fans, and anything that doesn’t require “data” but just voltage.

The Smalltalk code for doing that is:

```smalltalk
"This is an example of the VA tooling for accessing and dealing with the GPIOs.
Instructions to run: 
1) Make sure you have pigpio installed and that deamon is running. You can start it 
`sudo pigpiod` from a terminal.
2) On a terminal, run `pinout` and identify GPIO17 (phisical pin 11) and the GROUND 
at phisical pin 14.
3) Take a volt meter and put it in DCV 10 or 20. 
4) Put the negative (usually black) cable into the ground (phisical pin 14) and the positive 
cable (usually red) into the GPIO17  (phisical pin 11). It should display 0V.
5) Run below snipped of code which in fact turns on pin GPIO17 for output. 
6) Repeat number 4) but now you should get a 3.3V output"

| gpioInterface |
[gpioInterface := RaspberryGpioDaemonInterface raspberryGpioStart.
gpioInterface 
	pinSetAsOutput: 17;
	pin: 17 write: 1.
] ensure: [
	gpioInterface shutDown
]
```	

In the comment of above snippet you can see how you can validate that it actually worked… You can use a volt meter or use `gpio readall` to confirm the change. For the volt meter, set it in 10 / 20 volt (DCV) range. Then with the negative cable (usually black) touch any ground pin (for example, physical pin 6) and with the positive cable (usually red) touch GPIO pin 17 (physical pin 11). When the GPIO is on, then you should see the meter register about 3.3 volts and 0 when off. **Welcome to the hardware debugger!**

### Conclusion

In this post you saw how to install pigpio C library, how to install the wrapper in VASmalltalk and see one basic example. In future posts we will see more advanced GPIO uses and protocols.

%[https://twitter.com/martinezpeck/status/1131378821784068096]

### Acknowledgments

The pigpio wrapper (RaspberryHardwareInterface) was a community pushed project. I think Tim Rowledge started with it in Squeak Smalltalk, then Louis LaBrunda started a port to VASmalltalk, and finally [Instantiations](https://twitter.com/instantiations) helped to get that port finished up and running.
