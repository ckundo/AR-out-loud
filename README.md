# Using Vision in Real Time with ARKit

Manage Vision resources for efficient execution of a Core ML image classifier, and use SpriteKit to display image classifier output in AR.

## Overview

This sample app runs an [ARKit][0] world-tracking session with content displayed in a SpriteKit view. The app uses the [Vision][1] framework to pass camera images to a [Core ML][2] classifier model, displaying a label in the corner of the screen to indicate whether the classifier recognizes anything in view of the camera. After the classifier produces a label for the image, the user can tap the screen to place that text in AR world space. 

- Note: The Core ML image classifier model doesn't recognize and locate the 3D positions of objects. (In fact, the `Inceptionv3` model attempts only to identify an entire scene.) When the user taps the screen, the app adds a label at a real-world position corresponding to the tapped point. How closely a label appears to relate to the object it names depends on where the user taps.


[0]:https://developer.apple.com/documentation/arkit
[1]:https://developer.apple.com/documentation/vision
[2]:https://developer.apple.com/documentation/coreml

## Getting Started

ARKit requires iOS 11.0 and a device with an A9 (or later) processor. ARKit is not available in iOS Simulator. Building the sample code requires Xcode 9.0 or later.

## Implement the Vision/Core ML Image Classifier

The sample code's [`classificationRequest`](x-source-tag://ClassificationRequest) property, [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method, and [`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) method manage:

- A Core ML image-classifier model, loaded from an `mlmodel` file bundled with the app using the Swift API that Core ML generates for the model
- [`VNCoreMLRequest`][3] and [`VNImageRequestHandler`][4] objects for passing image data to the model for evaluation

For more details on using [`VNImageRequestHandler`][4], [`VNCoreMLRequest`][3], and image classifier models, see the [Classifying Images with Vision and Core ML][5] sample-code project. 

[3]:https://developer.apple.com/documentation/vision/vncoremlrequest
[4]:https://developer.apple.com/documentation/vision/vnimagerequesthandler
[5]:https://developer.apple.com/documentation/vision/classifying_images_with_vision_and_core_ml

## Run the AR Session and Process Camera Images

The sample `ViewController` class manages the AR session and displays AR overlay content in a SpriteKit view. ARKit captures video frames from the camera and provides them to the view controller in the [`session(_:didUpdate:)`][6] method, which then calls the [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method to run the Vision image classifier.

``` swift
func session(_ session: ARSession, didUpdate frame: ARFrame) {
    // Do not enqueue other buffers for processing while another Vision task is still running.
    // The camera stream has only a finite amount of buffers available; holding too many buffers for analysis would starve the camera.
    guard currentBuffer == nil, case .normal = frame.camera.trackingState else {
        return
    }
    
    // Retain the image buffer for Vision processing.
    self.currentBuffer = frame.capturedImage
    classifyCurrentImage()
}
```
[View in Source](x-source-tag://ConsumeARFrames)

[6]:https://developer.apple.com/documentation/arkit/arsessiondelegate/2865611-session

## Serialize Image Processing for Real-Time Performance

The [`classifyCurrentImage()`](x-source-tag://ClassifyCurrentImage) method uses the view controller's `currentBuffer` property to track whether Vision is currently processing an image before starting another Vision task. 

``` swift
// Most computer vision tasks are not rotation agnostic so it is important to pass in the orientation of the image with respect to device.
let orientation = CGImagePropertyOrientation(UIDevice.current.orientation)

let requestHandler = VNImageRequestHandler(cvPixelBuffer: currentBuffer!, orientation: orientation)
visionQueue.async {
    do {
        // Release the pixel buffer when done, allowing the next buffer to be processed.
        defer { self.currentBuffer = nil }
        try requestHandler.perform([self.classificationRequest])
    } catch {
        print("Error: Vision request failed with error \"\(error)\"")
    }
}
```
[View in Source](x-source-tag://ClassifyCurrentImage)

- Important: Making sure only one buffer is being processed at a time ensures good performance. The camera recycles a finite pool of pixel buffers, so retaining too many buffers for processing could starve the camera and shut down the capture session. Passing multiple buffers to Vision for processing would slow down processing of each image, adding latency and reducing the amount of CPU and GPU overhead for rendering AR visualizations.

In addition, the sample app enables the [`usesCPUOnly`][7] setting for its Vision request, freeing the GPU for use in rendering.

[7]:https://developer.apple.com/documentation/vision/vnrequest/2923480-usescpuonly

## Visualize Results in AR

The [`processClassifications(for:error:)`](x-source-tag://ProcessClassifications) method stores the best-match result label produced by the image classifier and displays it in the corner of the screen. The user can then tap in the AR scene to place that label at a real-world position. Placing a label requires two main steps. 

First, a tap gesture recognizer fires the [`placeLabelAtLocation(sender:)`](x-source-tag://PlaceLabelAtLocation) action. This method uses the ARKit [`hitTest(_:types:)`][8] method to estimate the 3D real-world position corresponding to the tap, and adds an anchor to the AR session at that position.

``` swift
@IBAction func placeLabelAtLocation(sender: UITapGestureRecognizer) {
    let hitLocationInView = sender.location(in: sceneView)
    let hitTestResults = sceneView.hitTest(hitLocationInView, types: [.featurePoint, .estimatedHorizontalPlane])
    if let result = hitTestResults.first {
        
        // Add a new anchor at the tap location.
        let anchor = ARAnchor(transform: result.worldTransform)
        sceneView.session.add(anchor: anchor)
        
        // Track anchor ID to associate text with the anchor after ARKit creates a corresponding SKNode.
        anchorLabels[anchor.identifier] = identifierString
    }
}
```
[View in Source](x-source-tag://PlaceLabelAtLocation)

Next, after ARKit automatically creates a SpriteKit node for the newly added anchor, the [`view(_:didAdd:for:)`][9] delegate method provides content for that node. In this case, the sample [`TemplateLabelNode`](x-source-tag://TemplateLabelNode) class creates a styled text label using the string provided by the image classifier.

``` swift
func view(_ view: ARSKView, didAdd node: SKNode, for anchor: ARAnchor) {
    guard let labelText = anchorLabels[anchor.identifier] else {
        fatalError("missing expected associated label for anchor")
    }
    let label = TemplateLabelNode(text: labelText)
    node.addChild(label)
}
```
[View in Source](x-source-tag://UpdateARContent)

[8]:https://developer.apple.com/documentation/arkit/arskview/2875733-hittest
[9]:https://developer.apple.com/documentation/arkit/arskviewdelegate/2865588-view