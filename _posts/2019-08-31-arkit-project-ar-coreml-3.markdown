今天我们来研究下另外一个项目的代码： Gesture-Recognition-101-CoreML-ARKit。

这个项目的主要功能：捕捉手势，在屏幕的中央来显示对应的动画手势。

### ViewController

我们还是从类ViewController的实现开始讲起。

在这个类中，我们需要如下的几个要素：
- ARSCNView 
- dispatchqueue
- request
- **viewDidload**
- **viewWillAppear**
- **viewWillDisappear**
- renderer
- loopCoreMLUpdate
- updateCoreML
- classificationCompleteHandler

以上函数中，加粗的是重新实现了父类的方法，实现定制化。其他的函数都是该类内部函数，供其自己使用。

### ARSCNView
这个是arkit项目的必备品。
结合我们之前看过的几个项目，我们可以大概总结一下对这个对象进行的操作。

- 指定delegate，将其关联到ViewController
- 关联指定的场景，在ARkit中，我们使用的几乎都是SCNScene。
- 启动/终止session。

#### SCNScene
这是一个3d场景，3d场景中的物体对象是以node的形式组织起来的。

### dispatchqueue
本章，我们将详细讨论下dispatchqueue，因为它在arkit+coreml的项目中很重要。

在apple的开发网站上，dispatchqueue的定义是这样子的。
>An object that manages the execution of tasks serially or concurrently on your app's main thread or on a background thread.
>一个对象，它可以管理任务的执行，可以是顺序执行，也可以是并发执行，任务是在app的主线程或者后台线程上执行的。

#### 该队列的几个特性
- 像所有的队列一样，它是先进先出的。
- 提交给该队列的任务是由系统提供的一个线程池来执行
    - 除了代表你的app的主线程的那个dispatchqueue之外，系统不能保证你提交的任务是给哪个线程去执行的

>需要注意的是：不要在主线程上顺序的执行一个任务，否则会造成死锁。

既然是线程池，那么就会面临线程池中的线程资源被用光的问题，那么如何才能避免呢？

- 如果你的任务想并发执行，那么尽量确保你的任务不要block其所在的线程
    - 因为线程一旦被堵住，系统就会使用线程池中的其他线程来执行queue中的其他任务。
- 尽量不要给每个类创建过多的私有并发dispatchqueue，如果可能的话，尽量将任务提交给全局并发dispatchqueue。
    - 那样既可以保证队列的顺序化行为，同时又不至于引发线程资源紧张的问题。

所以在我们这个项目中，创建了一个私有的queue。

### request
这是一个request的数组，里边放的呢是VNRequest，VNRequest是个基类，是不能被实例化的。但是它的各种子类是可以被实例化的，在本项目中用到的VNCoreMLRequest就是其中的一种，它是VNImageBasedRequest的子类。所以这几个类之间的关系是。

```mermaid
graph TD
A[VNCoreMLRequest] --> |继承| B(VNImageBasedRequest)
B --> |继承| C[VNRequest]

```
*VNCoreMLRequest的主要功能就是使用模型进行预测，并将结果返回。*

### viewDidLoad
视角/视图加载完成之后会调用这个函数，那么在这个函数中会做什么事情呢？
- 调用父类的同名函数
- 将视图的delegate设置为当前的viewController
- 为视图创建场景
- 选择模型
- 创建VNCoreMLRequest
    - 指定模型
    - 指定结果处理函数
- 开始模型处理的循环

### 函数loopCoreMLUpdate
这个函数做的事情从代码表面来看很简单，它给dispatchqueue提交了一个任务。任务是以代码块的形式提交的。在该代码块中，
- 触发request来使用模型处理数据
    - 在本项目中，就是相机截取到的image
- 递归的调用自己，使得循环继续。

正如前面提到的，在本函数中，使用的async的方式向dispatchQueue提交任务，之所以如此，是因为在使用模型来处理相机截取的image是需要花时间的，同步执行的话势必会将线程堵死，此时的表现就是视角中的图片死掉了。

既然是异步处理，那自然需要提供在模型处理完数据之后的处理函数。该处理函数在定义创建request的时候就已经指定好了。

### 结果处理函数
- 读取在request的结果中保存着的模型处理的结果
    - 根据模型的不同，返回的内容也会不同，这个是由模型决定的
    - 在本项目中，模型是用来进行分类的，所有处理的结果当然就是模型从当前的图像中识别出来的物体。
- 将模型处理结果按照你自己想要的方式进行拼接，处理。
- 渲染你的结果

### 结果的最终渲染
结果的渲染是提交给全局的main dispatchQueue进行处理的，而且是以异步的方式（这主要是为了防止死锁）。注意，此处提交的也是一个序列化的代码块。

所以这里有两个queue，一个queue是私有queue，专门用来处理request。一个是全局的主queue，这个queue专门用来在拿到了处理结果之后，进行结果的渲染。

在拿到了处理结果之后，将额外处理完的文字图像赋值给了两个特殊的对象，debugTextView和textOverlay。

开始的时候，不是很理解，问题在于，我这里只是做了赋值，那什么时候显示到屏幕上了，是不是需要另外去渲染。后来发现，其实并不需要，特殊之处就在于debugTextView和textOverlay就像dock一样，它是一直在屏幕上的。我们在后期给他们赋值会直接显示在dock中，不需要另外的渲染。

至此，项目研究完毕。

### 后续
在后续的章节会再分析一两个项目，弄清楚要实现一个AR+CoreML要实现哪些功能/函数。
另外，研究一下不同的模型对应的接口，看看他们之间的异同之处，为后续将自己训练的模型融入到app中捉准备。






