#########################################
通过 Conda 扩展 SageMaker 容器
#########################################

Sagemaker 在训练和推理时，会按您在 Estimator 中的配置，指定指定规格和指定数量的实例，然后加载运行指定的 docker 镜像。

.. contents::

镜像的内容
==============

官方镜像内容可以在 `github <https://github.com/aws/deep-learning-containers/>`__ 找到，比如 PyTorch 1.10 用于训练、支持 GPU 的镜像在 https://github.com/aws/deep-learning-containers/blob/master/pytorch/training/docker/1.10/py3/cu113/Dockerfile.sagemaker.gpu。
在这个 Dockfile 中我们可以清楚的知道，这个 docker 镜像具体包含了哪些内容。

镜像的地址
==============

在 `github  <https://github.com/aws/deep-learning-containers/blob/master/available_images.md>`__ 页面提供了 AWS 不同区域 ECR 镜像地址。
其中，中国宁夏的基础地址是 ``727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/<repository-name>:<image-tag>``
再在下面找到镜像的名称和 tag ，中国区域镜像更新有时稍晚，我们可以选用一个较早版本的镜像，比如 ``763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.8.1-gpu-py36-cu111-ubuntu18.04``
然后把两者进行拼接，用国内的 endpoint 地址，再接上指定的镜像名称和 tag，结果像  ``727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.8.1-gpu-psy36-cu111-ubuntu18.04``

访问镜像
=================

在已经安装了 docker，并且配置了 AWS 访问凭据的机器上执行以下命令。Sagemaker Studio 未提供 docker，Notebook instance 已经自带 docker，或者运行 Deep Learning AMI 的 EC2，或者您本地计算机。
Notebook Instance 根卷只有 120GB，已占用 108GB，如果需要处理复杂的镜像，建议在 EC2 上进行。
本地计算机需要通过 AWS configure 配置 AWS 凭据，AWS 资源可以通过 role 来取得临时 token。确保他们包含 ECR 读取权限。Notebook instance 一般会配置 SageMaker ExecutionRole ，它会具备 SageMakerFullAccess 策略，其中已经包含 ECR 访问权限。

.. code:: bash

    aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn

如果命令成功返回，可以尝试拉取镜像

.. code:: bash

    docker pull 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.8.1-gpu-py36-cu111-ubuntu18.04

生成镜像
==================

新建一个目录，目录中创建一个新文件 Dockerfile，内容如下

.. code:: Dockerfile

    From 727897471807.dkr.ecr.cn-northwest-1.amazonaws.com.cn/pytorch-training:1.8.1-gpu-py36-cu111-ubuntu18.04
    RUN conda install -y -c conda-forge faiss-gpu

``sagemaker_customized_image`` 是镜像名称，如果使用 SageMaker ExecutionRole，镜像名称必须包含 **sagemaker**

.. code:: bash

    docker build -t sagemaker_customized_image . 

完成后在 ECR 中创建一个 repository

.. code:: bash

    aws ecr create-repository \
        --repository-name sagemaker_customized_image \
        --image-scanning-configuration scanOnPush=true \
        --region cn-northwest-1

获取您自己 ECR 的访问权限，替换 aws_account_id 为您的 AWS账号ID，

.. code:: bash

    aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn

推送 docker 镜像到您的 ECR 中

.. code:: bash

    docker tag sagemaker_customized_image aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker_customized_image:20211118
    docker push aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker_customized_image:20211118

使用镜像
=================

在 Estimator 中去掉 **framework_version** 和 **py_version**，改使用 **image_uri** 来指定上来的 ECR 镜像地址。

.. code:: python

    pytorch_estimator = PyTorch(entry_point='pytorch-train.py',
                                instance_type='ml.p3.2xlarge',
                                instance_count=1,
                                image_uri='aws_account_id.dkr.ecr.cn-northwest-1.amazonaws.com.cn/sagemaker_customized_image:20211118'
                                hyperparameters = {'epochs': 20, 'batch-size': 64, 'learning-rate': 0.1})



