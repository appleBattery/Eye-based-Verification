# Eye-based-Verification

> Proposed contact-less attendance system for organizations by verifying identity using iris and periocular region.

## Abstract:

The proposed method to verify identity for attendance would be to capture images at a distance using Near Infrared Imaging. Before capturing image, user must scan id card barcode to match with database entry. Iris must be clearly visible and image must be high resolution if taken from a distance (so that there is atleast 224x224 pixels in the eye region.

By use of *haarcascade_eye.xml* and *haarcascade_frontalface_default.xml* in **OpenCV** both eyes can be detected and extracted from the captured image. Transfer learning was applied by using an ImageNet trained **ResNet50** Deep Learning Network (easily deployed using [torchvision.models](https://pytorch.org/docs/stable/torchvision/models.html)) for feature extraction and adding a few linear layers which are then used for verification of the input eye image with the image already stored in the database. By adding a more weight in the cost to false positives, precision of the system was improved.

Further, liveness detection can be incorporated into the system to make it more robust to spoofing.

## Approach:

Transfer Learning was applied in **Pytorch** by freezing all the layers of pre-trained ResNet50 (thus, fine tuning was not performed) and removing the last layer. Two architectures were trained by using either 4 fully connected layers (*model0_100.pth* and *model1_100.pth*) or 3 fully connected layers after ResNet50 (*model2_100.pth* and *model3 reg 6e-5.pth*). In both the cases, features are extracted for both input image and database image using pre-trained network. The outputs are merged and passed through two fully connected layers and finally passed through a logistic regression function for binary classification (*whether the images are from the same user or not*).

![Scheme 1](resources/classifier%201.JPG?raw=true)
<p align="center"><b>Scheme 1</b></p>

![Scheme 2](resources/classifier%202.JPG?raw=true)
<p align="center"><b>Scheme 2</b></p>

### The following datasets are relevant to the problem at hand:

* [**IIT Delhi dataset**](https://www4.comp.polyu.edu.hk/~csajaykr/IITD/Database_Iris.htm) - <ins> NIR images</ins>. 224 subjects, but excluded subjects 1-13, 27,55,65. Highly detailed images of iris but lacks any peripheral features which is not expected to be the case in deployment (unless images are captured from very small distance). To be augmented with another dataset which contains images captured from distance. It should be useful to teach model to identify the iris as a major differentiating factor. 

* **Multimedia Universtiy (MMU) dataset** - <ins> NIR images</ins>. MMU.2 iris database consists of 995 iris images. The iris images are collected using Panasonic BM-ET100US Authenticam and its operating range is even farther with a distance of 47-53 cm away from the user. These iris images are contributed by 100 volunteers with different age and nationality. They come from Asia, Middle East, Africa and Europe. Each of them contributes 5 iris images for each eye. Used for training. Slightly noisy dataset as iris features are not very clear and varying image conditions.

* [**CASIA Iris v1 to v4**](http://biometrics.idealtest.org/dbDetailForUser.do?id=4) - <ins> NIR Images</ins>. Contains both images at a distance and under various lighting conditions. Although not used for training, this would lead to significant improvement in model performance if used together with the above two datasets. One should note that subjects are predominantly east asian, so one must check for bias.

* [**The Hong Kong Polytechnic University Cross-Spectral Iris Images Database**](https://www4.comp.polyu.edu.hk/~csajaykr/polyuiris.htm) - <ins> both NIR and Visible Images</ins>. This database of iris images has been acquired under simultaneous bi-spectral imaging, from both right and left eyes. This database consists of total 12,540 iris images which are acquired with 15 instances from 209 different subjects.

* [**UBIRIS v2**](http://iris.di.ubi.pt/ubiris2.html) - <ins> Visible Images</ins>. Noisy Dataset with various angles. Best reflects real conditions of images captured in visible wavelength from a distance. Weights from a model trained to identify iris in NIR might be initialized and then training can be done on a combined UBIRIS and VISOB dataset.

* [**VISOB**](https://sce.umkc.edu/research-sites/cibit/visob_v1.html) - <ins> Visible Images</ins>. Noisy images but very large dataset of eye images captured through smartphones.

> Datasets are not included in repo as they are not available in general public domain and access is given on case to case basis (refer the links above for procedure on applying for access to the datasets).

### For verifying two images if they are same or not there are essentially two approaches:

* Define Anchor, Positive and Negative sets of images from dataset and optimize triplet loss function. But one must pick these from dataset only for the images which are harder to train and are close in distance.

* Create sets of positive and negative images from the datasets. This was used for training. Although large combinations of negative sets can be created from the datasets, equal number of postive and negative training sets were chosen from the dataset for training. A *BCEWithLogitsLoss* function was optimized with pos_weight parameter<1 to increase specificity. Suggested approach is to instead create larger number of negative sets of images from the dataset itself (possibly 3-4 times the number of positive examples).

### Various models trained -

* **Model0_100 epochs** (Scheme 1) - Training was done on the IITD dataset (8320 training sets). Equal number of positive image sets and negative sets were trained using transfer learning on a resNet50 network pre-trained on ImageNet. Size of network parameters is about 100Mb.
  - Similarity function approach was used for training. Outputs from the pre-trained Siamese network were compared using 3 linear layers.
  - Accuracy of 95% was obtained on the test set (comprising images on the same dataset). Training was done for a total of 100 epochs.
  - Optimizer = SGD; Learning Rate = 0.01; 
  - 70% accuracy on the MMU dataset

* **Model1_100 epochs** (Scheme 1) - Training was done on the IITD dataset + 0.7 \* MMU dataset (11,106 training sets total). 
  - Optimizer = SGD; Learning rate - 0.01;
  - Training accuracy 100% and Validation Accuracy is 89% which indicates overfitting of the model. 
  - Batch size of 256 used for training.
  
* **Model2_100 epochs** (Scheme 2)- Training was done on the IITD dataset + 0.7 \* MMU dataset (11,106 training sets total). 
  - Weight decay added with parameter 1e-5. Increased learning rate to 0.02 with decay.
  - Adding regularization in model2 to reduce overfitting and removing one layer from the model.
  - Parameter, pos_weight = 0.9 in the cost function for increasing the specificity.
  
* **Model2_60 epochs** (Scheme 2)- Training was done on the IITD dataset + 0.7 \* MMU dataset (11,106 training sets total). Validation on 0.3 \* MMU dataset.
  - Slightly increased weight decay to 3e-5. Learning rate increased to 0.04.
  - Pos_weight in cost fn further decreased to 0.5 to increase specificity at the cost of sensitivity.
  - Training accuracy **99.78%** and validation accuracy **91.54%** at 60 epochs by early stopping.
  - Results of model2 are the best on the following datasets.
  
* **Model3 (with various weight decay parameters)** (Scheme 2)- Training was done on the IITD dataset + 0.7 \* MMU dataset (11,106 training sets total). Validation on 0.3 \* MMU dataset
  - Weight_decay of 6e-5 and 4e-5 used. 
  - Learning rate of 0.075 with decay and pos_weight = 0.3 and 0.4 respectively.
  - Accuracy after 85 epochs: training accuracy 97% and 84% validation for 6e-5
  - training accuracy 100% and 88% validation for 4e-5 weight decay
  
> **Note** - most of the code was implemented using google colab GPU's.

![Accuracy Curves](resources/accuracy%20curves.png?raw=true)
<p align="center"><b>Model2_65 epochs: Accuracy Curves (Orange - Validation accuracy; Blue - Training Accuracy)</b></p>

![Loss Curve](resources/loss%20curve.png?raw=true)
<p align="center"><b>Model2_65 epochs: Loss Curve</b></p>

## Code Present in Repository:

* **Extract eyes from Frontal Faces using OpenCV**: Simple Demonstration of Extracting eyes from the frontal face images

* **Save iitd database to h5.ipynb**: Initial training of model0 and model1 involved complete loading of IITD dataset to memory and then making an array containing all possible image sets (warning: bad idea obviously and gets pretty close to overloading a 12GB RAM). For this purpose, this code was implemented to store iitd dataset as an h5 file. Later dataloader class was implemented to read images from disk (included as part of the training code).

* **Training on IITD dataset.ipynb**: Training of model0 on IITD dataset.

* **Testing model1 (Scheme 1) on MMU2 dataloader.ipynb**: Dataloader class for MMU2 dataset created and tested on perviously trained model 1.

* **Train classifier on MMU2 and IITD ver 1.ipynb (model 1, 2 and 3)**: Training and testing Code for the mentioned datasets. Master training file.

## Learnings & Future Steps:

* Use of lighter and more efficient Convolution Networks trained on ImageNet to be explored for deployment.

* Positive image sets constructed from IITD and MMU are very similar to each other (probably because images were collected through a burst of shots). This is very unlikely to be the real use case scenario! Other datasets like Poly U Cross or UBIRIS dataset to be used to make sure that there are significant differences between images of the same subject for training.

* Image augmentation not implemented. This can be done.

* Only a few Negative image combinations were considered, equal to the maximum number of possible positive image sets of the same subject. Suggest use of many more negative sets which will also ensure a high precision (specificity).

* After training, another exercise can be to verify which regions of the image are being used for classification.

* Further improvements to the model can be done by training on the NIR PolyUCross dataset and also the CASIA Iris datasets. Try to incorporate differences in positive images i.e. images captured from the same subject in different settings. Enlargement of pupils, and other changes in conditions must be captured in training. Also care to be taken with inclusion/exclusion of periocular features in training set. With increase in number of datasets, enhancing architecture might be considered.

* Once a robust NIR detection model is achieved, another model can be trained on Visible image datasets using the weights from the NIR detection model to expand the scope and practicality of the project.

## References:

* [DeepIris paper](https://arxiv.org/abs/1907.09380): Proposed an iris recognition framework based on transfer learning approach. They fine-tune a pre-trained convolutional neural network (ResNet 50 - trained on ImageNet), on a popular iris recognition dataset (IIT Delhi iris dataset)

* [Iris Recognition With Off-the-Shelf CNN Features](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=8219390): Performed iris segmentation but suggests middle layers of such deep learning architectures might give better performance since iris does not contain large complexity as was in the case of imageNet trained networks.

* [An end to end Iris Recognition system using Pytorch](https://github.com/thuyngch/Iris-Recognition-PyTorch): Torch implementation on efficient net using MMU and CASIA1 datasets.
