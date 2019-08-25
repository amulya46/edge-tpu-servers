# edge-tpu-servers
This project enables running object and face detection / recognition using the Google edge Tensorflow Processing Unit (TPU) based [development board](https://coral.withgoogle.com/products/dev-board/). This work was done originally as part of the [smart-zoneminder](https://github.com/goruck/smart-zoneminder) project.

The TPU-based object and face detection server, [detect_servers_tpu.py](./detect_servers_tpu.py), runs [TPU-based](https://cloud.google.com/edge-tpu/) Tensorflow Lite inference engines using the [Google Coral](https://coral.withgoogle.com/) Python APIs and employs [zerorpc](http://www.zerorpc.io/) to communicate with clients. The server is run as a Linux service using systemd and it is configured to come up automatically after power-on. 

Images are sent to the object detection as an array of strings, in the following form.
```javascript
['path_to_image_1','path_to_image_2','path_to_image_n']
```

Object detection results are returned as json, an example is shown below.
```json
[ { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00506-capture.jpg",
    "labels": 
     [ { "name": "person",
         "id": 0,
         "score": 0.98046875,
         "box": 
          { "xmin": 898.4868621826172,
            "xmax": 1328.2035827636719,
            "ymax": 944.9342751502991,
            "ymin": 288.86380434036255 } } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00509-capture.jpg",
    "labels": 
     [ { "name": "person",
         "id": 0,
         "score": 0.83984375,
         "box": 
          { "xmin": 1090.408058166504,
            "xmax": 1447.4291610717773,
            "ymax": 846.3531160354614,
            "ymin": 290.5584239959717 } } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00515-capture.jpg",
    "labels": [] } ]
```

The object detection results then in turn can be sent to the face detector, an example of the face detection results returned is shown below.
```json
[ {"image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00506-capture.jpg",
    "labels": 
     [ { "box": 
          { "xmin": 898.4868621826172,
            "xmax": 1328.2035827636719,
            "ymax": 944.9342751502991,
            "ymin": 288.86380434036255 },
         "name": "person",
         "face": "lindo_st_angel",
         "faceProba": 0.88145875,
         "score": 0.98046875,
         "id": 0 } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00509-capture.jpg",
    "labels": 
     [ { "box": 
          { "xmin": 1090.408058166504,
            "xmax": 1447.4291610717773,
            "ymax": 846.3531160354614,
            "ymin": 290.5584239959717 },
         "name": "person",
         "face": null,
         "faceProba": null,
         "score": 0.83984375,
         "id": 0 } ] } ]
```

# Installation
1. Using the [Get Started Guide](https://coral.withgoogle.com/tutorials/devboard/), flash the Dev Board with the latest software image from Google.

2. The Dev Board has a modest 8GB on-board eMMC. You need to insert a MicroSD card (at least 32 GB) into the Dev Board to have enough space to install the software in the next steps. The SD card should be auto-mounted so on power-up and reboots the board can operate unattended. I mounted the SD card at ```/media/mendel```. My corresponding ```/etc/fstab``` entry for the SD card is shown below.  

```bash
#/dev/mmcblk1 which is the sd card
UUID=ff2b8c97-7882-4967-bc94-e41ed07f3b83 /media/mendel ext4 defaults 0 2
```

3. Create a swap file.
```bash
$ cd /media/mendel

# Create a swapfile else you'll run out of memory compiling.
$ sudo mkdir swapfile
# Now let's increase the size of swap file.
$ sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 oflag=append conv=notrunc
# Setup the file as a "swap file".
$ sudo mkswap /swapfile
# Enable swapping.
$ sudo swapon /swapfile
```

4. Install zerorpc.
```bash
# Update repo.
$ sudo apt-get update

# Install dependencies if needed. 
$ sudo apt install python3-dev libffi-dev

$ pip3 install zerorpc

# Test...
$ python3
>>> import zerorpc
>>> 
```

5. Install OpenCV.

NB: The Coral's main CPU is a Quad-core Cortex-A53 which uses an Armv8 microarchitecture and supports single-precision (32-bit, aka AArch32) and double-precision (64-bit, aka AArch64) floating-point data types and arithmetic as defined by the IEEE 754 floating-point standard. OpenCV can use [SIMD (NEON)](https://developer.arm.com/architectures/instruction-sets/simd-isas/neon) instructions to accelerate its computations which is enabled by the cmake options as shown below. For more information about floating point operations from Arm, see [Floating Point](https://developer.arm.com/architectures/instruction-sets/floating-point).

```bash
# Update repo.
$ sudo apt-get update

# Install basic dependencies.
$ sudo apt install python3-dev python3-pip python3-numpy \
build-essential cmake git libgtk2.0-dev pkg-config \
libavcodec-dev libavformat-dev libswscale-dev libtbb2 libtbb-dev \
libjpeg-dev libpng-dev libtiff-dev libdc1394-22-dev protobuf-compiler \
libgflags-dev libgoogle-glog-dev libblas-dev libhdf5-serial-dev \
liblmdb-dev libleveldb-dev liblapack-dev libsnappy-dev libprotobuf-dev \
libopenblas-dev libgtk2.0-dev libboost-dev libboost-all-dev \
libeigen3-dev libatlas-base-dev libne10-10 libne10-dev liblapacke-dev

# Install neon SIMD acceleration dependencies.
$ sudo apt install libneon27-dev libneon27-gnutls-dev

# Download source.
$ cd /media/mendel
$ wget -O opencv.zip https://github.com/opencv/opencv/archive/3.4.5.zip
$ unzip opencv.zip
$ wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/3.4.5.zip
$ unzip opencv_contrib.zip

# Configure OpenCV using cmake.
$ cd /media/mendel/opencv-3.4.5
$ mkdir build
$ cd build
# NB VFPv3 is not used in Armv8-A (AArch66)...don't set -DENABLE_VFPV3=ON.
$ cmake -D CMAKE_BUILD_TYPE=RELEASE -D ENABLE_NEON=ON -D ENABLE_TBB=ON \
-D ENABLE_IPP=ON -D WITH_OPENMP=ON -D WITH_CSTRIPES=OFF -D WITH_OPENCL=ON \
-D BUILD_TESTS=OFF -D INSTALL_PYTHON_EXAMPLES=OFF D BUILD_EXAMPLES=OFF \
-D CMAKE_INSTALL_PREFIX=/usr/local \
-D OPENCV_EXTRA_MODULES_PATH=/media/mendel/opencv_contrib-3.4.5/modules/ ..

# Compile and install. This takes a while...cross compile if impatient
$ make
$ sudo make install

# Rename binding.
$ cd /usr/local/lib/python3.5/dist-packages/cv2/python-3.5
$ sudo mv cv2.cpython-35m-aarch64-linux-gnu.so cv2.so

# Test...
$ python3
>>> import cv2
>>> cv2.__version__
'3.4.5'
>>>
```

6. Install scikit-learn.
```bash
# Install
$ pip3 install scikit-learn

# Test...
$ python3
>>> import sklearn
>>> sklearn.__version__
'0.20.3'
>>>
```

7. Install XGBoost.
```bash
# Install - see https://xgboost.readthedocs.io/en/latest/build.html
# This takes a while...cross compile if impatient.
$ pip3 install xgboost

# Test...
$ python3
>>> import xgboost
>>> xgboost.__version__
'0.90'
>>>
```

8. Install dlib.
```bash
$ cd /media/mendel

# Update repo.
$ sudo apt-get update

# Install dependencies if needed. 
$ sudo apt-get install build-essential cmake

# Clone dlib repo.
$ git clone https://github.com/davisking/dlib.git

# Build and install. 
$ cd dlib
# -O3 enables auto vectorization optimizations
# so that the compiler automatically uses NEON instructions.
$ python3 setup.py install --set DLIB_NO_GUI_SUPPORT=YES \
--set DLIB_USE_CUDA=NO --compiler-flags "-O3"

# Test...
python3
>>> import dlib
>>> dlib.__version__
'19.17.99'
>>>
```

9. Install face_recognition.
```bash
$ pip3 install face_recognition

# Test...
$ python3
>>> import face_recognition
>>> face_recognition.__version__
'1.2.3'
>>>
```

9. Disable and remove swap.
```bash
$ cd /media/mendel
$ sudo swapoff /swapfile
$ sudo rm -i /swapfile
```

10. Create a directory called *tpu-servers* in ```/media/mendel``` on the Coral dev board.

11. Copy *detect_server_tpu.py*, *config.json*, *encode_faces.py*, *train.py*, in this directory to ```/media/mendel/tpu-servers/``` on the Coral dev board.

12. Create the face image data set in ```/media/mendel/tpu-servers/``` needed to train the svm- or xgb-based face classifier per the steps in the [face-det-rec README](https://github.com/goruck/smart-zoneminder/blob/master/face-det-rec/README.md) or just copy the images to this directory if you created them before.

13. Download the face embedding dnn model *nn4.v2.t7* from [OpenFace](https://cmusatyalab.github.io/openface/models-and-accuracies/) to the ```/media/mendel/tpu-servers``` directory. You can skip this step if you aren't going to use OpenCV to generate the facial embeddings. Currently this does not work well since face alignment isn't implemented. The default facial embedding method used in the project currently is dlib (which does face alignment before generating the embeddings). 

14. Download the tpu face recognition dnn model *MobileNet SSD v2 (Faces)* from [Google Coral](https://coral.withgoogle.com/models/) to the ```/media/mendel/tpu-servers``` directory.

15. Download both the *MobileNet SSD v2 (COCO)* tpu object detection dnn model and label file from [Google Coral](https://coral.withgoogle.com/models/) to the ```/media/mendel/tpu-servers``` directory.

*NB: You can instead use transfer learning to train your own models and use them instead of the Google stock models in the steps above, see [TensorFlow Models with Edge TPU Training](https://github.com/goruck/models/tree/edge-tpu).*

16. Run the face encoder program, [encode_faces.py](./encode_faces.py), using the images copied above. This will create a pickle file containing the face embeddings used to train the svm- and xgb-based face classifiers. The program can run tpu-, dlib- and openCV-based face detectors and dlib- and OpenCV-based facial embedders. Its currently configured to use tpu-based face detection and dlib-based facial embeddings which gives the best results vs compute. 

17. Run the face classifier training program, [train.py](./train.py). This will create three pickle files - one for the svm model, one for the xgb model and one for the model labels. The program will optimize the model hyperparameters and then evaluate the model, including thier F1 scores which should be close to 1.0 for good classifier performance. The optimal hyperparameters and model statistics are output after training is completed.

18. Mount a local or remote image store on the Dev Board so the server can find the images and process them. The store should be auto-mounted using ```sshfs``` at startup which is done by an entry in ```/etc/fstab```. Below is an example of a using a remote store at ```lindo@192.168.1.4:/nvr```.
```bash
# Setup sshfs.
$ sudo apt-get install sshfs

# Create mount point.
$ sudo mkdir /mnt/nvr

# Setup SSH keys to enable auto login.
# See https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/
$ mkdir -p $HOME/.ssh
$ chmod 0700 $HOME/.ssh
# Create the key pair
$ ssh-keygen -t rsa
# Install the public key on the server hosting the images.
$ ssh-copy-id -i $HOME/.ssh/id_rsa.pub lindo@192.168.1.4

# Edit /etc/fstab so that the store is automounted. Here's mine.
$ more /etc/fstab
...
lindo@192.168.1.4:/nvr /mnt/nvr fuse.sshfs auto,user,_netdev,reconnect,uid=1000,gid=1000,IdentityFile=/home/mendel/.ssh/id_rsa,idmap=user,allow_other 0 2

# Test mount the zm store. This will happen at boot from now on. 
$ sudo mount -a
$ ls /mnt/nvr
camera-share  lost+found  zoneminder
```

19. Edit the [config.json](./config.json) to suit your installation. The configuration parameters are documented in the detect_server_tpu.py code.

20. Use systemd to run the server as a Linux service. Edit [detect-tpu.service](./detect-tpu.service) to suit your configuration and copy the file to ```/lib/systemd/system/detect-tpu.service```. Then enable and start the service:
```bash
$ sudo systemctl enable detect-tpu.service && sudo systemctl start detect-tpu.service
```

21. Test the entire setup by editing ```detect_servers_test.py``` with paths to test images and running that program.