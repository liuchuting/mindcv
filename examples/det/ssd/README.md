# SSD Based on MindCV Backbones

> [SSD: Single Shot MultiBox Detector](https://arxiv.org/abs/1512.02325)


## Introduction

SSD is an single-staged object detector. It discretizes the output space of bounding boxes into a set of default boxes over different aspect ratios and scales per feature map location, and combines predictions from multi-scale feature maps to detect objects with various sizes. At prediction time, SSD generates scores for the presence of each object category in each default box and produces adjustments to the box to better match the object shape.

<p align="center">
  <img src="https://github.com/DexterJZ/mindcv/assets/16130861/50bc9627-c71c-4b1a-9de4-9e6040a43279" width=800 />
</p>
<p align="center">
  <em>Figure 1. Architecture of SSD [<a href="#references">1</a>] </em>
</p>

In this example, by leveraging [the multi-scale feature extraction of MindCV](https://github.com/mindspore-lab/mindcv/blob/main/docs/en/how_to_guides/feature_extraction.md), we demonstrate that using backbones from MindCV much simplifies the implementation of SSD.

## Requirements
| mindspore | ascend driver |  firmware   | cann toolkit/kernel |
| :-------: | :-----------: | :---------: | :-----------------: |
|   2.5.0   |   24.1.0      | 7.5.0.3.220 |     8.0.0.beta1     |

## Configurations

Here, we provide three configurations of SSD.
* Using [MobileNetV2](https://github.com/mindspore-lab/mindcv/tree/main/configs/mobilenetv2) as the backbone and the original detector described in the paper.
* Using [ResNet50](https://github.com/mindspore-lab/mindcv/tree/main/configs/resnet) as the backbone with a FPN and a shared-weight-based detector.
* Using [MobileNetV3](https://github.com/mindspore-lab/mindcv/tree/main/configs/mobilenetv3) as the backbone and the original detector described in the paper.

## Dataset

We train and test SSD using [COCO 2017 Dataset](https://cocodataset.org/#download). The dataset contains
* 118000 images about 18 GB for training, and
* 5000 images about 1 GB for testing.

## Quick Start

### Preparation

1. Clone MindCV repository by running
```
git clone https://github.com/mindspore-lab/mindcv.git
```

2. Install dependencies as shown [here](https://mindspore-lab.github.io/mindcv/installation/).

3. Download [COCO 2017 Dataset](https://cocodataset.org/#download), prepare the dataset as follows.
```
.
└─cocodataset
  ├─annotations
    ├─instance_train2017.json
    └─instance_val2017.json
  ├─val2017
  └─train2017
```
Run the following commands to preprocess the dataset and convert it to [MindRecord format](https://www.mindspore.cn/docs/zh-CN/master/api_python/mindspore.mindrecord.html) for reducing preprocessing time during training and testing.
```
cd mindcv  # change directory to the root of MindCV repository
python examples/det/ssd/create_data.py coco --data_path [root of COCO 2017 Dataset] --out_path [directory for storing MindRecord files]
```
Specify the path of the preprocessed dataset at keyword `data_dir` in the config file.

4. Download the pretrained backbone weights from the table below, and specify the path to the backbone weights at keyword `backbone_ckpt_path` in the config file.


|    MobileNetV2   |     ResNet50     |    MobileNetV3   |
|:----------------:|:----------------:|:----------------:|
| [backbone weights](https://download.mindspore.cn/toolkits/mindcv/mobilenet/mobilenetv2/mobilenet_v2_100-d5532038.ckpt) | [backbone weights](https://download.mindspore.cn/toolkits/mindcv/resnet/resnet50-e0733ab8.ckpt) | [backbone weights](https://download.mindspore.cn/toolkits/mindcv/mobilenet/mobilenetv3/mobilenet_v3_large_100-1279ad5f.ckpt) |



### Train

It is highly recommended to use **distributed training** for this SSD implementation.

For distributed training using **`msrun`**, simply run
```
cd mindcv  # change directory to the root of MindCV repository
msrun --bind_core=True --worker_num [# of devices] python examples/det/ssd/train.py --config [the path to the config file]
```
For example, if train SSD distributively with the `MobileNetV2` configuration on 8 devices, run
```
cd mindcv  # change directory to the root of MindCV repository
msrun --bind_core=True --worker_num 8 python examples/det/ssd/train.py --config examples/det/ssd/ssd_mobilenetv2.yaml
```

For distributed training with [Ascend rank table](https://github.com/mindspore-lab/mindocr/blob/main/docs/en/tutorials/distribute_train.md#12-configure-rank_table_file-for-training), configure `ascend8p.sh` as follows
```
#!/bin/bash
export DEVICE_NUM=8
export RANK_SIZE=8
export RANK_TABLE_FILE="./hccl_8p_01234567_127.0.0.1.json"

for ((i = 0; i < ${DEVICE_NUM}; i++)); do
    export DEVICE_ID=$i
    export RANK_ID=$i
    echo "Launching rank: ${RANK_ID}, device: ${DEVICE_ID}"
    if [ $i -eq 0 ]; then
      echo 'i am 0'
      python examples/det/ssd/train.py --config [the path to the config file] &> ./train.log &
    else
      echo 'not 0'
      python -u examples/det/ssd/train.py --config [the path to the config file] &> /dev/null &
    fi
done
```
and start training by running
```
cd mindcv  # change directory to the root of MindCV repository
bash ascend8p.sh
```

For single-device training, please run
```
cd mindcv  # change directory to the root of MindCV repository
python examples/det/ssd/train.py --config [the path to the config file]
```

### Test

For testing the trained model, first specify the path to the model checkpoint at keyword `ckpt_path` in the config file, then run
```
cd mindcv  # change directory to the root of MindCV repository
python examples/det/ssd/eval.py --config [the path to the config file]
```
For example, for testing SSD with the `MobileNetV2` configuration, run
```
cd mindcv  # change directory to the root of MindCV repository
python examples/det/ssd/eval.py --config examples/det/ssd/ssd_mobilenetv2.yaml
```


## Performance

Experiments are tested on Ascend Atlas 800T A2 machines with mindspore 2.5.0 graph mode.

| model name       | params(M) | cards | batch size | resolution | jit level | graph compile | ms/step | img/s   | mAP   | recipe                                                                                           | weight                                                                                      |
| ---------------- | --------- | ----- | ---------- | ---------- | --------- |---------------|---------|---------|-------| ------------------------------------------------------------------------------------------------ |---------------------------------------------------------------------------------------------|
| ssd_mobilenetv2  | 4.45      | 8     | 32         | 300x300    | O2        | 347s          | 69.63   | 3676.58 | 23.39 | [yaml](https://github.com/mindspore-lab/mindcv/blob/main/examples/det/ssd/ssd_mobilenetv2.yaml)  | [weights](https://download.mindspore.cn/toolkits/mindcv/ssd/ssd_mobilenetv2-dc7035ec-910v2.ckpt)  |

## References

[1] Liu, W., Anguelov, D., Erhan, D., Szegedy, C., Reed, S., Fu, C. Y., & Berg, A. C. (2016). SSD: Single Shot Multibox Detector. In Computer Vision–ECCV 2016: 14th European Conference, Amsterdam, The Netherlands, October 11–14, 2016, Proceedings, Part I 14 (pp. 21-37). Springer International Publishing.
