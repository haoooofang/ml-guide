#########################################
使用 SageMaker 自带算法
#########################################

SageMaker 为用户提供了数种 `开箱即用的算法 <https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html>`__。以下我们以 DeepAR 这个基于深度学习的预测算法进行讲解。
示例 Notebook 可以在 sagemaker examples 下的 **introduction_to_amazon_algorithms/deepar_synthetic** 找到。

.. contents::

**************************
获得镜像地址
**************************

和之前的自带算法不一样，在 estimator 配置中，您不再通过 entry_point 指定您的算法代码，而是需要指定已封装的镜像地址。在 `这个页面 <https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-algo-docker-registry-paths.html>`__ 您可以找到不同区域的镜像地址列表。
或者通过以下代码

.. code:: python

    from sagemaker.amazon.amazon_estimator import get_image_uri

    image_uri = get_image_uri(boto3.Session().region_name, "forecasting-deepar")


**************************
初始化 Estimator
**************************

这里有几个不同点:

- 因为算法不依赖某个框架，所以这里使用基础类 estimator。
- 不使用框架镜像，而是使用算法镜像，通过 image_uri 来指定。

.. code:: python

    estimator = sagemaker.estimator.Estimator(
        sagemaker_session=sagemaker_session,
        image_uri=image_uri,
        role=role,
        instance_count=1,
        instance_type="ml.c4.xlarge",
        base_job_name="DEMO-deepar",
        output_path=f"s3://{s3_output_path}",
    )

**************************
指定超参数
**************************

每个算法有超参可供配置，在算法介绍页面可以找到，比如 DeepAR 的在: https://docs.aws.amazon.com/sagemaker/latest/dg/deepar_hyperparameters.html

.. code:: python

    hyperparameters = {
        "time_freq": freq,
        "context_length": str(context_length),
        "prediction_length": str(prediction_length),
        "num_cells": "40",
        "num_layers": "3",
        "likelihood": "gaussian",
        "epochs": "20",
        "mini_batch_size": "32",
        "learning_rate": "0.001",
        "dropout_rate": "0.05",
        "early_stopping_patience": "10",
    }

    estimator.set_hyperparameters(**hyperparameters)

**************************
开始训练
**************************

正常指定数据 channel 后，就可以调用 fit 方法进行训练了。
