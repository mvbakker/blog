---
title: "Docker for AWS review"
date: "2018-08-28"
draft: false
slug: docker-for-aws-review
author: "Michiel Bakker"
---

### Introduction
So I stumbled upon this site: [Docker for AWS](https://docs.docker.com/docker-for-aws/why/)

Basically, Docker created a template to deploy a cluster on AWS easily that takes care, among other things, of the loadbalancing and storage, the two architectural topics that Docker usually leaves up to you to resolve and implement correctly. 

### Motivation
A few weeks ago I decided I wanted to create a highly available container platform on AWS. I was studying then for the [Docker Certified Associate](https://success.docker.com/certification) exam and wanted to see what it takes to create such a platform. Additionally, I decided to run the latest [Nextcloud](https://nextcloud.com/) on this platform, all in containers. This seemed like a proper final challenge to get ready for the certification exam. (This is also the reason why I didn't use one of the AWS containers services, as I needed to learn more about Docker).

### Why Docker for AWS
Setting up this platform for hosting an actual application immediately challenged me to think about persistent storage and load-balancing. Why? Because:

* Docker Volumes are generally saved and persist, but they do this locally. If a node fails and Docker Swarm deploys the same container on a different node, the volume that was attached to it on the failed node is not available if the default volume driver is used. A different storage solution for Docker Volumes is needed for a cluster.
* In AWS the IP addresses of the EC2 instances can easily change. Additionally, if you point your DNS-Domain to one EC2 instance and that one fails, the whole application is unreachable.

For load-balancing AWS provides [a great solution](https://aws.amazon.com/elasticloadbalancing), so I wasn't too worried about that. 

For the persistent storage I looked first into other solutions and decided to see if I could install and implement the [REX-Ray volume driver for S3FS](https://rexray.readthedocs.io/en/latest/user-guide/schedulers/docker/plug-ins/aws/#simple-storage-service). This seemed like a very nice solution for the volumes. The storage would be very cheap (0.03ct/GB) compared to EBS (0.12ct/GB) or EFS (0.33ct/GB). Since I would only use it for Nextcloud, a file-sharing platform, it wouldn't have to be very fast anyway.

However, I stumbled upon many issues when trying to install and configure the driver. Additionally, REX-Ray makes use of the [S3FS-fuse plugin](https://github.com/s3fs-fuse/s3fs-fuse/wiki/Fuse-Over-Amazon), so often when trouble-shooting you're being send from the 'closet to the wall' as we say in Dutch. After a few days of figgling and trying to get it to work, I decided not to persue my ambitions to create a persistent storage solution with the REX-Ray S3FS volume driver.

That's when I stumbled upon [Docker for AWS](https://docs.docker.com/docker-for-aws/why/).

This solves the two challanges of load-balancing and persistent storage. 'Unfortunately' it also takes away the fun and learning experience of installing and configuring a whole platform/cluster by yourself. Usually this kind of automation via infrastructure-as-code is what you want, but for learning purposes it would've been nice to install and configure it manually myself.
The CloudFormation TemplateThe 'Docker for AWS' solution is deployed automatically via CloudFormation, the source is shared [here](https://github.com/docker/for-aws) on Github. A template is simplyl available where you will have to fill in the following variables that define the size/architecture of the container platform it will deploy:

* Select the SSH key that you'd want to use
* EC2 instance type for workers and managers (separately)
* Amount of worker and manager instances
* Storage options for the worker and manager instances
* EnableSystemPrune (when enabled the command 'docker system prune' will run everyday automatically)
* Enable CloudWatch logs 

Everything will be configured and deployed by the CloudFormation template then. After this installation you can simply log on to the nodes using the SSH key you supplied and start running your containers/services.  

### The Cloudstor plug-in
One of the nice features of Docker for AWS is the Cloudstor plugin. This plugin offers two different type of volumes:

* Relocatable volumes, backed by EBS
* Shared volumes, backed by EFS

###### Relocatable volumes
Relocatable volumes are backed by EBS, the Elastic Block Storage service provided by AWS. Additionally, you can specify which kind of EBS storage you need (gp2, io1, st1, sc1) depending on your workload. On the background I believe the plugin mounts a volume to the node/instance of the host that the container runs on. When this node/instance fails or the container is moved to another node/instance for another reason, the plugin takes care of the EBS volume to be mounted on the new host node/instance. It might take a while for the volume to be available on the other node/instance though, especially if the node/instance is in a different availablility zone (this might cause troubles with your orchestrator). Moreover, it is not possible for containers to have both access to the same volume if the containers are running on different nodes/instances.

###### Shared volumes
Shared volumes are backed by EFS, the Elastic File System service provided by AWS. Each node/instance in the 'Docker for AWS' cluster will be mounted to the EFS service and therefore each volume created on the EFS storage will be available on all nodes/instances at all times. This makes it also possible for containers that run on different nodes/instances to use the same volumes (although I would not recommend this). Containers can however change host much more quickly.

All in all, this gives a multi-flavoured out-of-the-box solution for persistent storage, nice job!

Note that there is no option with this plugin to use S3 buckets. This is a pity, although I wonder in general whether this would be a proper solution. It could well be that volumes on S3 buckets are not fast enough for Docker to work with and will therefore cause a lot of issues. (Luckily I found that on Nextcloud itself you can install a 3rd-party storage plugin, that allows you to mount S3 buckets to Nextcloud from an application point of view, instead of on the infrastructure layer, therefore I could still make sure that all the large files would be stored on cheap storage and not on expensive EBS or EFS).

### Load-Balancing
The load-balancer is configured to listen to port 80 and 443 (only). When you open a port on the cluster via a container or a service, than the load-balancer is able to reach it. If you are planning on only running one web-service on your cluster, than this will be sufficient. Otherwise you'll have to install something like [Traefik](https://traefik.io/). 

Traefik will ensure that you can route multiple services over port 80 and 443, while traefik takes care of the internal load-balancing within the container platform. Pascal Andy created a nice [Traefik tutorial](https://pascalandy.com/blog/traefik-demo-docker-stack-and-play-with-docker/) to deploy it quickly on http://labs.play-with-docker.com/! Additionally, I created a small [stack-file to deploy Treafik on the Docker for AWS](https://github.com/mvbakker/Traefik_setup) platform.

Additionally Docker for AWS provides you to make use of the [free SSL certificate service](https://aws.amazon.com/certificate-manager/). This makes secure web-services very easy. I used DNS validation, where you basically have you domain provider configured to have one CNAME-record pointing from (one of your sub)domain(s) to the url of the ELB of the Docker for AWS cluster and one CNAME-record for validation. Note: make sure to [add the trailing dot](http://www.dns-sd.org/trailingdotsindomainnames.html) to the validation address that AWS provides you. This was a stupid mistake and valuable lesson, please use it to your advantage.

### Costs
Finally I destroyed the Docker for AWS cluster after a month. The bill was pretty high, as I wanted to have my Nextcloud available 24/7. I set up Docker for AWS with two micro instances, one manager and one worker (which was an overkill probably, but you simply need at least two instances to have a cluster). I used the EFS storage for the plugin, this costs only 14 cents for the whole month, since this was only used for the storage needed by the application to function. 

The costs of running Docker for AWS with two t2.micro instances for ~750 hours costs in total: $57.66

The biggest cost was to have to load-balancer in place ($21.46 for 766 hours and 0.292GB traffic).

Eventhough Docker for AWS is a very nice out-of-the-box solution to deploy a container cluster on AWS, the costs were simply too high for my small application. Other than that, it all worked pretty sweet!
