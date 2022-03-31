#########################################
创建 Notebook Instance 
#########################################

使用 SageMaker Notebook Instance 可以快速部署一个托管的 Jupyter 环境，相对于 SageMaker Studio，您有权限访问这个 Instance，因此可以做一些定制化工作。比如通过 Lifecycle 配置来加装依赖的组件。

.. contents::

**************************
添加 Lifecycle 配置
**************************

与平常的 EC2 实例不一样，Notebook Instance 在停止后，根卷会重置 （SageMaker 卷会保留,指 /home/ec2-user/SageMaker，默认5GB，可扩容到最多到16TB）。
如果您在根卷上安装了应用的话，重启后，需要重新安装。您可以通过 Lifecycle 配置一个脚本，实现自动安装。

除此之外，您可以在这里找到更多的 Lifecycle `使用示例。 <https://github.com/aws-samples/amazon-sagemaker-notebook-instance-lifecycle-config-samples>`__ 

另外，要注意您可以在 Lifecycle 里定义两种类型的脚本: **Create** 时运行或者 **Start** 时运行。很容易理解，前者只会运行一次，而后者在每次停止后，再重启时，都会运行。

**************************
创建 Notebook Instance
**************************

不需要修改任何设置即可成功创建一个 Notebook Instance，但默认设置不一定最优，我们建议您考虑进行修改。

- Notebook instance type. 如果您不需要在 Notebook 以本地模式进行训练（在 Notebook Instance 上进行训练），您可以选择小规格，比如 ml.t3.medium。另外，要记住在 AWS，新实例比旧实例便宜。
- Platform identifier. 建议选择 notebook-al2-v1，基于 Amazon Linux 2。
- Additional configuration. 在这里可以指定 Lifecycle 配置和 SageMaker 卷的大小。但您无法修改根卷大小，根卷默认 120 GiB，剩余可用空间大约 18 GiB。
- Network. 可以为 Notebook Instance 指定 VPC 设置，SageMaker 会在您指定的 VPC 和子网内创建一个 ENI，并套用您指定的安全组。这种设置下，我们建议您禁止 Direct Internet access，但需要这个子网有关联的 NAT Gateway 提供 Intrnet access，并且您指定的安全组允许出站访问。

**************************
修改 Notebook Instance
**************************

您可以在停止状态修改 Notebook Instance 配置，停止不需要的 Notebook Instance 也能为您节省成本，但您的 SageMaker 卷仍会有持续的存储费用产生。
您可以修改实例类型、扩容 SageMaker 卷、替换绑定的 IAM Role等等。