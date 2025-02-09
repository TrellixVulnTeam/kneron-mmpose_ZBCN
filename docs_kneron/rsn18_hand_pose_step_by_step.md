# Step 0. Environment

## Prerequisites

- Python 3.6+
- PyTorch 1.3+ (We recommend you installing PyTorch using Conda following the [Official PyTorch Installation Instruction](https://pytorch.org/))
- (Optional) CUDA 9.2+ (If you installed PyTorch with cuda using Conda following the [Official PyTorch Installation Instruction](https://pytorch.org/), you can skip CUDA installation)
- (Optional, used to build from source) GCC 5+
- [mmcv-full](https://mmcv.readthedocs.io/en/latest/#installation) (Note: not `mmcv`!)

**Note:** You need to run `pip uninstall mmcv` first if you have `mmcv` installed.
If mmcv and mmcv-full are both installed, there will be `ModuleNotFoundError`.

### Install kneron-mmpose

1. We recommend you installing mmcv-full with pip:

  ```shell
  pip install mmcv-full -f https://download.openmmlab.com/mmcv/dist/{cu_version}/{torch_version}/index.html
  ```
  Please replace {cu_version} and {torch_version} in the url to your desired one. For example, to install the latest mmcv-full with CUDA 11.0 and PyTorch 1.7.0, use the following command:
  ```shell
  pip install mmcv-full -f https://download.openmmlab.com/mmcv/dist/cu110/torch1.7.0/index.html
  ```    
  See [here](https://github.com/open-mmlab/mmcv#installation) for different versions of MMCV compatible to different PyTorch and CUDA versions.

2. Clone the Kneron-version kneron-mmpose repository.

  ```bash
  git clone https://github.com/kneron/kneron-mmpose.git
  cd kneron-mmpose
  ```

3. Install required python packages for building and installing kneron-mmpose.

    ```shell
    pip install -r requirements/build.txt
    pip install -v -e .  # or "python setup.py develop"
    ```

# Step 1: Training models on standard datasets

MMPose provides hundreds of existing and existing pose models in [Model Zoo](https://mmpose.readthedocs.io/en/latest/modelzoo.html), and supports several standard datasets like COCO, MPII, FREIHAND, etc. This note demonstrates how to perform common object detection tasks with these existing models and standard datasets, including:

- Use existing models to inference on given images.
- Test existing models on standard datasets.
- Train models on standard datasets.

## Train models on standard datasets

MMPose also provides out-of-the-box tools for training pose models.
This section will show how to train models under [configs](https://github.com/kneron/kneron-mmpose/blob/main/configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py) on standard datasets i.e. FREIHAND.

**Important**: You might need to modify the [config file](https://github.com/kneron/kneron-mmpose/blob/main/configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py) according your GPUs resource (such as "samples_per_gpu","workers_per_gpu" ...etc due to your GPUs RAM limitation).
The default learning rate in config files is for 1 GPUs and 64 img/gpu (batch size = 64\*1 = 64).

### Step 1-1: Prepare datasets

Public datasets such as [FREIHAND](https://lmb.informatik.uni-freiburg.de/projects/freihand/).
We suggest that you download and extract the dataset to somewhere outside the project directory and symlink (`ln`) the dataset root to `kneron-mmpose/data`(`ln -s realpath/to/freihand kneron-mmpose/data/freihand`), as shown below:

```plain
kneron-mmpose
├── mmpose
├── configs
├── tools
├── data
│   ├── freihand
│   │   ├── training
│   │   ├── annotations
│   │   ├── evaluation
│   ├── rhd
│   │   ├── training
│   │   ├── annotations
│   │   ├── evaluation
...
```

It's recommended to *symlink* the dataset folder to kneron-mmpose folder. However, if you place your dataset folder at different place and do not want to symlink, you have to change the corresponding paths in config files (absolute path is recommended).

### Step 1-2: Training Example with RSN18:

[RSN: Range Sparse Net for Efficient, Accurate LiDAR 3D Object Detection](https://arxiv.org/abs/2106.13365)

We only need the configuration file (which is provided in `configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py`) to train pose model:
```python
python tools/train.py configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py
```
* (Note) you might need to create a folder name 'work_dir' in kneron-mmpose root folder because we set 'work_dir' as default folder in 'pose_config_name.py'
* (Note 2) The whole training process might take several days, depending on your computational resource (number of GPUs, etc). If you just want to take a quick look at the deployment flow, we suggest that you download our trained model so you can skip the training process:
```bash
mkdir work_dirs
cd work_dirs
wget https://github.com/kneron/Model_Zoo/blob/main/mmpose/rsn18_freihand/latest.zip
unzip latest.zip
cd ..
```
* (Note 3) This is a "training from scratch" tutorial, which might need lots of time and gpu resource. If you want to train a model on your custom dataset, it is recommended that you read [finetune.md](https://github.com/open-mmlab/mmpose/blob/master/docs/en/tutorials/1_finetune.md), [customize_dataset.md](https://github.com/open-mmlab/mmpose/blob/master/docs/en/tutorials/2_new_dataset.md).

# Step 2: Test trained model
'tools/test_kneron.py' is a script that generates inference results from test set with our pytorch model and evaluates the results to see if our pytorch model is well trained (if `--eval` argument is given). Note that it's always good to evluate our pytorch model before deploying it.

```python
python toos/test_kneron.py \
    configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py \
    work_dirs/latest.pth \
    --eval AUC EPE PCK \
```
* `configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py` is your pose training config
* `work_dirs/latest.pth` is your trained pose model

The expected result of the command above will be something similar to the following text (the numbers may slightly differ):
```
AUC: 0.8579125542518
EPE: 3.5918244972577695
PCK: 0.9885881741008469
```

# Step 3: Export ONNX and Verify
### Step 3-1: Export ONNX:
'tools/deployment/pytorch2onnx_kneron.py' is a script provided by MMPose to help user to convert our trained pth model to onnx:
```python
python tools/deployment/pytorch2onnx_kneron.py \
    configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py \
    work_dirs/rsn18_freihand2d_224x224/latest.pth \
    --output-file work_dirs/rsn18_freihand2d_224x224/latest.onnx \
    --shape 1 3 224 224
```
* 'configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py' is your pose training config
* 'work_dirs/rsn18_freihand2d_224x224/latest.pth' is your trained pose model

The output onnx should be the same name as 'work_dirs/latest.pth' with '.onnx' post-fix in the same folder.
### Step 3-2: Verify ONNX:
tools/test_kneron.py is a script provided by kneron-mmpose to help users to verify if our exported ONNX generates similar outputs with what our PyTorch model does:
```python
python toos/test_kneron.py \
    configs/hand/2d_kpt_sview_rgb_img/topdown_heatmap/freihand2d/rsn18_freihand2d_224x224.py \
    work_dirs/rsn18_freihand2d_224x224/latest.onnx \
    --eval AUC EPE PCK \
```
The expected result of the command above should be something similar to the following text (the numbers may slightly differ):
```
AUC: 0.8550600860638303
EPE: 3.6879744058500914
PCK: 0.9876521630553889
```
Note that the ONNX results may differ from the PyTorch results due to some implementation differences between PyTorch and ONNXRuntime.

# Step 4: Convert onnx to [NEF](http://doc.kneron.com/docs/#toolchain/manual/#5-nef-workflow) model for Kneron platform

### Step 4-1: Install Kneron toolchain docker:
* check [document](http://doc.kneron.com/docs/#toolchain/manual/#1-installation)

### Step 4-2: Mout Kneron toolchain docker
* Mount a folder (e.g. '/mnt/hgfs/Competition') to toolchain docker container as `/data1`. The converted onnx in Step 3 should be put here. All the toolchain operation should happen in this folder.
```
sudo docker run --rm -it -v /mnt/hgfs/Competition:/data1 kneron/toolchain:latest
```

### Step 4-3: Import KTC and required lib in python shell
* Here we demonstrate how to go through all Kneron Toolchain (KTC) flow through Python API:
```python
import ktc
import numpy as np
import os
import onnx
from PIL import Image
```

### Step 4-4: Optimize the onnx model
```python
onnx_path = '/data1/latest.onnx'
m = onnx.load(onnx_path)
m = ktc.onnx_optimizer.onnx2onnx_flow(m)
onnx.save(m,'latest.opt.onnx')
```

### Step 4-5: Configure and load data necessary for ktc, and check if onnx is ok for toolchain
```python
# npu (only) performance simulation
km = ktc.ModelConfig((&)model_id_on_public_field, "0001", "720", onnx_model=m)
eval_result = km.evaluate()
print("\nNpu performance evaluation result:\n" + str(eval_result))
```

### Step 4-6: Quantize the onnx model
We [random sampled 50 images from voc dataset](https://www.kneron.com/forum/uploads/112/SMZ3HLBK3DXJ.7z) (50 images) as quantization data , we have to
1. Download the data
2. Uncompression the data as folder named `voc_data50"`
3. Put the `voc_data50` into docker mounted folder (the path in docker container should be `/data1/voc_data50`)

The following script will do some preprocess(should be the same as training code) on our quantization data, and put it in a list:
```python
import os
from os import walk

img_list = []
for (dirpath, dirnames, filenames) in walk("/data1/voc_data50"):
    for f in filenames:
        fullpath = os.path.join(dirpath, f)

        image = Image.open(fullpath)
        image = image.convert("RGB")
        image = Image.fromarray(np.array(image)[...,::-1])
        img_data = np.array(image.resize((224, 224), Image.BILINEAR)) / 256 - 0.5
        print(fullpath)
        img_list.append(img_data)
```

Then perform quantization. The BIE model will be generated at `/data1/output.bie`.

```python
# fixed-point analysis
bie_model_path = km.analysis({"input": img_list})
print("\nFixed-point analysis done. Save bie model to '" + str(bie_model_path) + "'")
```

### Step 4-7: Compile
The final step is compile the BIE model into an NEF model.
```python
# compile
nef_model_path = ktc.compile([km])
print("\nCompile done. Save Nef file to '" + str(nef_model_path) + "'")
```

You can find the NEF file at `/data1/batch_compile/models_720.nef`. `models_720.nef` is the final compiled model.

# Step 5: Run [NEF](http://doc.kneron.com/docs/#toolchain/manual/#5-nef-workflow) model on [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1)

* N/A

# Step 6 (For Kneron AI Competition 2022): Run [NEF](http://doc.kneron.com/docs/#toolchain/manual/#5-nef-workflow) model on [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1)

[WARNING] Don't do this step in toolchain docker enviroment mentioned in Step 4

Recommend you read [Kneron PLUS official document](http://doc.kneron.com/docs/#plus_python/#_top) first.

### Step 6-1: Download and Install PLUS python library(.whl)
* Go to [Kneron Education Center](https://www.kneron.com/tw/support/education-center/)
* Scroll down to `OpenMMLab Kneron Edition` table
* Select `Kneron Plus v1.3.0 (pre-built python library, firmware)`
* Select `python library`
* Select Your OS version (Ubuntu, Windows, MacOS, Raspberry pi)
* Download `KneronPLUS-1.3.0-py3-none-any_{your_os}.whl`
* Unzip downloaded `KneronPLUS-1.3.0-py3-none-any.whl.zip`
* `pip install KneronPLUS-1.3.0-py3-none-any.whl`

### Step 6-2: Download and upgrade KL720 USB accelerator firmware
* Go to [Kneron education center](https://www.kneron.com/tw/support/education-center/)
* Scroll down to `OpenMMLab Kneron Edition table`
* Select `Kneron Plus v1.3.0 (pre-built python library, firmware)`
* Select `firmware`
* Download `kl720_frimware.zip (fw_ncpu.bin、fw_scpu.bin)`
* unzip downloaded `kl720_frimware.zip`
* upgrade KL720 USB accelerator firmware(fw_ncpu.bin、fw_scpu.bin) by following [document](http://doc.kneron.com/docs/#plus_python/getting_start/), `Sec. 2. Update AI Device to KDP2 Firmware`, `Sec. 2.2 KL720`

### Step 6-3: Download RSN18-Hand example code
* Go to [Kneron education center](https://www.kneron.com/tw/support/education-center/)
* Scroll down to **OpenMMLab Kneron Edition** table
* Select **kneron-mmpose**
* Select **RSN18-Hand**
* Download **rsn18hand_plus_demo.zip**
* unzip downloaded **rsn18hand_plus_demo**

### Step 6-4: Test enviroment is ready (require [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1))
In `rsn18hand_plus_demo`, we provide a RSN18-HandPose example model and image for quick test. 
* Plug in [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1) into your computer USB port
* Go to the rsn18hand_plus_demo folder
```bash
cd /PATH/TO/rsn18hand_plus_demo
```

* Install required python libraries
```bash
pip install -r requirements.txt
```

* Run example on [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1)
```python
python KL720DemoGenericInferenceRSN18Hand_BypassHwPreProc.py -nef example_RSN18Hand_720.nef -img rsn_hand_test.png
```

Then you can see the inference result shown on your console window.
The expected result of the command above will be something similar to the following text:
```plain
...
[Connect Device]
 - Success
[Set Device Timeout]
 - Success
[Upload Model]
 - Success
======== NEF Info =========

Toolchain ver= kneron/toolchain:v0.17.2

Schema ver   = v0.9.1

============================
[Read Image]
 - Success
[Starting Inference Work]
 - Starting inference loop 1 times
 - .
[Retrieve Inference Node Output ]
 - Success
[Output Result Image]
 - Output hand-pose result on 'output_rsn_hand_test.png'
...
```

### Step 6-4: Run your NEF model and your image on [KL720 USB accelerator](https://www.kneo.ai/products/hardwares/HW2020122500000007/1)
Use the same script in previous step, but now we change the input NEF model path and image to yours
```bash
python KL720DemoGenericInferenceRSN18Hand_BypassHwPreProc.py -img /PATH/TO/YOUR_IMAGE.bmp -nef /PATH/TO/YOUR/720_NEF_MODEL.nef
```
