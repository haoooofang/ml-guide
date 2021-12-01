#########################################
扩展 SageMaker PyTorch 容器
#########################################

Amazon SageMaker 在训练和推理时，会按您在 Estimator 中的配置，指定指定规格和指定数量的实例，然后加载运行指定的 Docker 镜像。这些镜像会有不同的来源，包括：

- Amazon SageMaker 自带算法，可以 `通过 image_uris.retrieve 来获取 <https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-algo-docker-registry-paths.html>`__ ；

- Amazon SageMaker 提供框架，用户自带算法，我们称之为 Bring Your Own Script (BYOS)，以 `PyTorch 为例 <https://sagemaker.readthedocs.io/en/stable/frameworks/pytorch/sagemaker.pytorch.html>`__ ，我们在 ``entry_point`` 指定算法代码；

- 用户自带镜像(Bring Your Own Container, BYOC)，指由 `用户提供符合特性和安全需求的 Docker 镜像 <https://docs.aws.amazon.com/sagemaker/latest/dg/docker-containers-adapt-your-own.html>`__

除此之外，很多的时候，我们基于常用框架进行开发，但需要一些依赖包，这些依赖包并未包含在 SageMaker 提供的容器镜像里面，我们需要对这些预置容器镜像进行扩展。
以 PyTorch 为例，对于自有模块，我们可以和 entry point 文件放置在同一目录下（或者一个 S3 上的 tar.gz 文件），然后在 PyTorch ``source_dir`` 中指定这个目录的位置。
如果是外部模板，可以通过 ``pip`` 进行扩展。pip 方式非常简单，您在前面提到的 ``source_dir`` 中放置一个 ``requirements.txt``，容器在执行时，会自动安装指定的依赖包。因为在 PyTorch 1.3.1 版本以后，推理时这个文件必须处于 ``code`` 目录中，所以我们建议使用 ``code`` 作为 ``source_dir`` 目录名。

另外，我们也可以对 SageMaker 提供的 Docker 镜像进行扩展，通过 pip 或者 conda 安装依赖包，然后生成一个自定义镜像，但仍然以 PyTorch 这个 Estimator 来调用。这种方式的优点是减少了训练或者推理时的依赖包安装时间，比完全自定义镜像又来得简便些。以下我们就以在宁夏区域通过 conda 和 pip  添加 ``faiss`` 和 ``pyg`` 扩展包为例进行讲解。

.. contents::

原来镜像的内容
==============

官方镜像内容可以在 `github <https://github.com/aws/deep-learning-containers/>`__ 找到，比如 PyTorch 1.9 用于训练、支持 GPU 的镜像在 https://github.com/aws/deep-learning-containers/blob/master/pytorch/training/docker/1.9/py3/cu111/Dockerfile.gpu 。
在这个 Dockfile 中我们可以清楚的知道，这个 docker 镜像具体包含了哪些内容。其中我们可以看到镜像中已经内置了对 conda 环境的支持，我们无需重复安装。

镜像的地址
==============

在 `github  <https://github.com/aws/deep-learning-containers/blob/master/available_images.md>`__ 页面提供了 AWS 不同区域 ECR 镜像地址。
以宁夏区域为例，基础地址是 ``727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/<repository-name>:<image-tag>``
然后再在下面找到镜像的名称和 tag ，比如 ``763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.9.0-gpu-py38-cu111-ubuntu20.04``
把两者进行拼接，用宁夏区域的 endpoint 地址，再接上指定的镜像名称和 tag，结果像  ``727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.9.0-gpu-py38-cu111-ubuntu20.04``

访问镜像
=================

在已经安装了 docker，并且配置了亚马逊云科技访问凭据的机器上执行以下命令。因为 SageMaker Studio 未提供 docker，Notebook instance 自带 docker，但根卷剩余空间不大，建议在运行 Deep Learning AMI 的 EC2 实例中操作。
需要先给 EC2 实例相应的 ECR 操作权限。

.. code:: json

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "ecr:CreateRepository",
                    "ecr:GetDownloadUrlForLayer",
                    "ecr:BatchGetImage",
                    "ecr:CompleteLayerUpload",
                    "ecr:GetAuthorizationToken",
                    "ecr:UploadLayerPart",
                    "ecr:InitiateLayerUpload",
                    "ecr:BatchCheckLayerAvailability",
                    "ecr:PutImage"
                ],
                "Resource": "*"
            }
        ]
    }

接下来登录公共仓库，来获取基础镜像。

.. code:: bash

    aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn

如果命令成功返回，可以尝试拉取镜像

.. code:: bash

    docker pull 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.9.0-gpu-py38-cu111-ubuntu20.04

生成镜像
==================

容器中测试
------------

先在镜像中测试一下安装命令是否能够成功运行，先启动并进入容器。

.. code:: bash

    docker run -it --rm 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.9.0-gpu-py38-cu111-ubuntu20.04 /bin/sh

再运行 conda 安装命令。

.. code:: bash

    conda install -c conda-forge faiss-gpu cudatoolkit=11.1

把 `示例代码 <https://github.com/facebookresearch/faiss/blob/main/tutorial/python/1-Flat.py>`__ 复制到本地，然后通过 ``pyhon3 1-Flat.py`` 来执行。 
确认能正常运行后，我们开始正式构建镜像。

构建镜像
----------

从容器退出到 EC2 ，新建一个目录，目录中创建一个新文件 Dockerfile，内容如下

.. code:: Dockerfile

    From 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.9.0-gpu-py38-cu111-ubuntu20.04
    RUN conda install -y -c conda-forge faiss-gpu cudatoolkit=11.1 \
     && conda clean -ya

因为构建过程不能交互输入，所以加上 ``-y`` 来自动确认。

运行以下命令来生成镜像。 ``sagemaker-customized-image`` 是假设的镜像名称。镜像名需包含 ``sagemaker``，且不建议包含下划线，因为之后 SageMaker 会根据镜像名称生成训练任务，而训练任务名称不能含有下划线。

.. code:: bash

    docker build -t sagemaker-customized-image . 

完成后在 ECR 中创建一个 repository

.. code:: bash

    aws ecr create-repository \
        --repository-name sagemaker-customized-image \
        --image-scanning-configuration scanOnPush=true \
        --region cn-northwest-1

获取您自己 ECR 的访问权限，替换 ``aws_account_id`` 为您的 AWS账号ID，

.. code:: bash

    aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn

推送 docker 镜像到您的 ECR 中

.. code:: bash

    docker tag sagemaker-customized-image aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker-customized-image:20211118
    docker push aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker-customized-image:20211118

使用镜像
=================

启动一个 SageMaker Notebook Instance，选择PyTorch内核，运行以下代码。
先创建一个源代码目录。

.. code:: bash

    !mkdir code

把 `示例代码 <https://github.com/facebookresearch/faiss/blob/main/tutorial/python/1-Flat.py>`__ 复制到 ``code`` 目录，并重命名为 ``flat.py``。
安装 SageMaker 本地扩展。

.. code:: bash

    !pip install 'sagemaker[local]' --upgrade

在本地执行示例代码。 其中 ``source_dir`` 必须使用绝对地址，``aws_account_id``替换为自己的AWS账号id。

.. code:: python

    import sagemaker
    from sagemaker.local import LocalSession
    from sagemaker.pytorch import PyTorch
    import boto3
    from sagemaker import get_execution_role

    boto_session = boto3.Session(region_name='cn-northwest-1')
    sagemaker_session = LocalSession(boto_session=boto_session)
    sagemaker_session.config = {'local': {'local_code': True}}
    role = get_execution_role()

    pytorch_estimator = PyTorch(entry_point='flat.py',
                                instance_type='local_gpu',
                                instance_count=1,
                                image_uri='aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker-customized-image:20211118',
                                sagemaker_session=sagemaker_session,
                                role=role,
                                source_dir='/home/ec2-user/SageMaker/code'
                            )

    pytorch_estimator.fit()

可以在本地完成训练代码的调试，完成后就可以在云端进行正式的训练过程，仅需要把上面的代码作几作改动即可。

.. code:: python

    import sagemaker
    from sagemaker import Session
    from sagemaker.pytorch import PyTorch
    import boto3
    from sagemaker import get_execution_role

    boto_session = boto3.Session(region_name='cn-northwest-1')
    sagemaker_session = Session(boto_session=boto_session)
    role = get_execution_role()

    pytorch_estimator = PyTorch(entry_point='flat.py',
                                instance_type='ml.p3.2xlarge',
                                instance_count=1,
                                image_uri='aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker-customized-image:20211118',
                                sagemaker_session=sagemaker_session,
                                role=role,
                                source_dir='./code'
                            )

    pytorch_estimator.fit()

pip 安装依赖包
=================

有些情况下，包只有 pip 安装方式，或者 pip 安装更灵活。
我们以 ``pip`` 安装 ``pyg`` 依赖包来进行示范。
经过查看 pyg 和 PyTorch 官网，我们发现 PyTorch 1.10 和 Cuda 11.3 的支持环境。但我们经过查询，宁夏暂时没有这个版本的镜像。

.. code:: bash

    aws ecr list-images --registry-id 727897471807 --repository-name pytorch-training

我们可以把它搬回来。
首先，您需要在 EC2 上配置访问亚马逊云科技海外区域的凭据，可以运行 ``awscli`` 中的命令 ``aws configure`` 进行配置，或者设置环境变量。然后，就可以访问海外区域的 ECR 了。

.. code:: bash

    aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 763104351884.dkr.ecr.us-east-1.amazonaws.com
    docker pull 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.10.0-gpu-py38-cu113-ubuntu20.04-sagemaker

成功拉取镜像后，我们还是先进行容器内测试。
先运行镜像。

.. code:: bash

   docker run -it --rm 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.10.0-gpu-py38-cu113-ubuntu20.04-sagemaker /bin/sh

在 Docker 容器中删除镜像自带的 PyTorch ，再从官网安装标准版本。然后安装 pyg。这种安装方式不容易出现兼容性问题。

.. code:: bash

    pip uninstall -y torch
    pip uninstall -y torchvision

    pip install torch-scatter torch-sparse torch-cluster torch-spline-conv torch-geometric -f https://data.pyg.org/whl/torch-1.10.0+cu113.html
    pip install torch==1.10.0+cu113 torchvision==0.11.1+cu113 torchaudio==0.10.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html

安装完毕后检测一下 PyTorch 和 Cuda 版本：

.. code:: bash

    python -c "import torch; print(torch.__version__)"
    python -c "import torch; print(torch.version.cuda)"

使用 pyg examples目录中的 `示例代码 <https://github.com/pyg-team/pytorch_geometric/blob/master/examples/mnist_nn_conv.py>`__ 进行测试。

.. code:: bash 

    python3 mnist_nn_conv.py

测试通过后，退出容器，我们开始构建镜像， ``Dockfile`` 内容如下：

.. code:: Dockerfile

    From 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.10.0-gpu-py38-cu113-ubuntu20.04-sagemaker
    RUN pip uninstall -y torch \
     && pip uninstall -y torchvision \
     && pip install --no-cache-dir -U torch==1.10.0+cu113 torchvision==0.11.1+cu113 torchaudio==0.10.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html \
     && pip install --no-cache-dir -U torch-scatter torch-sparse torch-cluster torch-spline-conv torch-geometric -f https://data.pyg.org/whl/torch-1.10.0+cu113.html

在推送到宁夏 ECR 之前，我们需要先去掉亚马逊云科技海外区域的访问凭据或者使用国内的访问凭据进行覆盖， ``awscli`` 配置的话，需要重新运行 ``aws configure``。
然后就可以和上面的步骤一样，先获取 ECR 凭据，然后推送镜像，进行云端测试了。pyg 生成的镜像比较大，Notebook Instance 根卷剩余空间不够进行本地测试。

在上面的示例里面，我们介绍了如何通过 conda 和 pip 来扩展已有的 PyTorch 镜像，如何从海外区域获得国内区域暂时未上线的镜像，以及怎么使用 SageMaker local 模式在本地进行镜像测试。
可以看到，SageMaker 非常灵活，可以根据使用场景不同，对其进行灵活的定制以满足我们的使用要求。


     
