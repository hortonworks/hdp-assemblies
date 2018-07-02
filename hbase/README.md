# Containerized HBase Service Example
This repo contains the Yarnfile, Docker image assets, and instructions on building and running a containerized HBase example service.

## Prerequesites
For this example to work, the hdp-assemblies/hbase image must be included in `docker.trusted.registries` in `container-executor.cfg`.

It is expected that Docker, YARN containerization, and the YARN Registry DNS server are installed and configured. The choice of network may require changes to the supplied Yarnfile. Update the `Dockerfile` and `Yarnfile` as needed for your environment.

## Build the image
Checkout the repo and run the following:
```
cd hbase && ./build_image.sh
```

##Upload Hadoop configurations to HDFS:
```
hadoop fs -copyFromLocal /etc/hadoop/conf/core-site.xml .
hadoop fs -copyFromLocal /etc/hadoop/conf/hdfs-site.xml .
```

##Launch the app through the YARN Services API
```
yarn app -launch hbase hbase/Yarnfile
```
