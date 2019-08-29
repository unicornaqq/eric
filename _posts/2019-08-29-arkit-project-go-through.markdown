### 起点

在打开这个应用的时候，可以看到一架飞机，那总归是一个图片，一般来说对于一个软件工程来讲，这类软件要用到的图片总归是放在资源文件夹中的。在这个新开的项目中，有个叫art.scnassets的文件夹，就它的字面意义来看，这个应该就是专门存放场景资料的文件夹。

在这个目录中，有一个叫ship.scn的文件。我就用这个文件名去搜索，看是哪块代码使用了这个场景资源。

```swift
// Create a new scene
        let scene = SCNScene(named: "art.scnassets/ship.scn”)!
```

### Scene
上面的这段代码创建了一个SCNScene的对象，这个对象Scenekit中的一部分，用来描述一个3d场景。这里的named，制定了我们需要在image中寻找的文件的名字，关于named到底是什么意思，后续还需要在研究一下。
- [x] named是什么意思？

### ViewController
创建scene的代码就在**ViewController**这个类中，所有接下来我们就来看一下这个类的内容。

```swift
class ViewController: UIViewController, ARSCNViewDelegate {

    //该类有个成员变量：sceneView，
    //注意它的类别是ARSCNView，我们在后面会重点讨论这个类的情况。
    @IBOutlet var sceneView: ARSCNView!
    
    override func viewDidLoad() {
        //这里的super指的是UIViewController
        //而且这个函数是在-loadView之后调用
        super.viewDidLoad()
        
        //设置当前view（ARSCNView）的delegate
        //将其设置为当前的ViewController，这就是为什么ViewController也要实现ARSCNViewDelegate这个protocol的原因。
        sceneView.delegate = self
        
        // Show statistics such as fps and timing information
        sceneView.showsStatistics = true
        
        // Create a new scene
        let scene = SCNScene(named: "art.scnassets/ship.scn")!
        
        //将场景添加到视图中去。
        sceneView.scene = scene
    }
    
    //视图即将被显示的时候调用。
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        // Create a session configuration
        let configuration = ARWorldTrackingConfiguration()

        // Run the view's session
        sceneView.session.run(configuration)
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        // Pause the view's session
        sceneView.session.pause()
    }

    // MARK: - ARSCNViewDelegate
    
/*
    // Override to create and configure nodes for anchors added to the view's session.
    func renderer(_ renderer: SCNSceneRenderer, nodeFor anchor: ARAnchor) -> SCNNode? {
        let node = SCNNode()
     
        return node
    }
*/
    
    func session(_ session: ARSession, didFailWithError error: Error) {
        // Present an error message to the user
        
    }
    
    func sessionWasInterrupted(_ session: ARSession) {
        // Inform the user that the session has been interrupted, for example, by presenting an overlay
        
    }
    
    func sessionInterruptionEnded(_ session: ARSession) {
        // Reset tracking and/or remove existing anchors if consistent tracking is required
        
    }
}
```

### ARSCNView
该view是arkit中的一部分，是为了在照相机镜头捕捉到的真实世界的图像中叠加的3d事物。

前面说过，这个view是两个世界的叠加，一个是现实世界，一个是虚拟世界。现实世界是背景，而虚拟世界是前景。在这个view中，两个世界的坐标系统是对应的，这也许就是为什么AR中能产生锚定的效果。而且，随着手机的移动，虚拟的世界也会跟着调整。

### delegate
在代码的很多地方都能看到delegate。delegate指向的是一个用户自己提供的对象。这个对象在3d场景渲染的过程中（有很多的步骤），会收到一些特定的事件通知。这就给用户提供了一个控制渲染的机会（添加自己想要渲染的内容，或者做一些基于每一帧画幅的逻辑处理）。

- [x] ARSCNView
- [ ] @IBOutlet

### 关于怎么写自己的delegate
在上面的示例项目中，有一段是要自己来实现的（注释掉了），于是我就参考其他的例子，借鉴了一下人家是如何实现的。

这个项目来自于ARKit2.0-Prototype。

接下来我们就来看下这段代码的意思。

```swift
    // MARK: - ARSCNViewDelegate
    //根据官方文档的解释，这个函数是在节点被添加到锚点完成之后会调用的函数。
    func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
        let alertController = UIAlertController(title: "Found Object", message: "This is your object.", preferredStyle: .alert)
        let action1 = UIAlertAction(title: "OK", style: .default) { (action:UIAlertAction) in
            print("You've pressed default");
        }
        alertController.addAction(action1)
        self.present(alertController, animated: true, completion: nil)
    }
    
    // Override to create and configure nodes for anchors added to the view's session.
    //一下这段代码就是apple自带的默认代码，参数，甚至包括第一行let代码都不要该，我们只需要实现后面的代码即可。这段代码用来创建一个节点。
    func renderer(_ renderer: SCNSceneRenderer, nodeFor anchor: ARAnchor) -> SCNNode? {
        let node = SCNNode()

        //这个anchor是外面传进来的参数。as?向下转型（基类转子类，与as!的差别在于，如果转换失败，将会转变成nil）。所以这里的逻辑是，先尝试从ARAnhor转成ARImageAnchor，如果转成功了，就继续处理，如果没有转成功的话，就走到else分支上去了。
        //ARImageAnchor: 代表一张图片的锚点。
        if let imageAnchor = anchor as? ARImageAnchor {
            //先创建一个平面，该平面可以设置它的宽度和高度，只有一面可见。它是scenekit的一部分。
            let plane = SCNPlane(width: imageAnchor.referenceImage.physicalSize.width, height: imageAnchor.referenceImage.physicalSize.height)
            //firstMaterial?: Optional Chaining, firstMaterial可能为空，如果为空的话就直接退出，不会造成runtime error。
            plane.firstMaterial?.diffuse.contents = UIColor(white: 1, alpha: 0.8)
            //SCNMaterial： 一个material就确定了如何渲染一个3d物体，它规定了颜色和相关的文字。
            let material = SCNMaterial()
            //diffuse光照条件。
            material.diffuse.contents = viewObj
            plane.materials = [material]
            let planeNode = SCNNode(geometry: plane)
            planeNode.eulerAngles.x = -.pi / 2
            
            node.addChildNode(planeNode)
            
        }
        else{
            //planeNode.eulerAngles = SCNVector3Make(0,0,Float(-M_PI_2))
            
            let shipScene = SCNScene(named: "art.scnassets/ArrowB.scn")!
            let shipNode = shipScene.rootNode.childNodes.first!
            shipNode.position =  SCNVector3.positionFromTransform(anchor.transform)
            shipNode.position.y = 0.25
            print(shipNode.position)
            sceneView.scene.rootNode.addChildNode(shipNode)
        }
        return node
    }
```

### renderer
这个是协议ARSCNViewDelegate提供的接口函数，从ARSCNViewDelegate的代码中可以看到，它有好几套同名接口，只是参数不同，使用的场景也不同。

- 在给定的锚点使用定制化的节点（node）。该节点会被在场景中被创建出来。这个实现就是上述代码示例中的第二个renderer的实现。
```swift
optional func renderer(_ renderer: SCNSceneRenderer, nodeFor anchor: ARAnchor) -> SCNNode?
```
- 在一个新的节点被映射到给定锚点的时候该函数会被调用。注意那个didAdd的符号。
```swift
optional func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor)
```
- 锚点更新了，节点也需要更新，此时该函数会被调用。
```swift
optional func renderer(_ renderer: SCNSceneRenderer, willUpdate node: SCNNode, for anchor: ARAnchor)
```
- 锚点更新了，节点也更新了，此时该函数会被调用。
```swift
optional func renderer(_ renderer: SCNSceneRenderer, didUpdate node: SCNNode, for anchor: ARAnchor)
```
- 当节点被从制定的锚点上被移除了，此时该函数会被调用。
```swift
optional func renderer(_ renderer: SCNSceneRenderer, didRemove node: SCNNode, for anchor: ARAnchor)
```

### ARAnchor
ARAnchor代表在一个3d场景中的位置和朝向。在这个类的内部，有个矩阵，该矩阵定义了rotation是怎么做的。
