今天我们来看一下，不同的模型会提供什么样的接口供我们使用。

我选中的模型是YOLOv3，之所以选择这个模型完全是被它的外观吸引过去的。所以就想看看那个外观和模型本身有没有关系。

还记得那部美剧吗？《疑犯追踪》，没错，就是它了，里边的一些桥段显示出来的场景就是这个样子的。

我打开了上次的那个空的默认的arkit的项目，并且通过拖拽的方式往里边添加了YOLOv3的模型，我注意到里边为这个模型生成了新的代码。

我对比了两个模型自动生成的代码，发现了他们内部的结构和提供的接口基本都一样，但是接口所能接受的参数类型以及接口内部具体的实现还是有差别的。

所以，现在我要做的事情就是替换原来在项目Gesture-Recognition-101-CoreML-ARKit中自带的模型example_5s0_hand_model,取而代之的是YOLOv3。

我需要做的事情就是适配新的模型。

- 一旦我替换完模型之后，项目会立刻报错，说原来的模型找不到了，很正常。找到模型使用的地方，直接替换名字。
- 因为YOLOv3只有12.0以上的版本才支持，所以继续抱怨版本的问题，所以，我就调整了target的版本到12.0. build一下子成功了。
- 因为没check模型的输出结果，直接导致数组访问越界。所以尝试添加如下代码。
```swift
print("the request is: " + String((request.results?.count)!))
        
        if request.results? != nil
```

找到一个很好的例子可以获得我们想要的效果：https://github.com/tucan9389/ObjectDetection-CoreML.git

>类型转换 as?

但是问题在于要使用这个模型需要使用所谓的boxview，而原来我们的这个项目里边是没有boxview这个东西的，所以现在不得不面对storyboard和view的相关知识，这里有个扫盲性质的介绍，可以参考一下。https://juejin.im/post/5a6b173c6fb9a01cbf3891b7


### 做了以下几件事情，总算把程序调出来了
- 原来的工程被我破坏的太厉害，以至于原来的故事板都乱了，所以我就新建了一个arkit的工程，从头开始
- 模仿上述github上的项目，在新创建的工程的故事板上添加了相关的组件，包括
    - 一个View
        - 重新命名这个view为boxes view （名字不指定，不知道有没有问题）
        - 将其对应的class指定为上面github项目中创建的DrawingBoundingBoxView。（这一部分的实现还要继续看）
            - 这个类的主要作用就是将识别出来的物体用方框框起来
            - 在方框上加上相应的label
    - 一个Table View
        - table view的图层和scene view对应的图层是重叠的，所以就存在需要把table view调到前台的需要
        
        ``` swift
            self.sceneView.bringSubviewToFront(self.boxesView)
        ```
    - table里的cell。在我运行项目的时候，总是发生uncached exception：
        > tableView:numberOfRowsInSection:]: unrecognized selector sent to instance
    
        一直不太明白这个错误的含义，很是抓狂，从字面的意思来看，根本看不出来是什么意思。
        参考了开发文档之后，终于得解。 
        我们在前面构建故事板的时候，指定了table view的data source是viewController，所以作为data source的viewController必须要遵从UITableViewDataSource的协议/规定/需求，开发文档中指出，实现UITableViewDataSource的类必须要实现两个方法，即下面代码片段中，提到的两个方法。

        ``` swift
        extension ViewController: UITableViewDataSource {
            func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
                return predictions.count
            }

            func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
                guard let cell = tableView.dequeueReusableCell(withIdentifier: "InfoCell") else {
                    return UITableViewCell()
                }

                let rectString = predictions[indexPath.row].boundingBox.toString(digit: 2)
                let confidence = predictions[indexPath.row].labels.first?.confidence ?? -1
                let confidenceString = String(format: "%.3f", confidence/*Math.sigmoid(confidence)*/)

                cell.textLabel?.text = predictions[indexPath.row].label ?? "N/A"
                cell.detailTextLabel?.text = "\(rectString), \(confidenceString)"
                return cell
            }
        }
        ```

        > UITableViewDataSource的介绍
        >    它是一个protocol
        >    这里不得不提及protocol的概念：
        >    protocol就像是一个蓝图，它定义了需求。
        >    一个类，数据结构可以实现这个需求。

### 事实证明：模型并不能决定显示出来的效果是什么样子。具体显示的效果是要通过不同的view来控制的。
在本例中，通过boxview（UIView）实现了我们想要的带框框和标签的效果。我们将其和viewController关联起来，然后在ViewController中将数据提供给boxview，让它去负责数据的呈现。
