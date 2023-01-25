# Dependencies

* Python 3.6
* TensorFlow 2.1
* Android Studio

# Outputs

# Project Structure

The project has 4 folders

* Android App -> The Android App code
* Model -> It has mode.tflite , it is the trained model
* APK -> The apk.app which can be installed onto an compatible Android device
* Training -> It has Training.ipynb whci can be opened in Google Colab

# How to build the Android App

* git clone <>
* cd Android App
* Open the same on the Android Studio


# How to create the Android App from scratch

1) Create a new project on Android Studio

2) Copy model.tflite and label.txt and create a new folder called assets here /android/app/src/main/assets get both of them inside assets's folder

3) Create a text file in labels.txt with the content as below (WithMask and withoutmask are the labels, refer the Training.ipynb in the training folder):

WithMask
WithoutMask

4) In build.gradle(:app)  

implementation 'org.tensorflow:tensorflow-lite:0.0.0-nightly'
implementation 'org.tensorflow:tensorflow-lite-gpu:0.0.0-nightly'
implementation 'org.tensorflow:tensorflow-lite-support:0.0.0-nightly'
implementation 'com.github.esafirm.android-image-picker:imagepicker:2.3.1'
implementation 'com.github.bumptech.glide:glide:4.5.0'
implementation 'com.google.android.gms:play-services-vision:20.0.0'
api 'com.otaliastudios:cameraview:2.6.2'

Follow the code in this file /android/app/res/layout/activity_main.xml (this contain the UI on the Device)
Follow the code in file /android/app/java/com.facemask.android/MainActivity.kt this is the main code.



# Pseudocode of MainActivity.kt
 ```
    cameraView.addFrameProcessor{ frame ->
        overlayView.boundingBox = function processBitmap(){
            val boundingBoxList = mutableListOf<Box>()
            val faces = faceDetector.detect(frame)
            for (i in 0 until faces.size()) {
                val thisFace = faces.valueAt(i)
                Crop this thisFace object as bitmapCropped
                val label = predict(bitmapCropped)
                Create a Box object to boundingBoxList
            }
            return boundingBoxList
        }
    }

    function predict(Image bitmap){
        1) Load The Model from model.tflite
        2) Create the labels from labels.txt
        3) Process the bitmap to model's input width and height
        4) Resize the bitmap image
        5) Run the model on inputimage buffer and get output buffer
        6) Using TensorLabel object parse the output buffer
        7) Return the label
    }
```


# MainActivity.kt high level explanation 

1) In the activity_main.xml add a com.otaliastudios.cameraview.CameraView object to process frame one by one  and also the com.example.facemaskdetection.OverlayView to show the bouding box

2) In the OnCreate() function 

We are using FaceDetector API (https://developers.google.com/android/reference/com/google/android/gms/vision/face/FaceDetector) from Google Play Services . This API helps to detect face in a given frame .

'''
val faceDetector = FaceDetector.Builder(this).setTrackingEnabled(true).build()
'''

This function returns FaceDetector.Builder object 

3) In the same onCreate() function 

Set lifecycle owner to the the MainActivity.kt

```
cameraView.setLifecycleOwner(this)
```

Then add frameprocessor to process a frame 
```
cameraView.addFrameProcessor{ frame ->
}
```

4) See Box.kt 

Which uses android.graphics.RectF
```
class Box(val rectF: RectF, val label: String, val isMask: Boolean)
```

5) For Each frame we pass it to processBitmap function which returns the List of Box object 
    5.1) Create List of Box Object
        ```
        val boundingBoxList = mutableListOf<Box>()
        ```

    5.2) We detect faces using the detect function in the FaceDetector.Builder object
    val faces = faceDetector.detect(frame)

    5.3 ) For Each face (See predict function)
        5.3.1) Load The Model from model.tflite
            ```
            val modelFile = FileUtil.loadMappedFile(this, "model.tflite")
            ```
        5.3.2) Create the labels from labels.txt
            ```
            val labels = FileUtil.loadLabels(this, "labels.txt")
            ```
        5.3.3) Resize the bitmap image
            ```
            val cropSize = kotlin.math.min(input.width, input.height)
            ResizeWithCropOrPadOp(cropSize, cropSize)
            ```
        5.3.4) Process the bitmap to model's input width and height
            ```
            val imageDataType = model.getInputTensor(0).dataType() 
            val inputShape = model.getInputTensor(0).shape() 

            val outputDataType = model.getOutputTensor(0).dataType() 
            val outputShape = model.getOutputTensor(0).shape() 

            var inputImageBuffer = TensorImage(imageDataType)
            val outputBuffer = TensorBuffer.createFixedSize(outputShape, outputDataType) 

            val imageProcessor = ImageProcessor.Builder()
                .add(ResizeWithCropOrPadOp(cropSize, cropSize)) 
                .add(ResizeOp(inputShape[1], inputShape[2], ResizeOp.ResizeMethod.NEAREST_NEIGHBOR)) 
                .add(NormalizeOp(127.5f, 127.5f)) 
                .build()
            ```
        5.3.5) Run the model on inputimage buffer and get output buffer
            ```
            model.run(inputImageBuffer.buffer, outputBuffer.buffer.rewind())
            ```
        5.3.6) Using TensorLabel object parse the output buffer
            ```
            val labelOutput = TensorLabel(labels, outputBuffer) 
            ```
        5.3.7) Return the label
            ```
            val label = labelOutput.mapWithFloatValue
            return label
            ```

    5.4) Create a Box object and append boundingBoxList

    Return the boundingBoxList object





# Also Refer The Referenced Project

* https://github.com/Cindyalifia/tflite-face-mask-detection-android