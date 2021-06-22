# Pose2Carton 

EE228 课程大作业 利用3D骨架控制3D卡通人物 (https://github.com/yuzhenbo/pose2carton) 

数据组别： group4

数据类型： 18组匹配 + 1组蒙皮)


# Maya 环境配置

Maya环境的配置主要参考https://zhuanlan.zhihu.com/p/367649237

最初试图在Ubuntu18.04上配置Maya环境，但是在配置Mayapy的过程中遇到不可知的错误，所以最后考虑在Windows系统上完成对Maya环境的配置。配置过程大致如下：

1. 在Maya官网申请学生教育版认证，从而获得Maya产品的使用权。

2. 申请完成后，可以在个人账户(Account)下的产品与服务(Products and Services)中下载对应的Maya2020

3. 下载完毕后，按照安装指示一步步完成即可(PS: 安装路径最好不要在系统盘，路径名称最好不要出现中文)

4. 安装完毕后，能够正常运行maya2020，即能够打开其界面，说明Maya环境配置接近完成。接下来，需要配置mayapy的一些基本的库。(接下来的过程中请勿提前关闭maya2020界面)

5. 首先，将maya2020安装目录里的bin文件的路径添加进环境变量。若通过cmd运行mayapy能够正常运行，这说明添加成功。

6. 接着分别在cmd里输入下列命令进行安装项目需要的pip和numpy库

   ```shell
   curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
   mayapy get-pip.py
   mayapy -m pip install -i https://pypi.anaconda.org/carlkl/simple numpy
   ```

7. 安装完毕后，检查下列命令是否能够正常运行，其中，前三行是为了解决“windows系统用户名为中文”的问题

   ```python
   import sys
   reload(sys)
   sys.setdefaultencoding("gbk")
   import maya
   import maya.standalone
   maya.standalone.initialize(name='python')
   import maya.OpenMaya as om
   import maya.cmds as cmds
   import pymel.core as pm
   import maya.mel as mel
   import numpy as np
   import os
   import glob
   ```

8. 最后，从网上下载一个fbx文件，与fbx_parser.py放在同一目录下，在cmd中运行

   ```shell
   mayapy fbx_parser.py xxxx.fbx
   ```

   若能顺利运行，则说明Maya环境配置完成。

# 匹配流程

##### 匹配代码运行流程如下：

​	1. 将人体模型序列和transfer.py 放在同一个目录下，文件名命名应为obj_seq_5

​	2. 打开transfer.py, 需要修改的变量主要为

​			sample_path: 模型的节点txt文件路径，str类型

​			use_online_model: 是否是网上下载的模型，是则赋值为True，bool类型 

​			manual_model_to_smpl: 各关节点的对应关系，dict类型

3. 填写好上述前两个变量后，第三个先为空，运行transfer.py，能够得到模型的关节点和索引的对应关系，依据这个关系填写第三个变量，再运行即可。
4. 运行完毕后，再通过运行vis.py文件，即可对匹配结果可视化，并生成mp4文件。

##### 蒙皮代码运行流程如下

​	1. 首先，从网站(如Mixamo)上下载模型的fbx文件(如model.fbx)，与fbx_parser.py放在同一个目录下，在cmd中运行	

```shell
  mayapy fbx_parser.py model.fbx
```

​		可以得到model.obj, model.txt, model_intermediate.obj, model_intermediate.mtl, model.fbm这几个文件。其中，通过前两个文件可以进行匹配流程。model.fbm中的png文件是从模型解析出的texture。

2. 将model.fbm和model_intermediate.mtl放入obj_seq_5_3dmodel文件夹(运行transfer.py后会自动生成)中，可能还需要修改mtl文件中的路径为相对路径。最后运行vis.py即可。

##### 对代码中关键函数的理解:

###### transfer.transfer_one_sequence(infofile, seqfile)：

​		对全部序列进行转换。infofile是filename.fbx对应的filename.txt文件，filename.txt文件一开始每一行是关节点信息，以“joints”开头，中间部分每一行是蒙皮信息，以“skin”开头，最后一部分是层次信息，以“hier”开头。而seqfile就是选择的info_seq_5.pkl文件， 其中保存的是所有帧的旋转矩阵。

###### transfer._lazy_get_model_to_smpl：

​		根据关节的名称自动生成对应关系的字典，从而进行匹配。

###### transfer.transfer_given_pose: 

​		第一步，读取filename.txt文件中的层次结构信息，将每一行的信息综合起来就可以建立关节点名称到索引的映射，建立动力学结构。然后重新组织映射，确保每条运动链上索引递增。

​		第二步则是从filename.txt中解析出蒙皮权重， 表征为 J x V 的矩阵, J是关节点数，而V是顶点数，即filename.txt中以“skin”开头的行数。同时读取filename.txt中以“joints”开头的部分, 从中解析出T-posed的卡通人物骨架，表现为J x 3的张量，即每一个关节点的空间坐标。

​		第三步是利用正向运动学将T姿势骨架转化为姿势化的角色骨架。对每一个关节点，我们将旋转向量和它与父节点(动力学结构中的上方相邻关节点)的坐标差放在一起组成新的信息，并和父节点的相应信息相乘，生成矩阵，然后从T-pose获取关节偏移，得到骨架G。

​		最后，根据第二步中解析得到的蒙皮权重和第三步得到的骨架生成混合权值，即线性蒙皮，从骨架得到mesh。

​	

# 新增脚本说明

​		考虑到从网上下载的模型可能存在多mesh的情况，导致fbx_parser.py解析时无法得到完整的模型，所以通过修改fbx_parser.py中record_obj函数，使得其将所有的mesh都解析出来，之后可以通过blender对多个mesh进行合并，从而解决此类问题。

​		<img src="D:\TOOL\Github Desktop\pose2carton\img\newcode.png" alt="image-20210621124633445" style="zoom: 25%;" /><img src="D:\TOOL\Github Desktop\pose2carton\img\resultoutput.png" alt="image-20210621124646172" style="zoom:33%;" />

​		脚本的修改主要是通过循环访问geoList列表中的mesh信息，并以不同的文件名生成。



# 项目结果

<img src="../img/176.png" alt="image" style="zoom:50%;" /><img src="D:\TOOL\Github Desktop\pose2carton\img\363.png" alt="363" style="zoom:50%;" />   



# 协议 
本项目在 Apache-2.0 协议下开源

所涉及代码及数据的所有权以及最终解释权归倪冰冰老师课题组所有. 