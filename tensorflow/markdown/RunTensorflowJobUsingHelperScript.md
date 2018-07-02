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

# Run Tensorflow Jobs Using Helper Script

## Prerequisites
1) Sufficient permissions (authorization and authentication, like `kinit`) are needed for the user (in case of secure cluster) to run `submit_tf_job.py` script as it also supports submitting Tensorflow service directly to YARN.

2) User could be use `--input_spec` argument to specify a sample spec file as a template. This package has `example_tf_job_spec.json` sample spec file and user could edit this file to specify resources per component (such as memory, cpu, gpu etc) and kerberos specification if needed.

## Setup presetup-tf.sh template
1) Rename `presetup-tf.sh_template` to `presetup-tf.sh`

2) In `presetup-tf.sh`

    - Update valid `HADOOP_HDFS_HOME`. This should point to `HADOOP_HDFS_HOME` **inside the docker image**.

    - Update `JAVA_HOME` as per the environment setup. This should point to `JAVA_HOME` **inside the docker image**.

3) Place `presetup-tf.sh` in HDFS under `hdfs://host:port/<tf-job-conf-path>/`.

4) Ensure that `<tf-job-conf-path>` is accessible with correct permission for user.

5) Upload core-site.xml, hdfs-site.xml to `<tf-job-conf-path>`.

6) (when security is enabled) Upload krb5.conf to `<tf-job-conf-path>`.

## Run `submit_tf_job.py` to submit Tensorflow job to YARN

User could run below command to submit Tensorflow job to YARN or to generate valid Yarnfile for the job.

`python submit_tf_job.py --remote_conf_path <tf-job-conf-path> --input_spec <INPUT_SPEC> --docker_image <DOCKER_IMAGE> --env <ENV> --job_name <JOB_NAME> --user <USER> --domain <DOMAIN> --kerberos`

Detailed argument summary for `submit_tf_job.py` command.

```
optional arguments:
  -h, --help            show this help message and exit
  --remote_conf_path REMOTE_CONF_PATH
                        Remote Configuration path to run TF job should include
                        core-site.xml/hdfs-site.xml/presetup-tf.sh, etc. By
                        default it uses hdfs:///etc/tf-configs
  --input_spec INPUT_SPEC
                        Yarnfile specification for TF job.
  --docker_image DOCKER_IMAGE
                        Docker image name for TF job.
  --env ENV             Environment variables needed for TF job in key=value
                        format.
  --dry_run             When this is not specified (default behavior), YARN
                        service will be automatically submitted. When this is
                        specified, generated YARN service spec will be printed
                        to stdout
  --job_name JOB_NAME   Specify job name of the Tensorflow job, which will
                        overwrite the one specified in input spec file
  --user USER           Specify user name if it is different from $USER (e.g.
                        kinit user)
  --domain DOMAIN       Cluster domain name, which should be same as
                        hadoop.registry.dns.domain-name in yarn-site.xml,
                        required for distributed Tensorflow
  --kerberos            Is this a kerberos-enabled cluster or not
  --verbose             Print debug information
```

Example:
`python submit_tf_job.py --input_spec example_tf_job_spec.json --docker_image tf-gpu:ubuntu-xyz --job_name distributed-tf --user ambari-qa --domain <your_domain_name> --remote_conf_path hdfs:///tf-job-conf/configs`

## Provide `input-spec` file to run Tensorflow jobs

### Run standalone TF job.

```

{
  "name": "standalone-tf",
  "version": "1.0.0",
  "components": [
    {
      "name": "worker",
      "dependencies": [],
      "resource": {
        "cpus": 1,
        "memory": "4096",
        "additional" : {
          "yarn.io/gpu" : {
            "value" : 1
          }
        }
      },
      "launch_command": "python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --train-steps=10000 --num-gpus=1 --eval-batch-size=16 --train-batch-size=16 --sync",
      "number_of_containers": 1
    }
  ],
  "configuration": {
    "properties": {},
    "env": {
    }
  },
  "kerberos_principal" : {
    "principal_name" : "test-user@EXAMPLE.COM",
    "keytab" : "file:///etc/security/keytabs/test-user.headless.keytab"
  }
}

```

Notes:

- `hdfs://host:port/<tf-job-conf-path>/presetup-tf.sh` will be automatically downloaded and mounted to the docker container, it will be executed before invoking `launch_command` of components specified in the spec.
- Component name can be customized (In above example it uses `worker`)
- In `resource` section, you can specify cpu/memory/gpu if you needed.
- Additional environment variables can be specified under `env`. This will be passed to launched docker container process.

### Run distributed TF job.

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
        "memory": "4096",
        "additional" : {
          "yarn.io/gpu" : {
            "value" : 1
          }
        }
      },
      "launch_command": "cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --train-steps=10000 --num-gpus=1 --eval-batch-size=16 --train-batch-size=16 --sync",
      "number_of_containers": 1
    },
    {
      "name": "worker",
      "dependencies": [],
      "resource": {
        "cpus": 1,
        "memory": "4096",
        "additional" : {
          "yarn.io/gpu" : {
            "value" : 1
          }
        }
      },
      "launch_command": "cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --train-steps=10000 --num-gpus=1 --eval-batch-size=16 --train-batch-size=16 --sync",
      "number_of_containers": 1
    },
    {
      "name": "ps",
      "dependencies": [],
      "resource": {
        "cpus": 1,
        "memory": "2048",
        "additional" : {
          "yarn.io/gpu" : {
            "value" : 1
          }
        }
      },
      "launch_command": "cd /test/cifar10_estimator && python cifar10_main.py --data-dir=hdfs://default/tmp/cifar-10-data --job-dir=hdfs://default/tmp/cifar-10-jobdir --num-gpus=1",
      "number_of_containers": 1
    }
  ],
  "configuration": {
    "properties": {},
    "env": {
    }
  },
  "kerberos_principal" : {
    "principal_name" : "test-user@EXAMPLE.COM",
    "keytab" : "file:///etc/security/keytabs/test-user.headless.keytab"
  }
}
```

Notes: (In addition to standalone TF spec)

- For distributed Tensorflow launch spec, `master`, `worker`, `ps` components are mandatory.
- Different value of `num_of_containers` can be specified for `worker` and `ps`.
- `TF_CONFIG` will be automatically generated and insert to spec to launch the job to do distributed training, you don't need to worry about this.

### Addition information about writing Yarnfile

#### Security

YARN Native Services provide an easy way to run services in a secure cluster by providing keytabs and kerberos principal as simple json input parameters.
```
"kerberos_principal" : {
      "principal_name" : <your-principle-name>,
      "keytab" : <path-to-your-keytab>
}
```
User could define the kerberos principal name (eg: test-user@EXAMPLE.COM) and keytab file path from HDFS or local file system. Given these information, user could run service from a secured shell.

#### Choosing correct Docker images for the application
```
"artifact" : {
  "id" : <TODO: docker image name>,
  "type" : "DOCKER"
}
```
Under each service component, user need to provide docker image name so that native service will use this image to launch containers.
User can use `--docker_image` to overwrite whatever defined in the input job spec.

## General Guidelines

1) For many env configurations, extra escape characters are used so that native service could export correct env's as per guideline. An improvement to this is ongoing in YARN-8257.

2) In secure clusters, user need to ensure that app is launched from a secure shell. (e.g. with proper Kerberos token).

## End-to-end example:

### Run Cifar10 distributed Tensorflow training on GPU/security-enabled cluster

#### Launch Command
```
python submit_tf_job.py --remote_conf_path hdfs:///tf-job-conf --input_spec example_tf_job_spec.json --docker_image gpu.cuda_9.0.tf_1.8.0 --job_name distributed-tf-gpu --user tf-user --domain tensorflow.site --kerberos
```

- `docker_image` file could be found under `docker/with-models/ubuntu-16.04/Dockerfile.gpu.cuda_9.0.tf_1.8.0` from Hadoop codebase and we assume docker image is created named as `gpu.cuda_9.0.tf_1.8.0` from this file.
- `input_spec` file could be found under `specs/` and make the necessary edits as needed.
