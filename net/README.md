# Online Action Recognition

We built an online action recognition system that recognizes different actions based on UCF-101 Dataset published in 2013. You can refer to it here: https://www.crcv.ucf.edu/data/UCF101.php

Online recognition means that we receive stream of frames from a camera and determine the recognized action directly. That is different from the offline approach which outputs the action after processing a captured video.

Our system is composed of many parts, and is mainly built on top of Temporal Segment Networks.

Temporal Segment Networks for Action Recognition in Videos, Limin Wang, Yuanjun Xiong, Zhe Wang, Yu Qiao, Dahua Lin, Xiaoou Tang, and Luc Van Gool, TPAMI, 2018.
[[Arxiv Preprint](https://arxiv.org/abs/1705.02953)]

## Prerequisites

* The code is written in Python 3.6 with the Anaconda Python Distribution.
* GPUs are required for training.
* Google Colab

We trained and tested our model on two Tesla K-80 GPUs (BA-HPC in Bibliotheca Alexandrina). You can finish training on only one GPU but testing will get you stuck due to the lack of cuda memory as you feed the network one video at a time. 

We ran and debugged all our codes on Google Colab, and once the model is ready for training, we switched to BA_HPC.

## Steps to get the model ready for training (Google Colab)

1. Change Runtime type to choose GPU.
2. Check if the GPU is operating by running the following command:
```
!df ~ --block-size=G
```
if the overall size is more than 300G, you are ready to go.

3. run the following commands to download and extract UCF-101 Dataset:
```
%%sh
# at /content/two-stream-action-recognition
wget http://ftp.tugraz.at/pub/feichtenhofer/tsfusion/data/ucf101_jpegs_256.zip.001 &
wget http://ftp.tugraz.at/pub/feichtenhofer/tsfusion/data/ucf101_jpegs_256.zip.002 &
wget http://ftp.tugraz.at/pub/feichtenhofer/tsfusion/data/ucf101_jpegs_256.zip.003 &
```

```
%%sh
# at /content/two-stream-action-recognition
cat ucf101_jpegs_256.zip.00* > ucf101_jpegs_256.zip &
```

```
# at /content/two-stream-action-recognition
!rm ucf101_jpegs_256.zip.00*
```

```
%%sh
# at /content/two-stream-action-recognition
unzip ucf101_jpegs_256.zip >> zip1.out &
```

```
# at /content/two-stream-action-recognition
!rm ucf101_jpegs_256.zip
```

```
# view the data jpegs_256 should be 33G
!du -sh --human-readable *
```

This should take a while. You can now see the dataset files are ready in the Files section on Google Colab.

4. Use git to clone this repository:
```
!pip install -q xlrd
!git clone https://github.com/The-FaZe/real-time-action-recognition.git
```

## Training

To train our model, use main.py script by running the following:

For RGB stream:
```
%%shell

python3 /content/real-time-action-recognition/main.py ucf101 RGB \
<ucf101_rgb_train_list> <ucf101_rgb_val_list> \
   --arch  BNInception --num_segments 3 \
   --gd 20 --lr 0.001 --lr_steps 30 60 --epochs 80 \
   -b 128 -j 8 --dropout 0.8 \
   --snapshot_pref <weights_file_name>
```

Hyperparameters tuning was done by the authors of TSN paper and we did a little bit of modifications to suit our GPU capacity. 
You can find what each symbol stands for in the scripting file "parser_commands.py".

For RGB Difference stream:
```
%%shell

python3 /content/real-time-action-recognition/main.py ucf101 RGBDiff \
<ucf101_rgb_train_list> <ucf101_rgb_val_list> \
  --arch  BNInception --num_segments 3 \
  --gd 40 --lr 0.001 --lr_steps 80 160 --epochs 180 \
  -b 128 -j 8 --dropout 0.8 \
  --gpus 0 1 \
  --KinWeights <kinetics_weights_directory> \
  --snapshot_pref <weights_file_name>
```

Parameters between <...> should be specified by yourself. 

b & j should be tuned according to your capacity of GPUs and memory size. They should be reduced to suit the GPU in Google Colab.

Note: You can fully train the RGB stream but your GPU memory will failt when training on RGB difference stream as you feed the network with 3 times frames of the RGB stream, so you might need two GPUs not only one.

KinWeights refer to the pretrained Kinetcis dataset. In the original work, the model is pretrained on ImageNet dataset to overcome overfitting. Adding Kinetics pretrained weights instead of ImageNet increases accuracy because Kinetics is a large dataset includes 600 different classes for actions.


## Testing

RGB stream:

```
python3 -u test_models.py ucf101 RGB <ucf101_rgb_test_list> <weights_directory> \
        --arch BNInception --save_scores <score_file_name> \
        --classInd_file <class_index_file> \
	      --gpus 0 1 -j 2
```

RGBDiff Stream:
```
python3 -u test_models.py ucf101 RGBDiff <ucf101_rgb_test_list> <weights_directory> \
   --arch BNInception --save_scores <score_file_name> \
   --classInd_file <class_index_file> \ 
   --gpus 0 1 -j 2
```

If you have only one GPU, you can remove gpus & j parameters.





