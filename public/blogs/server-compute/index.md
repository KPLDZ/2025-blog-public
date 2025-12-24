近日在`Duke dcc on-demand`上使用了服务器运行自己的脚本，记录一下使用的过程（账号是借用的）。
服务器端系统是`AlmaLinux 9.6 (Sage Margay)`，属于`Red Hat Enterprise Linux (RHEL)`的一个免费兼容版本。接下来记录提交离线任务的代码。

在网页上申请一个服务器端口，待端口成功运行开放后，进入服务器端页面，自动连接至服务器端的`jupyter lab`，创建你的工作目录并在工作目录中打开`Terminal`终端，输入如下指令创建一个sh文件用于`bash`提交。

```
# 创建sh文件
nano submit_training.sh
```

创建sh文件后，页面一般会自动跳转到sh文件内部，之后将下面的配置原封不动粘贴至sh文件中，需要注意的是"SBATCH"前面的"#"符号不是注释，而是必要的格式符，需要保留！

```
#!/bin/bash

# ===================================================
# SLURM 作业资源请求部分
# 请根据您的集群策略修改以下参数
# ===================================================

#SBATCH --job-name=UrbanSound_CV     # 作业名称，根据自己的情况修改
#SBATCH --partition=biostat-gpu              # 提交到 GPU 分区 (请根据您的集群实际分区名调整)
#SBATCH --nodes=1                    # 请求 1 个节点
#SBATCH --ntasks-per-node=1          # 每个节点运行 1 个任务
#SBATCH --gres=gpu:1                 # 请求 1 块 GPU (如果需要更多，请修改数字)
#SBATCH --mem=32G                    # 请求 32 GB 内存 (根据您的数据量调整)
#SBATCH --time=24:00:00              # 最长运行时间（48小时，请根据您的需求调整）
#SBATCH --output=slurm-%j.out        # 标准输出日志文件，%j 会替换为 Job ID

echo "SLURM Job ID: $SLURM_JOB_ID"
echo "开始运行时间: $(date)"

# ===================================================
# 环境设置和代码运行部分
# ===================================================

# 1. 加载 Anaconda 模块
# 此步骤一般和申请服务器jupyter lab端时设定的模块一致即可。
module purge
module load Anaconda3/2024.02
module list

# 2. 激活您的 Conda 环境
# base是环境名，依据你的实际情况修改名字，比如运行需要pytorch环境的代码需要确保该环境已配置好torch环境
source activate base 

# 3. 加载 CUDA 模块 (如果您的集群需要此步骤才能访问 GPU)
# 请使用您 PyTorch 安装时匹配的 CUDA 版本，如果集群默认版本就可兼容，那么可以注释掉这行代码。
module load cuda/12.8

# 4. 运行 Python 脚本
# 运行您在需要在服务器端离线运行的 Python 文件（方便起见建议将该文件也放在同级目录下）
python 1-10cv.py

# ===================================================

echo "程序运行结束时间: $(date)"
```

写完sh文件后保存，然后继续在同样的当前目录下打开`Terminal`终端，运行如下代码提交sh文件到服务器集群，这一步骤需要二次排队，服务器集群会在闲置资源满足sh文件中的要求时自动启动提交的任务。

```
# 提交sh文件
sbatch submit_training.sh
```

提交任务后通过以下代码实施查询任务状态,`your_username`改为你在服务器系统中的用户名即可，可以查询任务提交后经历的时间，任务的排队状态（排队或执行中）、任务的ID等信息。

```
# 查询任务状态
squeue -u [your_username]
```

执行完上面一行代码后会生成一个任务ID显示在终端中，可以及时记录并查询任务的运行结果

```
# 查询对应任务的ID，比如40629659，动态监控结果
tail -f slurm-[40629659].out
```