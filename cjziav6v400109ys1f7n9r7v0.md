## Object Detection with TensorFlow and Smalltalk

In a [previous post](https://dev.to/martinezpeck/recognizing-objects-in-images-with-tensorflow-and-smalltalk-1nep) we saw basic object recognition in images using Google’s [TensorFlow library](https://www.tensorflow.org) from Smalltalk. This post will walk you step by step through the process of using a pre-trained model to detect objects in an image.

It may also catch your attention that we are doing this from [VASmalltalk](https://twitter.com/instantiations) rather than Python. Check out [the previous post](https://marianopeck.blog/2019/08/07/recognizing-objects-in-images-with-tensorflow-and-smalltalk/) to see why I believe Smalltalk could be a great choice for doing Machine Learning.

%[https://twitter.com/martinezpeck/status/1161241020052516864]

### TensorFlow detection model Zoo

In this post, we will be again using a [pre-trained model](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md):

> We provide a collection of detection models pre-trained on the [COCO dataset](http://mscoco.org/), the [Kitti dataset](http://www.cvlibs.net/datasets/kitti/), the [Open Images dataset](https://github.com/openimages/dataset), the [AVA v2.1 dataset](https://research.google.com/ava/) and the [iNaturalist Species Detection Dataset](https://github.com/visipedia/inat_comp/blob/master/2017/README.md#bounding-boxes). These models can be useful for out-of-the-box inference if you are interested in categories already in those datasets. They are also useful for initializing your models when training on novel datasets.

The original idea of using these models was written in this [great post](https://www.kdnuggets.com/2018/03/google-tensorflow-object-detection-api-the-easiest-way-implement-image-recognition.html). Our work was also inspired by [this](https://github.com/tensorflow/models/blob/477ed41e7e4e8a8443bc633846eb01e2182dc68a/object_detection/object_detection_tutorial.ipynb) and [this](https://github.com/priya-dwivedi/Deep-Learning/blob/master/Object_Detection_Tensorflow_API.ipynb) Juypiter notebooks for the demo. 

### Designing the demo with a Smalltalk object-oriented approach

In the [previous post](https://dev.to/martinezpeck/recognizing-objects-in-images-with-tensorflow-and-smalltalk-1nep) you can see that all the demo was developed in the class `LabelImage`. While starting to implement this new demo we detected a lot of common behaviors when running pre-trained frozen prediction models. So… we first created a superclass called `FrozenImagePredictor` and changed `LabelImage` to be a subclass of it, overriding only a small part of the protocol.

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-11.32.54-AM.png?fit=748%2C581&ssl=1)

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-11.33.06-AM.png?fit=748%2C688&ssl=1)

After we finished the refactor it was quite easy to add a new subclass `ObjectDetectionZoo`. And all we needed to implement on that class was just 7 methods (and only 5 methods in`LabelImage`). So…as you can see, it’s quite easy now to add more and more frozen image predictors.

In the [previous example](https://dev.to/martinezpeck/recognizing-objects-in-images-with-tensorflow-and-smalltalk-1nep) (with `LabelImage`) we processed the “raw” results just as TensorFlow would answer it. However, with `ObjectDetectionZoo` the results were a bit more complex and in addition we needed to improve the readability of the information, for example, to render the “bounding boxes”. So we reified the TensorFlow results in `ObjectDetectionImageResults`.

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-11.42.24-AM.png?fit=748%2C455&ssl=1)

These pre-trained models can answer the data for the “bounding boxes”. We were interested in seeing if we could render the image directly from Smalltalk and draw the boxes and labels. Again, time to reify that in `ObjectDetectionImageRenderer`.

![](https://i0.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-11.43.37-AM.png?fit=748%2C472&ssl=1)

To conclude, we have `ObjectDetectionZoo` which will run the model and answer `ObjectDetectionImageResults` and then delegate to `ObjectDetectionImageRenderer` to display and draw the results.

### Running the examples!

To run the examples, you must first check [the previous post](https://dev.to/martinezpeck/recognizing-objects-in-images-with-tensorflow-and-smalltalk-1nep) to see how to install VASmalltalk and TensorFlow. After that, you can check the example yourself in the class comment of `ObjectDetectionZoo`.

This example runs the basic `mobilenet_v1` net which is fast but not very accurate:

```smalltalk
ObjectDetectionZoo new
    imageFiles: OrderedCollection new;
    addImageFile: 'examples\objectDetectionZoo\images\000000562059.jpg';
    graphFile: 'examples\objectDetectionZoo\ssd_mobilenet_v1_coco_2018_01_28\frozen_inference_graph.pb';
    prepareImageInput;
    prepareSession;
    predict;
    openPictureWithBoundingBoxesAndLabel
```

In the [tensorflow-vast repository](https://github.com/vasmalltalk/tensorflow-vast) we only provide a few frozen pre-trained graphs because they are really big. However, you can very easily [download additional ones](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md#coco-trained-models) and use them. The only thing you must do is to uncompress the `.tar.gz` and simply change the one line where you specify the graph (`graphFile:`) to use `rcnn_inception_resnet_v2` and you will see the results are much better: 

![](https://i2.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-12.14.33-PM.png?fit=748%2C587&ssl=1)

You can see with `mobilenet_v1` the spoon was detected as person, the apple on the left and the bowl were not detected and the cake was interpreted as a sandwich. With `rcnn_inception_resnet_v2` all looks correct:

Something very cool from TensorFlow is that you can run multiple images in parallel on a single invocation. So here is another example:

```smalltalk
ObjectDetectionZoo new
    imageFiles: OrderedCollection new;
    graphFile: 'z:\Instantiations\TensorFlow\faster_rcnn_inception_resnet_v2_atrous_coco_2018_01_28\frozen_inference_graph.pb';
    addImageFile: 'examples\objectDetectionZoo\images\000000463849.jpg';
    addImageFile: 'examples\objectDetectionZoo\images\000000102331.jpg';
    addImageFile: 'examples\objectDetectionZoo\images\000000079651.jpg';
    addImageFile: 'examples\objectDetectionZoo\images\000000045472.jpg';
    prepareImageInput;
    prepareSession;
    predict;
    openPictureWithBoundingBoxesAndLabel
```

Which brings these results:

![](https://i1.wp.com/marianopeck.blog/wp-content/uploads/2019/08/Screen-Shot-2019-08-18-at-12.34.23-PM.png?fit=748%2C505&ssl=1)

As you can see [here](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md#coco-trained-models) there are many different pre-trained models so you can use and experiment with any of those. In this post we just took 2 of them (`mobilenet_v1` and `rcnn_inception_resnet_v2`) but you can try with anyone. All you need to do is to download the `.tar.gz` of that model, uncompress it, and specify the graph file with `graphFile:`.

Finally, you can also try with different pictures. You can try with any image of your own or try with the ones provided in the databases used to train these models (COCO, Kitti, etc.).

### Conclusions and Future Work

We keep pushing to show TensorFlow examples from Smalltalk. In the future, we would really like to experiment with training models in Smalltalk itself. Particularly, we want to experiment with IoT boards with GPU (like Nvidia Jetson or similar).

It would also be interesting to try detecting objects on videos aside from pictures.

Finally, thanks to Gera Richarte for the help on this work and to Maxi Tabacman for reviewing the post.