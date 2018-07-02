# Containerized HTTPD with Proxy Service Example
This repo contains the Yarnfile, Docker image assets, and instructions on building and running a containerized load balanced HTTPD example.

## Prerequesites
For this example to work, the hdp-assemblies/httpd image must be included in `docker.trusted.registries` in `container-executor.cfg`.

It is expected that Docker, YARN containerization, and the YARN Registry DNS server are installed and configured. The choice of network may require changes to the supplied Yarnfile. Update the `Dockerfile` and `Yarnfile` as needed for your environment.

## Build the image
Checkout the repo and run the following:
```
cd httpd && ./build_image.sh
```

## Upload the HTTPD Proxy Configuration to HDFS:
```
hdfs dfs -copyFromLocal ${HADOOP_YARN_HOME}/share/hadoop/yarn/yarn-service-examples/httpd/httpd-proxy.conf .
```

## Launch the app through the YARN Services API:
```
yarn app -launch httpd httpd/Yarnfile
```
