---
layout: post
title: Traffic Sign Recognition using Convolutional Neural Network
excerpt: "Machine Learning - Image Classification"
modified: 04/05/2017, 1:27
tags: [CNN, Caffe]
comments: true
category: blog
---

## 1. Data Description
This dataset comes from [German Traffic Sign Recognition Benchmark](http://benchmark.ini.rub.de/?section=gtsrb&subsection=dataset), which contains 39209 images in 43 folders and each folder contain one same kind of traffic sign images taking from different places and at different distance. And the size of the images varies from 30 * 30 pixel to larger than 100 * 100 pixel. Here is how it looks like:  

<img src="/images/GTSRB/Dataset.png">  

## 2. Problem definition

Problem type: Image classification  

Method: Convolutional Neural Network(CNN)    

Software: caffe    

Task:
* i. Transfer original data image from .ppm to .jpg
* ii. Using .jpg to create .lmdb data file
* iii. Compute image mean file .binaryproto using .lmdb file 
* iv. Build network construction 
* v. Train network
* vi. Test the network with new images
* vii. Visualize the test result

## 3. Data Preparation
#### 3.1 Transfer original data image from .ppm to .jpg
I use two for loops to be able to automatically go through all 43 folders and transfer all .ppm files into .jpg. 

~~~ ruby
for f in range(0,43):
    # save folder name as the format: 00042
    folder = format(f, '05d')
    csv_path = image_root + folder + '/GT-' + folder + '.csv'
    csv = pd.read_csv(csv_path, sep= ';')
    ppm_name = csv['Filename']
    # len(ppm_name) refer to how many .ppm images in each folder
    for i in range(0, len(ppm_name)):
        # select 1/3 as testing image
        if i % 3 == 0:
            im = Image.open(image_root + folder + '/' + csv['Filename'][i])
            im.save('image_path_name' + 'TestJPG/' + format(f, '02d') + '_' +
                    csv['Filename'][i].replace('ppm', 'jpg'))
        else:
            im = Image.open(image_root + folder + '/' + csv['Filename'][i])
            im.save('image_path_name' + 'TrainJPG/' + format(f, '02d') + '_' +
                    csv['Filename'][i].replace('ppm', 'jpg'))
~~~
In this step, I separate 1/3 of my dataset into testing images (13070) and the rest as training images (26139) by if the name of the images can be divided by three. And here I add the label of the image(format(f,’02d’)) in front of the image name, which will be helpful to define the label for each image later. Now I have all training images in one folder and so do testing images. Next step is to create lmdb file based on those images. 

#### 3.2 Using .jpg to create .lmdb data file
For image preprocessing, I use histogram equalization to the image for adjusting image intensities to enhance contrast and resize the image to 80 * 80. 
~~~ ruby
IMAGE_WIDTH = 80
IMAGE_HEIGHT = 80

def transform_img(img, img_width=IMAGE_WIDTH, img_height=IMAGE_HEIGHT):

    #Histogram Equalization
    img[:, :, 0] = cv2.equalizeHist(img[:, :, 0])
    img[:, :, 1] = cv2.equalizeHist(img[:, :, 1])
    img[:, :, 2] = cv2.equalizeHist(img[:, :, 2])

    #Image Resizing
    img = cv2.resize(img, (img_width, img_height), interpolation = cv2.INTER_CUBIC)

    return img
~~~

Here are some comparison before I choose use the size = 80:  

| Preprocessing | Small image | Big image |   
|---------------|-------------|-----------|    
| Original images   | <img src="/images/GTSRB/imshow_original.png"> | <img src="/images/GTSRB/imshow_bigOriginal.png"> | 
| Histogram equalization | <img src="/images/GTSRB/imshow_HistEqual.png"> | <img  src="/images/GTSRB/imshow_bigHistEqual.png"> |  
| Resize to 50 * 50 | <img src="/images/GTSRB/imshow_resize50.png"> | <img src="/images/GTSRB/imshow_bigResize50.png"> |  
| Resize to 80 * 80 | <img src="/images/GTSRB/imshow_resize80.png"> | <img src="/images/GTSRB/imshow_bigResize80.png"> |  

Considering my computational power and the clearness of images, I decide to use 80 * 80.

#### 3.3 Compute image mean file .binaryproto using .lmdb file

I use caffe library tool [compute_image_mean](https://github.com/BVLC/caffe/blob/master/tools/compute_image_mean.cpp) to compute mean getting mean.binaryproto file.

#### 3.4 Build network construction

Considering a network layers' instruction from [Stanford CS class CS231n: Convolutional Neural Networks for Visual Recognition](http://cs231n.github.io/convolutional-networks/#layersizepat):  
![Layer](/images/GTSRB/Layer.png)  
 
My network structure is the following:  

<img src="/images/GTSRB/Network.png" width="300px" height="600px">  

#### 3.5 Train network  

In my training processing, the parameters I use are the following:

>test_iter: 50
>test_interval: 1000
>base_lr: 0.001
>momentum: 0.9
>weight_decay: 0.004
>lr_policy: "fixed"

##### Training Results

Overview of the network:  
![network overview](/images/GTSRB/Overview.png)
Performance:
![accuracy](/images/GTSRB/Accuracy.png)  
![loss](/images/GTSRB/Loss.png)  

Visualization of kernels and keature maps from all layers:

|             Kernel                    |              FM               |
|---------------------------------------|-------------------------------|
| ![kernel1](/images/GTSRB/Kernel1.png) | ![fm1](/images/GTSRB/FM1.png) |
| ![kernel2](/images/GTSRB/Kernel2.png) | ![fm2](/images/GTSRB/FM2.png) |
| ![kernel3](/images/GTSRB/Kernel3.png) | ![fm3](/images/GTSRB/FM3.png) |

One example output of image recoganiztion with probability of all 43 labels:  
![prob](/images/GTSRB/Prob.png)  
In this example, the image is categotized as label 31 since it has the highest probability. 

#### 3.6 Test the network with new images

To save trained modole can be done by adding just one command:  
~~~ ruby
solver.net.save('Model_43class.caffemodel')
~~~

Before we can use the well-trained model to test on any new traffic sign image, there are still some steps need to be finished:  
1. Net.prototxt file
(1). Change to input layer instead of data layer that used in training:  
Change the following 
~~~ ruby
name: "GTSRB43"
layer {
  name: "GTSRB43"
  type: "Data"
  top: "data"
  top: "label"
  include {
    phase: TRAIN
  }
  transform_param {
    mean_file: "meanTrain.binaryproto"
  }
  data_param {
    source: "train_lmdb"
    batch_size: 64
    backend: LMDB
  }
}
~~~

to

~~~ ruby
name: "GTSRB43Test"
layer {
  name: "TestData"
  type: "Input"
  top: "data"
  input_param { shape: { dim: 1 dim: 3 dim: 80 dim: 80 } }
}
~~~

(2). remove outcome layer that depend on label, for example:
~~~ ruby
layer {
  name: "accuracy"
  type: "Accuracy"
  bottom: "ip2"
  bottom: "label"
  top: "accuracy"
  include {
    phase: TEST
  }
}
layer {
  name: "loss"
  type: "SoftmaxWithLoss"
  bottom: "ip2"
  bottom: "label"
  top: "loss"
}
~~~



#### 3.7 Result
The fianl testset contains 12630 new traffic images in random order. Here is my full predict result:[German Traffic Sign Recognition Benchmark testset prediction](https://github.com/San-Wang/GTSRB/blob/master/TestResult_Final_small.csv)