# SSD-MobileNet

This document has instructions for how to run SSD-MobileNet for the
following modes/precisions:
* [FP32 inference](#fp32-inference-instructions)

Benchmarking instructions and scripts for model training and inference
other precisions are coming later.

## FP32 Inference Instructions

1. Clone the `tensorflow/models` repository with the specified SHA,
since we are using an older version of the models repo for
SSD-MobileNet.

```
$ git clone https://github.com/tensorflow/models.git
$ cd models
$ git checkout 20da786b078c85af57a4c88904f7889139739ab0
$ git clone https://github.com/cocodataset/cocoapi.git
```

The TensorFlow models repo will be used for running inference as well as
converting the coco dataset to the TF records format.

2. Follow the TensorFlow models object detection
[installation instructions](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/installation.md#installation)
to get your environment setup with the required dependencies.

3.  Download the 2017 validation
[COCO dataset](http://cocodataset.org/#home) and annotations:

```
$ mkdir val
$ cd val
$ wget http://images.cocodataset.org/zips/val2017.zip
$ unzip val2017.zip
$ cd ..

$ mkdir annotations
$ cd annotations
$ wget http://images.cocodataset.org/annotations/annotations_trainval2017.zip
$ unzip annotations_trainval2017.zip
$ cd ..
```

Since we are only using the validation dataset in this example, we will
create an empty directory and empty annotations json file to pass as the
train and test directories in the next step.

```
$ mkdir empty_dir

$ cd annotations
$ echo "{ \"images\": {}, \"categories\": {}}" > empty.json
$ cd ..
```

4. Now that you have the raw COCO dataset, we need to convert it to the
TF records format in order to use it with the inference script.  We will
do this by running the `create_coco_tf_record.py` file in the TensorFlow
models repo.

Follow the steps below to navigate to the proper directory and point the
script to the raw COCO dataset files that you have downloaded in step 2.
The `--output_dir` is the location where the TF record files will be
located after the script has completed.

```

# We are going to use an older version of the conversion script to checkout the git commit
$ cd models
$ git checkout 7a9934df2afdf95be9405b4e9f1f2480d748dc40

$ cd research/object_detection/dataset_tools/
$ python create_coco_tf_record.py --logtostderr \
      --train_image_dir="/home/myuser/coco/empty_dir" \
      --val_image_dir="/home/myuser/coco/val/val2017" \
      --test_image_dir="/home/myuser/coco/empty_dir" \
      --train_annotations_file="/home/myuser/coco/annotations/empty.json" \
      --val_annotations_file="/home/myuser/coco/annotations/instances_val2017.json" \
      --testdev_annotations_file="/home/myuser/coco/annotations/empty.json" \
      --output_dir="/home/myuser/coco/output"

$ ll /home/myuser/coco/output
total 1598276
-rw-rw-r--. 1 myuser myuser         0 Nov  2 21:46 coco_testdev.record
-rw-rw-r--. 1 myuser myuser         0 Nov  2 21:46 coco_train.record
-rw-rw-r--. 1 myuser myuser 818336740 Nov  2 21:46 coco_val.record

# Go back to the main models directory and checkout the SHA that we are using for SSD-MobileNet
$ cd /home/myuser/models
$ git checkout 20da786b078c85af57a4c88904f7889139739ab0
```

The `coco_val.record` file is what we will use in this inference example.

5. Download and extract the pre-trained SSD-MobileNet model from the
[TensorFlow detection model zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md#coco-trained-models).
The downloaded .tar file includes a `frozen_inference_graph.pb` which we
will be using when running inference.

```
$ wget http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_2018_01_28.tar.gz

$ tar -xvf ssd_mobilenet_v1_coco_2018_01_28.tar.gz
ssd_mobilenet_v1_coco_2018_01_28/
ssd_mobilenet_v1_coco_2018_01_28/model.ckpt.index
ssd_mobilenet_v1_coco_2018_01_28/checkpoint
ssd_mobilenet_v1_coco_2018_01_28/pipeline.config
ssd_mobilenet_v1_coco_2018_01_28/model.ckpt.data-00000-of-00001
ssd_mobilenet_v1_coco_2018_01_28/model.ckpt.meta
ssd_mobilenet_v1_coco_2018_01_28/saved_model/
ssd_mobilenet_v1_coco_2018_01_28/saved_model/saved_model.pb
ssd_mobilenet_v1_coco_2018_01_28/saved_model/variables/
ssd_mobilenet_v1_coco_2018_01_28/frozen_inference_graph.pb

$ cd ssd_mobilenet_v1_coco_2018_01_28

$ ll
total 58132
-rw-r--r--. 1 myuser myuser       77 Feb  1  2018 checkpoint
-rw-r--r--. 1 myuser myuser 29103956 Feb  1  2018 frozen_inference_graph.pb
-rw-r--r--. 1 myuser myuser 27380740 Feb  1  2018 model.ckpt.data-00000-of-00001
-rw-r--r--. 1 myuser myuser     8937 Feb  1  2018 model.ckpt.index
-rw-r--r--. 1 myuser myuser  3006546 Feb  1  2018 model.ckpt.meta
-rw-r--r--. 1 myuser myuser     4138 Feb  1  2018 pipeline.config
drwxr-sr-x. 3 myuser myuser     4096 Feb  1  2018 saved_model
```

6. Clone the [intelai/models](https://github.com/intelai/models) repo.
This repo has the launch script for running benchmarking, which we will
use in the next step.

```
$ git clone https://github.com/IntelAI/models.git
Cloning into 'models'...
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 11 (delta 3), reused 4 (delta 0), pack-reused 0
Receiving objects: 100% (11/11), done.
Resolving deltas: 100% (3/3), done.
```

7. Next, navigate to the `benchmarks` directory of the
[intelai/models](https://github.com/intelai/models) repo that was just
cloned in the previous step. SSD-MobileNet can be run for benchmarking
throughput and latency, or testing accuracy. Note that we are running
SSD-MobileNet with a TensorFlow 1.12 docker image.

To benchmarking for throughput and latency, use the following command,
but replace in your path to the unzipped coco dataset images from step 3
for the `--dataset-location`, the path to the frozen graph that you
downloaded in step 5 as the `--in-graph`, and use the `--benchmark-only`
flag:

```
$ cd /home/myuser/models/benchmarks

$ python launch_benchmark.py \
    --data-location /home/myuser/coco/output/coco_val.record \
    --in-graph /home/myuser/ssd_mobilenet_v1_coco_2018_01_28/frozen_inference_graph.pb \
    --model-source-dir /home/myuser/tensorflow/models \
    --model-name ssd-mobilenet \
    --framework tensorflow \
    --precision fp32 \
    --mode inference \
    --socket-id 0 \
    --docker-image intelaipg/intel-optimized-tensorflow:1.12.0-mkl \
    --benchmark-only
```

To test accuracy, use the following command but replace in your path to
the tf record file that you generated in step 4 for the `--data-location`,
the path to the frozen graph that you downloaded in step 5 as the
`--in-graph`, and use the `--accuracy-only` flag:

```
$ python launch_benchmark.py \
    --data-location /home/myuser/coco/output/coco_val.record \
    --in-graph /home/myuser/ssd_mobilenet_v1_coco_2018_01_28/frozen_inference_graph.pb \
    --model-source-dir /home/myuser/tensorflow/models \
    --model-name ssd-mobilenet \
    --framework tensorflow \
    --precision fp32 \
    --mode inference \
    --socket-id 0 \
    --docker-image intelaipg/intel-optimized-tensorflow:1.12.0-mkl \
    --accuracy-only
```

8. The log file is saved to the value of `--output-dir`.

Below is a sample log file tail when running benchmarking:

```
INFO:tensorflow:Processed 5001 images... moving average latency 37 ms
INFO:tensorflow:Finished processing records
Latency: min = 33.8, max = 6635.9, mean= 38.4, median = 37.2
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Ran inference with batch size -1
Log location outside container: {--output-dir value}/benchmark_ssd-mobilenet_inference_fp32_20190130_225108.log
```

Below is a sample log file tail when testing accuracy:

```
 Average Precision  (AP) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.231
 Average Precision  (AP) @[ IoU=0.50      | area=   all | maxDets=100 ] = 0.349
 Average Precision  (AP) @[ IoU=0.75      | area=   all | maxDets=100 ] = 0.254
 Average Precision  (AP) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.231
 Average Precision  (AP) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = -1.000
 Average Precision  (AP) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = -1.000
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=  1 ] = 0.209
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets= 10 ] = 0.264
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.264
 Average Recall     (AR) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.264
 Average Recall     (AR) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = -1.000
 Average Recall     (AR) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = -1.000
lscpu_path_cmd = command -v lscpu
lscpu located here: /usr/bin/lscpu
Ran inference with batch size -1
Log location outside container: {--output-dir value}/benchmark_ssd-mobilenet_inference_fp32_20190123_225145.log
```
