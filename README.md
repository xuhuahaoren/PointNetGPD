# PointNetGPD: Detecting Grasp Configurations from Point Sets
PointNetGPD ([arXiv](https://arxiv.org/abs/1809.06267)) is an end-to-end grasp evaluation model to address the challenging problem of localizing robot grasp configurations directly from the point cloud.

PointNetGPD is light-weighted and can directly process the 3D point cloud that locates within the gripper for grasp evaluation. Taking the raw point cloud as input, our proposed grasp evaluation network can capture the complex geometric structure of the contact area between the gripper and the object even if the point cloud is very sparse.

To further improve our proposed model, we generate a larger-scale grasp dataset with 350k real point cloud and grasps with the [YCB objects Dataset](http://ycb-benchmarks.s3-website-us-east-1.amazonaws.com/) for training.

<img src="data/grasp_pipeline.svg" width="100%">

## Video

[![Video for PointNetGPD](https://img.youtube.com/vi/RBFFCLiWhRw/0.jpg )](https://www.youtube.com/watch?v=RBFFCLiWhRw)
## Before Install
All the code should be installed in the following directory:
```
mkdir -p $HOME/code/
cd $HOME/code/
```
## Install all the requirements (Using a virtual environment is recommended)
1. Make sure in your Python environment do not have same package named ```meshpy``` or ```dexnet```.

1. Clone this repository:
    ```bash
    cd $HOME/code
    git clone https://github.com/lianghongzhuo/PointNetGPD.git
    mv PointNetGPD grasp-pointnet
    ```

1. Install our requirements in `requirements.txt`
    ```bash
    cd $HOME/code/grasp-pointnet
    pip install -r requirements.txt
    ```
1. Install our modified meshpy (Modify from [Berkeley Automation Lab: meshpy](https://github.com/BerkeleyAutomation/meshpy))
    ```bash
    cd $HOME/code/grasp-pointnet/meshpy
    python setup.py develop
    ```

1. Install our modified dex-net (Modify from [Berkeley Automation Lab: dex-net](https://github.com/BerkeleyAutomation/dex-net))
    ```bash
    cd $HOME/code/grasp-pointnet/dex-net
    python setup.py develop
    ```
1. Modify the gripper configurations to your own gripper
    ```bash
    vim $HOME/code/grasp-pointnet/dex-net/data/grippers/robotiq_85/params.json
    ```
    These parameters are used for dataset generation：
    ```bash
    "min_width":
    "force_limit":
    "max_width":
    "finger_radius":
    "max_depth":
    ```
    These parameters are used for grasp pose generation at experiment:
    ```bash
    "finger_width":
    "real_finger_width":
    "hand_height":
    "hand_height_two_finger_side":
    "hand_outer_diameter":
    "hand_depth":
    "real_hand_depth":
    "init_bite":
    ```




## Generate Grasp Dataset

1. Download YCB object set from [YCB Dataset](http://ycb-benchmarks.s3-website-us-east-1.amazonaws.com/).
2. Manage your dataset here:
    ```bash
    mkdir -p $HOME/dataset/ycb_meshes_google/objects
    ```
    Every object should have a folder, structure like this:
    ```bash
    ├002_master_chef_can
    |└── google_512k
    |    ├── kinbody.xml (no use)
    |    ├── nontextured.obj
    |    ├── nontextured.ply
    |    ├── nontextured.sdf (generated by SDFGen)
    |    ├── nontextured.stl
    |    ├── textured.dae (no use)
    |    ├── textured.mtl (no use)
    |    ├── textured.obj (no use)
    |    ├── textured.sdf (no use)
    |    └── texture_map.png (no use)
    ├003_cracker_box
    └004_sugar_box
    ...
    ```
3. Install SDFGen from [GitHub](https://github.com/jeffmahler/SDFGen.git):
    ```bash
    git clone https://github.com/jeffmahler/SDFGen.git
    cd SDFGen
    sudo sh install.sh
    ```
3. Install python pcl library [python-pcl](https://github.com/strawlab/python-pcl):
    ```bash
    git clone https://github.com/strawlab/python-pcl.git
    pip install --upgrade pip
    pip install cython==0.25.2
    pip install numpy
    cd python-pcl
    python setup.py build_ext -i
    python setup.py develop
    ```
4. Generate sdf file for each nontextured.obj file using SDFGen by running:
    ```bash
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python read_file_sdf.py
    ```
5. Generate dataset by running the code:
    ```bash
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python generate-dataset-canny.py [prefix]
    ```
    where ```[prefix]``` is the optional, it will add a prefix on the generated files.

## Visualization tools
- Visualization grasps
    ```bash
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python read_grasps_from_file.py [prefix]
    ```
    Note:
    - ```prefix``` is optional, if added, the code will only show a specific object, else, the code will show all the objects in order.
    - If you have error like this: ```ImportError: No module named shapely.geometry```, do ```pip install shapely``` should fix it.


- Visualization object normals
    ```bash
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python Cal_norm.py
    ```
This code will check the norm calculated by meshpy and pcl library.
## Training the network
1. Data prepare:
    ```bash
    cd $HOME/code/grasp-pointnet/PointNetGPD/data
    ```

    Make sure you have the following files, The links to the dataset directory should add by yourself:
    ```
    ├── google2cloud.csv  (Transform from google_ycb model to ycb_rgbd model)
    ├── google2cloud.pkl  (Transform from google_ycb model to ycb_rgbd model)
    ├── ycb_grasp -> $HOME/dataset/ycb_grasp  (Links to the dataset directory)
    ├── ycb_meshes_google -> $HOME/dataset/ycb_meshes_google/objects  (Links to the dataset directory)
    └── ycb_rgbd -> $HOME/dataset/ycb_rgbd  (Links to the dataset directory)
    ```

    Generate point cloud from rgb-d image, you may change the number of process running in parallel if you use a shared host with others
    ```bash
    cd ..
    python ycb_cloud_generate.py
    ```

1. Run the experiments:
    ```bash
    cd PointNetGPD
    ```

    Launch a tensorboard for monitoring
    ```bash
    tensorboard --log-dir ./assets/log --port 8080
    ```

    and run an experiment for 200 epoch
    ```
    python main_1v.py --epoch 200
    ```

    File name and corresponding experiment:
    ```
    main_1v.py        --- 1-viewed point cloud, 2 class
    main_1v_mc.py     --- 1-viewed point cloud, 3 class
    main_1v_gpd.py    --- 1-viewed point cloud, GPD
    main_fullv.py     --- Full point cloud, 2 class
    main_fullv_mc.py  --- Full point cloud, 3 class
    main_fullv_gpd.py --- Full point cloud, GPD
    ```

    For GPD experiments, you may change the input channel number by modifying `input_chann` in the experiment scripts(only 3 and 12 channels are available)

## Using the trained network (Only work in python2 as we use ROS)

1. Get UR5 robot state:

    Goal of this step is to publish a ROS parameter tell the environment whether the UR5 robot is at home position or not.
    ```
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python get_ur5_robot_state.py
    ```
2. Run perception code:
    This code will take depth camera ROS info as input, and gives a set of good grasp candidates as output.
    All the input, output messages are using ROS messages.
    ```
    cd $HOME/code/grasp-pointnet/dex-net/apps
    python kinect2grasp_python2.py

    arguments:
    -h, --help                 show this help message and exit
    --cuda                     using cuda for get the network result
    --gpu GPU                  set GPU number
    --load-model LOAD_MODEL    set witch model you want to use (rewrite by model_type, do not use this arg)
    --show_final_grasp         show final grasp using mayavi, only for debug, not working on multi processing
    --tray_grasp               not finished grasp type
    --using_mp                 using multi processing to sample grasps
    --model_type MODEL_TYPE    selet a model type from 3 existing models
    ```

## Citation
If you found PointNetGPD useful in your research, please consider citing:

```plain
@article{liang2018pointnetgpd,
  title={PointNetGPD: Detecting Grasp Configurations from Point Sets},
  author={Liang, Hongzhuo and Ma, Xiaojian and Li, Shuang and G{\"o}rner, Michael and Tang, Song and Fang, Bin and Sun, Fuchun and Zhang, Jianwei},
  journal={arXiv preprint arXiv:1809.06267},
  year={2018}
}
```
