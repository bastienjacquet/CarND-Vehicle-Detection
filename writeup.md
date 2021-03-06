# Vehicle Detection Project

# Introduction
In this project, the goal is to write a software pipeline to identify vehicles in a video from a front-facing camera on a car. I tried two different solutions:
* Histogram of Oriented Gradients (HOG) image descriptor and a Linear Support Vector Machine (SVM)
* SSD (Single Shot MultiBox Detector)

Here, for the impatient ones, an extract of the results:
#### HOG + SVM:
![](./videos/result_hog_svm.gif)
#### SSD:
![](./videos/result_hog_ssd.gif) 

# First solution: HOG + SVM
The steps of this project are the following:

* Get the training data: we need images of cars (positive samples) and images representing something else (negative samples).
* Feature extraction (for each sample of the training set):
  * Perform a [Histogram of Oriented Gradients (HOG)](http://lear.inrialpes.fr/people/triggs/pubs/Dalal-cvpr05.pdf) feature extraction
  * Extract binned color features, as well as histograms of color
  * Concatenate the previous results in a vector and normalize
* Train a classifier ( for example, a linear SVM classifier)
* Implement a sliding-window technique and use the trained classifier to search for vehicles in images.
* Run the pipeline on a video stream and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

You can find the code in the IPython notebook named [Vehicle-Detection.ipynb](https://github.com/jokla/CarND-Vehicle-Detection/blob/master/Vehicle-Detection.ipynb).

[//]: # (Image References)
[image1]: ./examples/car_not_car.png
[image2]: ./examples/HOG_example.jpg
[image3]: ./examples/sliding_windows.jpg
[image4]: ./examples/sliding_window.jpg
[image5]: ./examples/bboxes_and_heat.png
[image6]: ./examples/labels_map.png
[image7]: ./examples/output_bboxes.png
[video1]: ./project_video.mp4

## Traning data

I used the data provided by Udacity. Here are links to the labeled data for [vehicle](https://s3.amazonaws.com/udacity-sdc/Vehicle_Tracking/vehicles.zip) and [non-vehicle](https://s3.amazonaws.com/udacity-sdc/Vehicle_Tracking/non-vehicles.zip). These example images come from a combination of the [GTI vehicle image database](http://www.gti.ssr.upm.es/data/Vehicle_database.html), the [KITTI vision benchmark suite](http://www.cvlibs.net/datasets/kitti/), and examples extracted from the project video itself.

The dataset contains in total 17,760 color images of dimension 64×64 px. 8,792 samples contain a vehicle and 8,968 samples do not. 

## Feature extraction

I started by reading in all the `vehicle` and `non-vehicle` images.  Here is an example of one of each of the `vehicle` and `non-vehicle` classes:

<img src="./examples/car_not_car.png" width="500" alt="" />  

These are the features I used in this project:
* Spatial features: a down sampled copy of the image
* Color histogram features that capture the statistical color information of each image. Cars often have very saturated colors while the background has a pale color. This feature could help to identify the car by the color information.
* Histogram of oriented gradients (HOG): that capture the gradient structure of each image channel and work well under different lighting conditions

As you can see in the next picture, even reducing the size of the image to 32 x 32 pixel resolution, the car itself is still clearly identifiable, and this means that the relevant features are still preserved.

<img src="./examples/spatial-binning.jpg" width="600" alt="" />   

This is the function I used to compute the spatial features, it simply resizes the image and flatten to a 1-D vector:

```
# Define a function to compute binned color features  
def bin_spatial(img, size=(32, 32)):
    # Use cv2.resize().ravel() to create the feature vector
    features = cv2.resize(img, size).ravel() 
    # Return the feature vector
    return features
```

The second feature I used are the histograms of pixel intensity (color histograms). The function `color_hist` compute the histogram of the color channels separately and after it concatenates them in a 1-D vector.

```
# Define a function to compute color histogram features  
def color_hist(img, nbins=32, bins_range=(0, 256)):
    # Compute the histogram of the color channels separately
    channel1_hist = np.histogram(img[:,:,0], bins=nbins, range=bins_range)
    channel2_hist = np.histogram(img[:,:,1], bins=nbins, range=bins_range)
    channel3_hist = np.histogram(img[:,:,2], bins=nbins, range=bins_range)
    # Concatenate the histograms into a single feature vector
    hist_features = np.concatenate((channel1_hist[0], channel2_hist[0], channel3_hist[0]))
    # Return the individual histograms, bin_centers and feature vector
    return hist_features
```

I then explored different color spaces and different `skimage.hog()` parameters (`orientations`, `pixels_per_cell`, and `cells_per_block`).  I grabbed random images from each of the two classes and displayed them to get a feel for what the `skimage.hog()` output looks like.

Here is an example using the `YCrCb` color space and HOG parameters of `orientations=12`, `pixels_per_cell=(8, 8)` and `cells_per_block=(2, 2)`:


<img src="./examples/YCrCb_example.png" width="500" alt="" />   
<img src="./examples/HOG_example.png" width="500" alt="" />   

Here the code that extracts the features:
```
### Traning phase
car_features = extract_features(cars, spatial_size=spatial_size, hist_bins=hist_bins, orient=orient, 
                        pix_per_cell=pix_per_cell, cell_per_block=cell_per_block, 
                        hog_channel=hog_channel)
print ('Extracting not-car features')
notcar_features = extract_features(notcars, spatial_size=spatial_size, hist_bins=hist_bins, orient=orient, 
                        pix_per_cell=pix_per_cell, cell_per_block=cell_per_block, 
                        hog_channel=hog_channel)
```

As in any machine learning application, we need to normalize our data. In this case, I use the function called  StandardScaler() in thePython's sklearn package.

```
# Create an array stack of feature vectors
X = np.vstack((car_features, notcar_features)).astype(np.float64)                        
# Fit a per-column scaler
X_scaler = StandardScaler().fit(X)
# Apply the scaler to X
scaled_X = X_scaler.transform(X)
```
Now we can create the labels vector, shuffle and split the data into a training and testing set:
```
# Define the labels vector
y = np.hstack((np.ones(len(car_features)), np.zeros(len(notcar_features))))

# Split up data into randomized training and test sets
rand_state = np.random.randint(0, 100)
X_train, X_test, y_train, y_test = train_test_split(
    scaled_X, y, test_size=0.2, random_state=rand_state)

```

We are ready to train our classifier!


### Training phase


I tried various combinations of parameters, seeking to keep the length of the feature vector as small as possible. Practically, I run the SVM classifier several times changing the parameters to get the best accuracy value for the test set. Firstly, I tried different color spaces, and it turned out that the `YCrCb` color space gave me the best result. I used, therefore, the `YCrCb` color space to compute all the features.

After, I took care of the HOG parameters. I found out that the parameters `orient`, `pix_per_cell`, `cell_per_block` did not have a big impact while using all the channels increased the accuracy of 1%.

Finally, I decided to use the smallest values possible for the spatial size and histogram bins without losing in accuracy. Tweaking the parameters improved the accuracy from 97% to 99%. These are the parameters that gave me the best result:


```
# HOG parameters
orient = 12
pix_per_cell = 8
cell_per_block = 2
hog_channel = "ALL" # Use all channels

# Spatial size and histogram parameters
spatial_size=(16, 16)
hist_bins=16
```


I trained a linear SVM provide by sklearn.svm with default settings: 

```
# Use a linear SVC 
svc = LinearSVC()
# Check the training time for the SVC
t=time.time()
svc.fit(X_train, y_train)
```

It takes 26.57 Seconds to train the classifier. I finally got a test accuracy of 99.2%.

I tried to use the function [GridSearchCV](http://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GridSearchCV.html) to look for the best C parameter, but the accuracy did not improve much.

I also tested the [MLPClassifier](http://scikit-learn.org/stable/modules/generated/sklearn.neural_network.MLPClassifier.html). The accuracy improved a little (99.5%) but the prediction time was slower. Since the performance in accuracy was similar, I decided to use the SVM, that has a faster prediction time (0.00065s for the SVM versus 0.008 for the MLP).  



### Sliding Window Search

We have to deal now with images coming from a front-facing camera on a car. We need to extract from these full-resolution images some sub-regions and check if they contain a car or not. To extract subregions of the image I used a sliding window approach. It is important to minimize the number of subregions used to improve the performance and to avoid looking for cars where they cannot be (for example in the sky).

For each subregion, we need to compute the feature vector and feed it to the classifier. The classifier, a SVM with linear kernel, will predict if there is a car or not in the images.

The function `find_cars` can both extract features and make predictions by computing the HOG transform only once for the entire picture. The HOG is then sub-sampled to get all of its overlaying windows. This function is called three times at different scale: 1, 1.5 and 2. 

Here are some example images:

<img src="./examples/sliding_window.jpg" width="700" alt="" />     


I recorded the positions of positive detections in each frame of the video.  From the positive detections, I created a heat map and then thresholded it to identify vehicle positions.  I then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heat map.  I then assumed each blob corresponded to a vehicle. Finally, I constructed bounding boxes to cover the area of each blob detected.  

Here are six frames and their corresponding heat maps:

<img src="./examples/bboxes_and_heat.png" width="700" alt="" />   

Here is the output of `scipy.ndimage.measurements.label()` on the integrated heatmap from all six frames:
<img src="./examples/labels_map.png" width="700" alt="" />


Here the resulting bounding boxes are drawn onto the last frame in the series:
<img src="./examples/output_bboxes.png" width="700" alt="" />   


---
### Test on images

The hardest challenge of the project is to get rid of the false positives. One thing that really helped was adding a threshold on the decision function of the classifier, which helps to ensure high confidence predictions. In fact, by using the threshold you can ensure that you are only considering high confidence predictions as vehicle detections.

Here is how I computed the prediction value in the function `find_cars`:
```
test_prediction = svc.decision_function(test_features)
```
Finally, I considerd valid only the prediciton with a confidence value bigger than 0.4. 

Another way to reduce the false positive is to apply a threshold on the heatmap:

```
# Apply threshold to help remove false positives
heat = apply_threshold(heat,1)
```
Since I am calling the function `find_cars` three times, at difference scale, I decided to remove the detections with a small value in the heatmap. 

Now let's test the pipeline with some images. You can find the original ones in the folder `test_images`:

<img src="./output_images/test_1.png" width="700" alt="" />   
<img src="./output_images/test_2.png" width="700" alt="" />   
<img src="./output_images/test_3.png" width="700" alt="" />   
<img src="./output_images/test_4.png" width="700" alt="" />   
<img src="./output_images/test_5.png" width="700" alt="" />   
<img src="./output_images/test_6.png" width="700" alt="" />   


The pipeline detects the cars without false positives.

### Video Implementation

Finally, I tested the pipeline on a video stream. In this case, I did not consider each frame individually, in fact, we can take advantage of the previous detections. A deque collection is used to accumulate the detections of the last N frames, allowing to eliminate false positives. The only difference is that the threshold for the heat map is higher. 

This is the result of the detection:

![](./videos/result_hog_svm.gif)   
Click [here](https://youtu.be/yEB0MM7yQbQ) to see the complete video. The pipeline performs reasonably well on the entire project video. The vehicles are identified most of the time with no false positives.

---

# Second solution: SSD (Single Shot MultiBox Detector)

In the last years, Convolutional Neural Networks demonstrated to be very successful for object detection. That's why I was curious to test a deep learning approach to detect vehicles. [Here](http://www.cs.toronto.edu/~urtasun/courses/CSC2541_Winter17/detection.pdf) you can find a nice presentation showing the object detection state-of-the-art.

Here a graph showing the result of the Pascal VOC Challenge in the past years (Slide credit: [Renjie Liao](http://www.cs.toronto.edu/~urtasun/courses/CSC2541/05_2D_detection.pdf)):

<img src="./examples/detection_CNN.png" width="700" alt="" />    

Finally, I decided to use SSD that seems to be one of the best methods, taking into account speed and accuracy (Slide credit: Wei Liu):


<img src="./examples/SSDvsYOLO.png" width="700" alt="" />     

I found in GitHub [this](https://github.com/rykov8/ssd_keras) implementation of the SSD in Keras (see [here](https://handong1587.github.io/deep_learning/2015/10/09/object-detection.html#ssd) for other implementations), and later I discovered on Facebook that another Udacity student, [antorsae](https://github.com/antorsae) had already tested it. 

This is the result that I got with the SSD running on a GTX 1080: 

![](./videos/result_hog_ssd.gif)   
Click [here](https://youtu.be/Ycb-1BTaGis) to see the complete video.


# Object detection with Deep Learning with Dlib 
In the [Dlib library](http://blog.dlib.net/), we can find an implementation of the [max-margin object-detection algorithm (MMOD)](http://blog.dlib.net/2016/10/easily-create-high-quality-object.html), that can work with small amounts of training data. Here you can find a description and the code of the method.

I plan to test this approach after the submission and to update this section with the result.


# Discussion
The current implementation using the HOG and the SVM classifier works quite well for the tested images and videos, but it turned out to be very slow (few frames per second). Even if it could be optimized in  C++ and by parallelizing the search on different scales, a deep learning approach would be probably better for real word applications. The detectors based on CNN are faster, more accurate and more robust. However, it has to be said that this is not a fair comparison since that the SSD is using the GPU.

It would be useful also to use a tracking algorithm when the detection fails (if the detection is fast enough). It would worth a try [Open TDL](http://kahlan.eps.surrey.ac.uk/featurespace/tld/Publications/2011_tpami) or the [correlation_tracker](http://blog.dlib.net/2015/02/dlib-1813-released.html) from the dlib C++ library.

To reduce the false positive for the HOG+SVM solution, it would also be useful to apply hard-negative mining: take the falsely detected patch, add them to the training set with a negative label, and train again the classifier.
