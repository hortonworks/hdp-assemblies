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

## Using raw YARN native service spec to run job

Yarnfile is a normal JSON file, typically you should save the Yarnfile to a local file and use following command to run:

```
yarn app -launch distributed-tf <path-to-saved-yarnfile>
```

Or you can use curl to post the Yarnfile.

```
hadoop fs -rmr /tmp/cifar-10-jobdir;
yarn application -destroy distributed-tf;
curl --negotiate -u: -H "Content-Type: application/json" \
  -X POST http://<RM-host>:8088/app/v1/services -d '... content of Yarnfile...'
```

## Example Yarnfiles

### Generate Dockerfiles

Please refer to [Dockerfile for running on Tensorflow on YARN](Dockerfile.md) for more details.

### Single node Tensorflow (with GPU and access Kerberorized HDFS)

```
{
    "name": "single-node-tensorflow",
    "version": "1.0.0",
    "components": [
        {
            "artifact" : {
              "id" : <docker-image-name>,
              "type" : "DOCKER"
            },
            "name": "worker",
            "dependencies": [],
            "resource": {
                "cpus": 1,
                "memory": "4096",
                "additional" : {
                  "yarn.io/gpu" : {
                    "value" : 2
                   }
                }
            },
            "launch_command": "export HADOOP_HDFS_HOME=/hadoop-3.1.0; export HADOOP_HOME=; export HADOOP_YARN_HOME=; export HADOOP_CONF_DIR=/etc/hadoop/conf; export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre/; export CLASSPATH=\\`\\$HADOOP_HDFS_HOME/bin/hadoop classpath --glob\\`; export LD_LIBRARY_PATH=\\$LD_LIBRARY_PATH:/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/; cd /test/cifar10_estimator  && ls -l && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --num-gpus=1 --train-batch-size=16 --train-steps=40000",
            "number_of_containers": 1,
            "run_privileged_container": false
        }
    ],
    "kerberos_principal" : {
      "principal_name" : "test-user@EXAMPLE.COM",
      "keytab" : "file:///etc/security/keytabs/test-user.headless.keytab"
    }
}
```

### Distributed Tensorflow (with CPU and access non-Kerberorized HDFS)

#### Generate TF_CONFIG

```TF_CONFIG``` is an environment variable which passes training parameters to Tensorflow. It is widely used to run distributed Tensorflow training.

Here's an example of ```TF_CONFIG```

```
{
   "cluster":{
      "master":[
         "<master-host>:<port>"
      ],
      "ps":[
         "<ps-host-0>:<port>",
         "<ps-host-1>:<port>",
         "<ps-host-2>:<port>"
         ...
      ],
      "worker":[
         "<worker-host-0>:<port>",
         "<worker-host-1>:<port>",
         "<worker-host-2>:<port>"
         ...
      ]
   },
   "task":{
      "type": "worker",
      "index": 0
   },
   "environment":"cloud"
}
```

It includes two parts, the first is ```cluster```. ```cluster``` is a collection of endpoints of all roles of a Tensorflow job. Roles include:

- ```ps```: saves the parameters among all workers. All workers can read/write/update the parameters for model via ps. As some models are extremely large the parameters are shared among the ps (each ps stores a subset).
- ```worker```: does the training.
- ```master```: basically a special worker, it does training, but also restores and saves checkpoints and do evaluation.

```cluster``` part is identical to all roles of a given Tensorflow job.

(Description of these roles copied from https://github.com/tensorflow/models/tree/master/tutorials/image/cifar10_estimator)

The second is ```task```, which describes role of the launched process, which is different for different roles. For example, if ```task``` is specified to:
```
   "task":{
      "type": "worker",
      "index": 0
   }
```
The launched instance will use: ```<worker-host-0>:<port>``` as endpoint.

Following script can be used to generate ```TF_CONFIG```:

```
import sys
def get_component_array(name, count, hostname_suffix):
    component = "\\\\" + '\\"' + name + "\\\\" + '\\":'
    component_names = '['
    for i in xrange(0, count):
        component_names = component_names + "\\\\" + '\\' + '"' + name + "-" + str(i) + hostname_suffix + "\\\\" + '\\"'
        if i != count - 1:
            component_names = component_names + ','
    component_names = component_names + ']'
    return component + component_names
def get_key_value_pair(name, keys, values, count):
    block_name = "\\\\" + '\\"' + name + "\\\\" + '\\":'
    block_values = ''
    if count == 1:
        block_values = block_values + '\\' + "\\\\" + '"' + values[0] + "\\\\" + '\\"'
        return block_name + block_values
    block_values = '{'
    for i in xrange(0, count):
        block_values = block_values + "\\\\" + '\\' + '"' + keys[i] + "\\\\" + '\\"' + ':' +  values[i]
        if i != count - 1:
            block_values = block_values + ','
    block_values = block_values + '}'
    return block_name + block_values
# Generate TF_CONFIG from username and domain name. Use this to create an ENV variable which could be used as env in native service spec.
if len (sys.argv) != 6 :
    print "Usage: python generate_tf_config.py <username> <domain_name> <service_name> <num_workers (exclude master)> <num_ps>"
    sys.exit (1)
username = sys.argv[1]
domain =  sys.argv[2]
servicename = sys.argv[3]
num_worker = int(sys.argv[4])
num_ps = int(sys.argv[5])
hostname_suffix = "." + servicename + "."+ username + "." + domain + ":8000"
cluster = '"{' + "\\\\" + '\\"cluster' + "\\\\" + '\\":{'
master = get_component_array("master", 1, hostname_suffix) + ","
ps = get_component_array("ps", num_ps, hostname_suffix) + ","
worker = get_component_array("worker", num_worker, hostname_suffix) + "},"
component_name = "\\\\" + '\\"' + "${COMPONENT_NAME}" + "\\\\" + '\\"'
component_id = "${COMPONENT_ID}"
task = get_key_value_pair("task", ["type", "index"], [component_name, component_id], 2) + ","
env = get_key_value_pair("environment", "", ["cloud"], 1) + '}"'
print '"{}"'.format("TF_CONFIG"), ":" , cluster, master, ps, worker, task, env
```

Running

```
python path/to/saved/python-file <user_name> example.com distributed-tf 10 3
```

Generates ```TF_CONFIG``` for given user_name, domain name at example.com (which is same as ```hadoop.registry.dns.domain-name``` in ```yarn-site.xml```), service name is ```distributed-tf```. 10 workers (exclude master), and 3 parameter servers. The python script can be tailored according to your cluster environment.

#### Yarnfile
```
{
    "name": "distributed-tf",
    "version": "1.0.0",
    "components": [
        {
            "name": "master",
            "dependencies": [],
            "resource": {
                "cpus": 1,
                "memory": "4096"
            },
            "artifact" : {
              "id" : <docker-image-name>,
              "type" : "DOCKER"
            },
            "launch_command": "export HADOOP_HDFS_HOME=/hadoop-3.1.0; export HADOOP_HOME=; export HADOOP_YARN_HOME=; export CLASSPATH=\\`\\$HADOOP_HDFS_HOME/bin/hadoop classpath --glob\\`; export LD_LIBRARY_PATH=\\$LD_LIBRARY_PATH:/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/; cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --train-steps=10000 --num-gpus=0 --eval-batch-size=16 --train-batch-size=16 --sync",
            "number_of_containers": 1,
            "run_privileged_container": false
        },
        {
            "name": "worker",
            "dependencies": [],
            "resource": {
                "cpus": 1,
                "memory": "4096"
            },
            "artifact" : {
              "id" : <docker-image-name>,
              "type" : "DOCKER"
            },
            "launch_command": "export HADOOP_HDFS_HOME=/hadoop-3.1.0; export HADOOP_HOME=; export HADOOP_YARN_HOME=; export CLASSPATH=\\`\\$HADOOP_HDFS_HOME/bin/hadoop classpath --glob\\`; export LD_LIBRARY_PATH=\\$LD_LIBRARY_PATH:/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/; cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --train-steps=10000 --num-gpus=0 --eval-batch-size=16 --train-batch-size=16 --sync",
            "number_of_containers": 1,
            "run_privileged_container": false
        },
        {
            "name": "ps",
            "dependencies": [],
            "resource": {
                "cpus": 1,
                "memory": "2048"
            },
            "artifact" : {
              "id" : <docker-image-name>,
              "type" : "DOCKER"
            },
            "launch_command": "export HADOOP_HDFS_HOME=/hadoop-3.1.0; export HADOOP_HOME=; export HADOOP_YARN_HOME=; export CLASSPATH=\\`\\$HADOOP_HDFS_HOME/bin/hadoop classpath --glob\\`; export LD_LIBRARY_PATH=\\$LD_LIBRARY_PATH:/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/; cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --num-gpus=0",
            "number_of_containers": 1,
            "run_privileged_container": false
        }
    ],
    "configuration": {
        "properties": {},
        "env": {
            "TF_CONFIG" : <Generated TF_CONFIG from script>,
            "HADOOP_CONF_DIR" : "/etc/hadoop/conf",
            "JAVA_HOME" : "/usr/lib/jvm/java-8-openjdk-amd64/jre/",
            "YARN_CONTAINER_RUNTIME_DOCKER_CONTAINER_NETWORK": "bridge"
        }
    }
}
```
