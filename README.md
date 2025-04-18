
# RLBench_ACT: Running ALoha ACT and Diffusion Policy in the RLBench Framework

## Declaration

This repository is a fork of the following projects:
- [Aloha ACT](https://github.com/tonyzhaozh/act)
- [RLBench](https://github.com/stepjam/RLBench)
- [PyRep](https://github.com/stepjam/PyRep)

It is part of the work on [Constrained Behavior Cloning for Robotic Learning](https://arxiv.org/abs/2408.10568?context=cs.RO).

## What's New?

- **February 12, 2025**: Added support for Joycon collect datasets with [joycon-robotics](https://github.com/box2ai-robotics/joycon-robotics).

- **January 8, 2025**: Added support for **variable step size** and **curve trajectory dataset generation**, as well as **dynamic step size training**, further optimizing **gripper control** during both generation and inference.

- **November 26, 2024**: Now supports **Sawyer**, **Panda**, and **UR5** robots. Added support for Panda's original environment acquisition, training, and inference.

## Installation (Ubuntu 20.04)

RLBench-ACT is built around **ACT**, **RLBench**, **PyRep**, and **CoppeliaSim v4.1.0**. We recommend using [Mamba](https://github.com/conda-forge/miniforge) as your Python version manager.

### 1. Create a Python virtual environment for RLBench_ACT:

```bash
conda create -n rlbench_act python=3.8.10  # The version is strict
conda activate rlbench_act
```

### 2. Download and set up CoppeliaSim (Ubuntu 20.04):

Download the [CoppeliaSim Ubuntu 20.04 package](https://www.coppeliarobotics.com/files/V4_1_0/CoppeliaSim_Edu_V4_1_0_Ubuntu20_04.tar.xz).

Install CoppeliaSim automatically:

```bash
mkdir -p $COPPELIASIM_ROOT && tar -xf CoppeliaSim_Edu_V4_1_0_Ubuntu20_04.tar.xz -C $COPPELIASIM_ROOT --strip-components 1
ln -s $COPPELIASIM_ROOT/libcoppeliaSim.so $COPPELIASIM_ROOT/libcoppeliaSim.so.1
```

Add the following to your `~/.bashrc` file:

```bash
export COPPELIASIM_ROOT=~/COPPELIASIM  # Adjust this path if needed
export LD_LIBRARY_PATH=$COPPELIASIM_ROOT:$LD_LIBRARY_PATH
export QT_QPA_PLATFORM_PLUGIN_PATH=$COPPELIASIM_ROOT
```

To test the CoppeliaSim installation:

```bash
bash $COPPELIASIM_ROOT/coppeliaSim.sh
```

### 3. Install the repository and dependencies:

```bash
# Clone the repository
conda activate rlbench_act
git clone https://github.com/Boxjod/RLBench_ACT.git
cd RLBench_ACT

# Install dependencies
pip install -r requirements.txt
pip install -e ./PyRep  # More info at https://github.com/stepjam/PyRep
pip install -e ./RLBench  # More info at https://github.com/stepjam/RLBench
pip install -e ./act/detr  # More info at https://github.com/tonyzhaozh/act

# Install PyTorch with CUDA support
conda install pytorch==2.1.2 torchvision==0.16.2 torchaudio==2.1.2 pytorch-cuda=12.1 -c pytorch -c nvidia
```

## Usage

### 1. Test RLBench Task Builder

```bash
conda activate rlbench_act
python3 RLBench/tools/task_builder2.py --task sorting_program5 --robot sawyer 
# [Remember: Do not save the scene in CoppeliaSim GUI but in the terminator with input 's']
```
**Note: Do not save code in Coppeliasim's GUI**, either by using Ctrl+S or by confirming in the “Did you save changes?” popup when closing the window. Saving the scene in the GUI can result in missing components and lead to errors in subsequent executions. If you accidentally save changes, restore the task with the following commands:

```bash
# If you encounter errors like "RuntimeError: Handle cam_head_mask does not exist"
cd RLBench/rlbench
rm task_design_sawyer.ttt task_design.ttt
cp back_ttt/task_design_sawyer_back.ttt task_design_sawyer.ttt
cp back_ttt/task_design_back.ttt task_design.ttt
cd ..; cd ..
```

### 2. get robot task demo from RLBench. 

    
```bash
python3 RLBench/tools/dataset_generator_hdf5.py \
--save_path Datasets \
--robot sawyer \
--tasks sorting_program5 \
--variations 1 \
--episodes_per_task 50 \
--dynamic_step=False \
--onscreen_render=True # False if you don't want to show the window
```

**NEW (2025.2.7)**: Data collection now supports RoboStick!  
Make sure you have RoboStick installed correctly by following the instructions in [this repository](https://github.com/box2ai-robotics/joycon-robotics). Once installed, you can start collecting data by running the command below:

```bash
python3 act/dataset_collect_joycon.py \
--save_path Datasets \
--robot sawyer \
--tasks sorting_program5 \
--variations 1 \
--episodes_per_task 50 
```
Tips for operation:
- `stick button down`: go down follow the direction the gripper.
- `stick movement`: the gripper move in the plane.
- `higher trigger key(R)`: go up opposite direction of the gripper.
- `lower trigger key(ZR)`: control the gripper open or close.
- `Y`: restart this episode.
- `A`: save the episode and start the next.

### 3. visualize episode

```bash
python3 act/visualize_episodes.py --dataset_dir Datasets/sorting_program5/variation0 --episode_idx 0
```
In addition, we recommend the use of hdf5 visualization web tools [myhdf5](https://myhdf5.hdfgroup.org/)

### 4. train task and eval
    
```bash
# train
python3 act/imitate_episodes_rlbench.py \
--task_name sorting_program5 \
--ckpt_dir Trainings/sorting_program5 \
--policy_class ACT --kl_weight 10 --chunk_size 20 --hidden_dim 512 --batch_size 8 --dim_feedforward 3200 \
--num_epochs 2000  --lr 1e-5 \
--seed 0 --robot sawyer 

# infrence
python3 act/imitate_episodes_rlbench.py \
--task_name sorting_program5 \
--ckpt_dir Trainings/sorting_program5 \
--policy_class ACT --kl_weight 10 --chunk_size 20 --hidden_dim 512 --batch_size 8 --dim_feedforward 3200 \
--num_epochs 2000  --lr 1e-5 --temporal_agg \
--seed 0 --eval --onscreen_render --robot sawyer 
```
Under normal conditions, after 2000 epochs, the Sawyer can achieve a **success rate of 56%** for the `sorting_program5` task.

## Task build

### 1. We recommend creating a new task from an already existing task. For example

```bash
python3 RLBench/tools/task_builder2.py --task sorting_program5 --robot sawyer
```
input the 'u' in the terminal window you can duplicate the task to a new name.

After your change, remember to save the task with the 's' in the terminal window. And you can test the task with "+" and "d".
But don't save this task scence in the CoppeliaSim GUI!!!

### 2. Edit the waypoints. 
And double-click on the waypoints and select Common Modification Extension string at the top of the pop-up window. The command list are bellow:

- `ignore_collisions`;
- `open_gripper()`;
- `close_gripper()`;
<!-- - `steps(12)`; # You can set a fixed number of steps to reach this path point. -->

### 3. To generate a dataset and train model based on waypoints 

add your task and modify the following parameters in `act/constants.py`:

- `dataset_dir`: Directory to store the dataset (e.g., `DATA_DIR + '/sorting_program5/variation0'`)
- `episode_len`: Reference steps for model training (e.g., `100`)
- `num_episodes`: Number of episodes to generate for training (e.g., `50`)
- `num_variation`: Target variation (e.g., `1`)
- `camera_names`: Cameras to record data or train (e.g., `['wrist']`)

After setting these, generate the dataset and start training.

## More Info
1. Watch the installation video on Bilibili: https://www.bilibili.com/video/BV1dExnerE2T/
2. Join the discussion on the QQ group: 948755626

## Cite
If you find the RLBench_ACT in this repository useful for your research, you can cite:
```
@software{junxie2024RLBench_ACT,
    title={RLBench-ACT},
    author={Jun Xie},
    year={2024},
    url = {https://github.com/Boxjod/RLBench_ACT},
}
```
or our work on Constrained Behavior Cloning for Robotic Learning
```
@article{junxie2024cbc,
  title   = {Constrained Behavior Cloning for Robotic Learning},
  author  = {Wensheng Liang and Jun Xie and Zhicheng Wang and Jianwei Tan and Xiaoguang Ma},
  year    = {2024},
  journal = {arXiv preprint arXiv:2408.10568}
}
```

If this project is helpful to you, please give us a star. We greatly appreciate it! ⭐ ⭐ ⭐ ⭐ ⭐


