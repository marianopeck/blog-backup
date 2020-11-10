## Developing, testing and mocking the largest MIDI instrument of the world with a Raspberry Pi and a DIY LEDs piano

## “El Carrillón”

“El Carrillón” is a bell tower located in “[La Fiesta Nacional de la Flor](http://www.fiestadelaflor.org.ar/web/index1.php)” in Escobar, Buenos Aires, Argentina. There, a huge event takes place every year:

![](https://marianopeck.files.wordpress.com/2019/06/2019-05-27-2.jpg?w=748)

There are 18 pneumatic bells, controlled by a Raspberry Pi Zero using two [IO Pi Zero boards](https://www.abelectronics.co.uk/p/71/io-pi-zero) and then a driver module composed of triacs. Each IO expander gives us 16 GPIOs with a total of 24 channels. There is a MIDI keyboard connected to the setup, so they can play live, and also record songs.

The Raspberry Pi setup is only two years old, and before that the automation was driven by a couple of PIC processors, that could only play one song: “Ayer” (Spanish for “Yesterday”), as the label in the memory chip advertises. The first move to revamp the instrument was to use a Pi and some python, but the time for the real thing has come, so, python don’t be shy and move aside, Smalltalk is coming.

Each bell is associated to a given musical note. So you can play any MIDI file on the Pi and the notes of the song will actually be mapped to the tuned bells. Of course, there are MIDIs that may not sound very well when played, for example, if they use more than the 18 available notes, or if they use chords or play too fast. So… there are MIDIs that sound better than others. But that’s outside the scope of this post.

![](https://marianopeck.files.wordpress.com/2019/06/pi-controller.jpg?w=748)

![](https://marianopeck.files.wordpress.com/2019/06/keyboard-controller.jpg?w=748)

All this setup, as well as the code (open-source and [available in Github](http://github.com/gerasdf/carrillon)), was developed by my friend Gera Richarte, so all credits go to him. The original code was 100% in Python.

For this year, Gera wanted to provide a web interface (also running in the Pi Zero!!!) so that you can select which songs to play, stop, pause, and do some admin tasks. As I was doing lots of experiments with IoT and [VASmalltalk](https://twitter.com/instantiations), I joined him so that both together could:

- Migrate (at least in part) the MIDI part to Smalltalk.
- Code in Smalltalk the interface to a MCP23017 chip (IO Pi Zero hats) via I2C.
- Create a web interface in Smalltalk to be run in the Pi 

## Smalltalk, start talking MIDI please!

The first thing we needed to do was to be able to “read” the MIDI events of the running MIDI song. For this we relied on some Python tooling and we created a simple `midi2tcp` script and a TCP client in Smalltalk. The Python listens for MIDI events using `rtmidi` and just forwards them to the TCP endpoint running in Smalltalk.

Even if that was enough for our needs, Gera went ahead and also coded a `tcp2midi` in Python so that we could also talk back MIDI events from Smalltalk to Python, and do things like:

```smalltalk
exampleProxyChorder
    "
      self exampleProxyChorder
    "
    | in out evt |
    in := MidiInput localProxy.
    out := MidiOutput localProxy.
    [[true] whileTrue: [
        evt := in nextEvent.
        out nextEventPut: evt.
        (evt isNoteOn | evt isNoteOff) ifTrue: [ 
            evt note: evt note + 4.
            out nextEventPut: evt.
            evt note: evt note + 3.
            out nextEventPut: evt.]
    ]] forkAt: Processor userBackgroundPriority
```

Below you can see a [nice video about it](https://www.youtube.com/watch?v=IIjqkqi6X-Q&feature=youtu.be):

%[https://twitter.com/martinezpeck/status/1133573515079233536]

With this step, we were able to start getting MIDI events on Smalltalk. You can find [this MIDI code here.](https://github.com/gerasdf/carrillon/tree/master/src.st)

## Coding a Smalltalk interface to MCP23017 (IO expander)

On a previous post, you can see how to [get started with Smalltalk and the Raspberry Pi](https://martinezpeck.hashnode.dev/getting-started-with-raspberry-pi-and-smalltalk-cjyhi7727005m5hs1360k0tdr) as well as [how to use the GPIOs with the pigpio wrapper](https://martinezpeck.hashnode.dev/accessing-raspberry-pi-gpios-with-smalltalk-cjysp2yc30004mps1ib050zgs).

The IO expander consist of a MCP23017 chip which supports the protocols I2C and SPI. We used I2C and so we created a class `I2CDeviceGPIOMCP23017` subclass of `I2cDevice`.

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-29-at-12.39.00-am.png?w=748)

But that was not all… the most amazing part is that Gera coded all this on a Linux desktop without any Pi. He wrote all unit tests with an interesting mocking technique (for another post!). And guess what? When we tried on the Pi, the whole `I2CDeviceGPIOMCP23017` was working perfectly.

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-29-at-12.40.38-am.png?w=748)

Eventually he tried on the Pi. But not his. Mine. Remotely. From 10km away. This is thanks to the ability of pigpio to work remotely with a daemon. [See this post for details.](https://martinezpeck.hashnode.dev/remote-controlling-raspberry-pis-from-across-the-world-with-smalltalk-cjyyr9wv0001yens13echp8t1)

%[https://twitter.com/martinezpeck/status/1138123492455530497]

So… after this step, we were able to use our IO expander via I2C from Smalltalk.

## Carrillón Web Interface

This is the part I am being involved the most. It’s a simple server-side rendering application made with Seaside and Bootstrap frameworks. All requests are AJAX. The application is then prepared for deployment and runs headless on the Pi Zero.

This is how it currently looks like:

![](https://marianopeck.files.wordpress.com/2019/06/screen-shot-2019-06-28-at-11.22.21-pm.png?w=330&h=816)

The code is quite simple. What we do to play a song is to fork a OS process and execute the Unix command `aplaymidi`. For example:

```bash
aplaymidi --port 128:0 "$CARRILLON_GIT_ROOT/web/files/Metallica - Nothing Else Matters.mid"
```

Then, the stop button does a `kill -SIGTERM XXXX`, the pause does a ` kill -SIGSTOP` and the resume a `kill -SIGCONT`.

## Putting all pieces together

We have a web interface that can start playing MIDI songs. We have a Python `midi2tcp` that process MIDI notes and send them via TCP to a server in Smalltalk. The MIDI notes are then mapped to concrete GPIOs (representing a particular bell) and we pulse those GPIOs that eventually end up hitting the physical bell.

## Testing with a Do-It-Yourself LEDs piano

So far everything great. But how do we test this without having to go there and test with the physical bells? Well… we could print into the console the MIDI notes being processed as well as the GPIO numbers being activated. But seriously, is that fun? No!

Therefore…I built a 16 leds piano. I assigned each led the correct note exactly in the same order as they are in “El Carrillon”. See the piano live:

%[https://twitter.com/martinezpeck/status/1138868855793770496]

And of course, not only we can play from our web console but I can also play any song with a virtual keyboard hahah:

%[https://twitter.com/martinezpeck/status/1139543333360078848]

That’s all! Hope you enjoyed.