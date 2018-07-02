# HDP Example Assembly Applications
The Hortonworks Data Platform 3.0 adds support for leveraging YARN for Docker container orchestration. 

## Example Services
This repo is intended to provide a place to share Docker image assets and Yarnfiles as a community. In an effort to jump start efforts, several examples have been included. Please see the individual example directory for more details.

## Distributing the Example Docker Images
*IMPORTANT* - If an image is stored in a registry or Docker hub, updates must be made to `docker.trusted.registries` in `container-executor.cfg` to reflect the location of the image.

Docker provides several options to aid in the distribution of images. These include Docker hub and a Docker private registry. Docker hub provides the ability to store and distribute both public and private images, but requires access to the internet. A Docker private registry allows for running an internally hosted services for storing and distributing images.

For detail on Docker Hub, please see [this overview|https://docs.docker.com/docker-hub/].

## Using a Docker Private Registry
### Starting the Registry
# Designate a server for the docker and start the registry.
# Designate a server in the cluster for use by the Docker registry. Minimal resources are required, but sufficient disk space is needed to store the images and metadata. Docker must be installed and running.
# Start the registry:
```
docker run -d -p 5000:5000 --name registry registry:2
```
# Optional: By default, data will only be persisted within the container. If you would like to persist the data on the host, you can customize the bind mounts using the -v option. 
```
docker run -d -p 5000:5000 -v /host_registry_path:/var/lib/registry --name registry registry:2
```

### Building the Example Images
See the README for the example images in this repository for details on how to build the images.

### Populating the Private Registry
Once the images are built on the local system, tag and push them to the private registry. Follow the steps below for each image. Note that the hostname, image name, and image tag below needs to be replaced in the commands below. 

# Tag the images
```
docker tag hdp-assemblies/<image_name>:<image_tag> <registry_hostname>:5000/hdp-assemblies/<image_name>:<image_tag>
```
# Push the images into the private registry
```
docker push <registry_hostname>:5000/hdp-assemblies/<image_name>:<image_tag>
```
# Configure Docker to allow pulling from this insecure registry. Modify `/etc/docker/daemon.json` on all nodes in the cluster to include the following configuration options.
```
{
 "live-restore" : true,
 "insecure-registries" : ["<hostname>:5000"]
}
```
# Restart Docker on all nodes.
