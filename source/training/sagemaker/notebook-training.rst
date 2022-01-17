#########################################
使用 Notebook Instance 训练模型
#########################################

最快速掌握 SageMaker 的使用，莫过于从官方示例开始，我们在 https://github.com/aws/amazon-sagemaker-examples 托管了示例 notebooks。另外在 https://sagemaker-examples.readthedocs.io/en/latest/get_started/index.html 提供了说明文档。

.. contents::

****************************************************
复制 SageMaker Examples 示例 notebook
****************************************************

在 Notebook Instance 控制台页面，点击 **Open JupyterLab** 打开 JupyterLab 页面。
在菜单栏 **Git** 中，选择 **Clone a Repository**，输入 https://github.com/aws/amazon-sagemaker-examples.git ，等待复制完成。
在文件夹中可以看到多了 **amazon-sagemaker-examples** ，您可以浏览子文件夹，选择感兴趣的示例

****************************************************
运行示例 Notebook
****************************************************

比如我们打开 sagemaker-python-sdk/pytorch_mnist 这个文件夹，双击运行 pytorch_mnist.ipynb 这个 Notebook。
每个示例 Notebook 都提供了背景说明、示例数据、示例代码。
有些时候需要安装依赖包，国内区域可以考虑使用加速镜像站，比如使用 `清华镜像站 <https://mirror.tuna.tsinghua.edu.cn/help/pypi/>`__

初始化环境
=================

后续我们会需要用到三个因素:

- SageMaker session. 用于设置 SageMaker 环境，可以通过传入一个 boto_session 来设置区域、AWS 访问凭据。
- Bucket. 存储训练结果等。SageMaker 会自动在所在区域创建一个包含您账号 ID 的默认 Bucket。可以通过 ``sagemaker_session.default_bucket`` 来引用。
- Role. 之后传递给 SageMaker，让训练实例使用这个 Role 来访问 AWS 资源，比如 S3 Bucket。
  
准备数据
=================

一般而言，我们会先把数据存储到 S3，然后在训练时通过 ``channel`` 指定 S3 路径，训练实例会自动把全部数据复制到特定目录。使用 **管道模式** 可以减少数据加载 `等待时间 <https://aws.amazon.com/blogs/machine-learning/accelerate-model-training-using-faster-pipe-mode-on-amazon-sagemaker/>`__ 。对速度要求更高的话，可以考虑 `Lustre 和 NFS <https://aws.amazon.com/cn/blogs/china/use-amazon-fsx-for-lustre-and-amazon-efs-as-data-source-to-speed-up-amazon-sagemaker-training/>`__

开始训练
=================

通过一个 estimator 对象，我们定义训练过程。包括:

- entry_point. 实际训练代码。
- role. 前面提到的，训练实例可以 assume 的 Role。
- py_version/framework_version. 如果使用 SageMaker 提供的训练镜像，您只需要指定 Python 和框架版本，SageMaker 会自动查找合适的 ECR 镜像并拉取到训练实例中。
- instance_count/instance_type. 训练实例数量和规格。训练任务是跑在单独的实例上，这些实例由 SageMaker 管理。
- hyperparameters. 传递给训练代码的参数。

调用 estimator 的 ``fit`` 方法开始训练过程，通过 ``channel`` 指定数据位置。
可以在 cell 下看到日志输出。与此同时，控制台 ``Training`` 页面也会出现这个训练任务。在 CloudWatch Logs 中还可以看到不同训练实例办理出的日志信息。

