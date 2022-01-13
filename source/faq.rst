#########################################
常见问题解答
#########################################

1. 国内区域没有 GroundTruth 怎么进行数据标注？
   如果是 CV 用户，可以看看我们的 `解决方案 <https://www.amazonaws.cn/en/solutions/machine-learning-bot/>`__。我们也有很多合作伙伴能提供标注服务，请和我们联系。

2. SageMaker 推理没有预留实例，怎么降低成本？
   海外区域有 Savings Plan，可以更灵活的降低成本。另外，您完全可以把模型导出，部署在 EC2、ECS、EKS上，我们推出了 `Deep Learning Containers <https://aws.amazon.com/cn/machine-learning/containers/>`__ 方便您快速部署。

3. 机型选择有什么建议
   建议用 P3/P4 进行 GPU 训练，C5 进行 CPU 训练/推理，G4/INF1 进行 GPU 推理。

4. Notebook 有些跑不通。
   因为各种上框架和依赖包迭代迅速，有时会出现 API 接口变化、数据存储位置变化等问题，造成 Notebook 跑不通，建议根据最新 API/包 对代码进行调整，也可以联系我们。

   