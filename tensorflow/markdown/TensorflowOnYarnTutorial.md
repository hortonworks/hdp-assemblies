<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
# Tutorial: Running Tensorflow on YARN

## Prepare data for training

CIFAR-10 is a common benchmark in machine learning for image recognition. Below example is based on CIFAR-10 dataset.

1) Checkout https://github.com/tensorflow/models/:
```
git clone https://github.com/tensorflow/models/
```

2) Go to `models/tutorials/image/cifar10_estimator`

3) Generate data by using following command: (required Tensorflow installed)

```
python generate_cifar10_tfrecords.py --data-dir=/tmp/cifar-10-data
```

4) Upload data to HDFS

```
hadoop fs -put /tmp/cifar-10-data/ /tmp/
```

**Please note that:**

a. All following examples are using ```/tmp/cifar-10-jobdir``` as snapshot directory for training, so suggest to run:
```
hadoop fs -rmr /tmp/cifar-10-jobdir
```
to cleanup snapshot between runs.

b. YARN service doesn't allow multiple services with the same name, so please run following command
```
yarn application -destroy <service-name>
```
to delete services if you want to reuse the same service name.

## Run Tensorflow jobs

### Use auto generated Yarnfile and run job

With the help of custom python scripts, user can run Tensorflow jobs easily by auto creating Yarnfile.

Please refer to [Run Tensorflow Job Using Helper Script](RunTensorflowJobUsingHelperScript.md) for more details. This is also recommended approach.

### Use raw YARN service spec to run job

This is useful if you want to understand things happening behind-the-scene. But this is not recommended for end-user to use.

Please refer to [Run Tensorflow Job Using Raw Native Service Spec](RunTensorflowJobUsingNativeServiceSpec.md) for more details.

## Outputs

Sample output of master:
```
...
allow_soft_placement: true
, '_tf_random_seed': None, '_task_type': u'master', '_environment': u'cloud', '_is_chief': True, '_cluster_spec': <tensorflow.python.training.server_lib.ClusterSpec object at 0x7fe77cb15050>, '_tf_config': gpu_options {
  per_process_gpu_memory_fraction: 1.0
}
...
2018-05-06 22:29:14.656022: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job master -> {0 -> localhost:8000}
2018-05-06 22:29:14.656097: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> ps-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:29:14.656112: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> worker-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:29:14.659359: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:8000
...
INFO:tensorflow:Restoring parameters from hdfs://default/tmp/cifar-10-jobdir/model.ckpt-0
INFO:tensorflow:Evaluation [1/625]
INFO:tensorflow:Evaluation [2/625]
INFO:tensorflow:Evaluation [3/625]
INFO:tensorflow:Evaluation [4/625]
INFO:tensorflow:Evaluation [5/625]
INFO:tensorflow:Evaluation [6/625]
...
INFO:tensorflow:Validation (step 1): loss = 1220.6445, global_step = 1, accuracy = 0.1
INFO:tensorflow:loss = 6.3980675, step = 0
INFO:tensorflow:loss = 6.3980675, learning_rate = 0.1
INFO:tensorflow:global_step/sec: 2.34092
INFO:tensorflow:Average examples/sec: 1931.22 (1931.22), step = 100
INFO:tensorflow:Average examples/sec: 354.236 (38.6479), step = 110
INFO:tensorflow:Average examples/sec: 211.096 (38.7693), step = 120
INFO:tensorflow:Average examples/sec: 156.533 (38.1633), step = 130
INFO:tensorflow:Average examples/sec: 128.6 (38.7372), step = 140
INFO:tensorflow:Average examples/sec: 111.533 (39.0239), step = 150
```

Sample output of worker:
```
, '_tf_random_seed': None, '_task_type': u'worker', '_environment': u'cloud', '_is_chief': False, '_cluster_spec': <tensorflow.python.training.server_lib.ClusterSpec object at 0x7fc2a490b050>, '_tf_config': gpu_options {
  per_process_gpu_memory_fraction: 1.0
}
...
2018-05-06 22:28:45.807936: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job master -> {0 -> master-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:28:45.808040: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> ps-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:28:45.808064: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> localhost:8000}
2018-05-06 22:28:45.809919: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:8000
...
INFO:tensorflow:loss = 5.319096, step = 0
INFO:tensorflow:loss = 5.319096, learning_rate = 0.1
INFO:tensorflow:Average examples/sec: 49.2338 (49.2338), step = 10
INFO:tensorflow:Average examples/sec: 52.117 (55.3589), step = 20
INFO:tensorflow:Average examples/sec: 53.2754 (55.7541), step = 30
INFO:tensorflow:Average examples/sec: 53.8388 (55.6028), step = 40
INFO:tensorflow:Average examples/sec: 54.1082 (55.2134), step = 50
INFO:tensorflow:Average examples/sec: 54.3141 (55.3676), step = 60
```

Sample output of ps:
```
...
, '_tf_random_seed': None, '_task_type': u'ps', '_environment': u'cloud', '_is_chief': False, '_cluster_spec': <tensorflow.python.training.server_lib.ClusterSpec object at 0x7f4be54dff90>, '_tf_config': gpu_options {
  per_process_gpu_memory_fraction: 1.0
}
...
2018-05-06 22:28:42.562316: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job master -> {0 -> master-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:28:42.562408: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job ps -> {0 -> localhost:8000}
2018-05-06 22:28:42.562433: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:215] Initialize GrpcChannelCache for job worker -> {0 -> worker-0.distributed-tf.root.tensorflow.site:8000}
2018-05-06 22:28:42.564242: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:316] Started server with target: grpc://localhost:8000
```
