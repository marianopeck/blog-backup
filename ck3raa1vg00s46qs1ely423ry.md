## El Carrillon: playing MIDI songs on an 18-bell tower with a Raspberry Pi and Smalltalk

Some time ago [I blogged about a fantastic project Gerardo Richarte and I were doing with an 18-bell tower located in Argentina](https://martinezpeck.hashnode.dev/developing-testing-and-mocking-the-largest-midi-instrument-of-the-world-with-a-raspberry-pi-and-a-diy-leds-piano-cjz98b31t001vpks1g3nulugc).

Back then, I showed some details of the architecture, code, and how I mocked-up “El Carrillon” with a homemade LED piano so I could test it at home.

In this post, you will see it running live and read some great news about the project!

### Playing live at “La Fiesta Nacional de la Flor”

“El Carrillon” had to play not only during my LED piano testing, but also on the real hardware with real bells during a huge event last November called: [“La Fiesta Nacional de la Flor”](http://www.fiestadelaflor.org.ar/web/).

We first started testing at home with the real hardware, but without the bells to confirm the GPIOs were responding correctly:

%[https://twitter.com/martinezpeck/status/1162360670362460162]

Once that was working, the next obvious step was to move to production and test with physical bells!! The mechanical/hardware part of some bells needed repair, so it took some time until we were able to test in-person. Once we did, everything worked as expected and without much trouble.

Deploying was really easy thanks to the “image” concept of Smalltalk and [a few bash scripts that we had prepared](https://github.com/gerasdf/carrillon/tree/master/scripts).

The good news is that it worked and was a huge success, see below:

%[https://twitter.com/martinezpeck/status/1186979011681030144]

### Won 3rd place at the Innovation Technology Awards at ESUG 2019

We presented this project at the [Innovation Technology Awards at ESUG 2019](https://esug.github.io/2019-Conference/awardsSubmissions.html) held in Koln, Germany. Below is the teaser video we submitted before the competition:

%[https://www.youtube.com/watch?v=mP-7XB4fnao]

Our efforts were worthwhile as we won 3rd place! Thanks to all that voted for us!

%[https://twitter.com/martinezpeck/status/1167051883896352770]

## Conclusion

I personally believe this was a great example of using IoT and Smalltalk. We used [VAST (VA Smalltalk)](https://www.instantiations.com/products/vasmalltalk/) for managing the GPIOs, running the web application, and all within a Raspberry Pi Zero.

VAST has great development and debugging tools, minimal Smalltalk images, good GPIO libraries, and excellent ARM support.

Let’s do more IoT projects with Smalltalk!!!

PS: The official Raspberry Pi twitter account did like our project! :)

%[https://twitter.com/martinezpeck/status/1199306239420837888]

