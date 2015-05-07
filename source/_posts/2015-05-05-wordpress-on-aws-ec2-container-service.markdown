---
layout: post
title: "WordPress on AWS EC2 Container Service"
date: 2015-05-05 21:47:04 -0500
comments: true
categories:
  - WordPress
  - Deployment
  - AWS
---

This week I had the task of starting up a new WordPress site for a start up
wedding industry company. Since I like tinkering and wanted more experience
working with AWS tools, I decided that would be the perfect place to host it.
Shared hosting is boring and I've set it up multiple times before. Hosting one
small WordPress site using an entire machine is probably overkill, so I decided
to make use of Amazons new [Docker](http://docker.io) service,
[ECS](http://aws.amazon.com/ecs/), and set up a container I could start on a
machine. Having no experience with Docker or with launching a WP site in AWS,
here is how I got the site running and somewhat optimized. This may be a little
naive, but it was a great learning experience. This is not a step by step guide,
but a general idea of what it takes to actually get a simple blog running in a
complicated infrastructure.

<!-- more -->

##AWS EC2 Container Service (ECS)

Recently Amazon launched their new [ECS](http://aws.amazon.com/ecs/) service
that allows you to run multiple docker containers on a cluster of EC2 instances
and they manage it all for you (kind of). Sounds like a good use of resources,
especially for a small WordPress blog that might not draw a lot of traffic.
Sounds promising.

When you first visit the ECS console you are presented with a nice quick setup
wizard. The wizard walks you through creating a task, a service, and spinning up
an EC2 instance along with some IAM roles and a load balance. This is nice way
to start off without knowing what's happening in the background. What it's
really doing is setting up a CloudFormation stack that includes the resources
you need to get running and play around with the features. Pretty nifty, but I
have not been able to get back to the wizard unless I deleted my entire cluster,
so it seems it's a one shot type deal and then you are on your own to create new
cluster.

###Task Definition

A task definition is a collection of containers that work together to do some
work and complete an action. Once the task is setup, you cannot modify it or
delete it. The only thing you can do is to create a new version of the task that
modifies the previous. A little annoying if you are experimenting, but it is a
new platform.

###Cluster

A cluster is a collection of services, tasks, and ECS instances. A service
lets you run and maintain running a specific number of tasks on the cluster. You
can also run a task once, or schedule task to run on regular bases. But for the
purpose of running WordPress we will need to use a service to keep the WordPress
task running.

##Starting up.

So the first thing that we need is to build or find a docker container that will
allow us to run our site. I found a ready WordPress official install to
experiment with
[here](https://github.com/docker-library/docs/tree/master/wordpress). The
container lets us set up all the variables we need to run the blog in the ECS
cluster.

So docker containers are not persistent and the data they contain will be gone
in a blink so we need to set up a few things to make it persist. Fortunately AWS
provides all of these tools for us.

First set up a MySQL database and database user to have a place to store the
data. We will also need to set up a place to store and serve the content from,
but I'll get to that later.

###Defining WordPress task

The first thing to do is to define a task to run the container. Go to the ECS
Console / Task Definitions and create a new task and enter the definition in
JSON:

```json WordPress ECS Task Definition
{
  "containerDefinitions": [
    {
      "portMappings": [
        {
          "hostPort": 80,
          "containerPort": 80
        }
      ],
      "environment": [
        {
          "name": "WORDPRESS_DB_USER",
          "value": "USER_NAME_FOR_RDS"
        },
        {
          "name": "MYSQL_PORT_3306_TCP",
          "value": "tcp://RDS_ADDRESS:3306"
        },
        {
          "name": "WORDPRESS_DB_NAME",
          "value": "RDS_DATABASE_NAME"
        },
        {
          "name": "WORDPRESS_DB_PASSWORD",
          "value": "RDS_DATABASE_PASSWORD"
        },
        {
          "name": "WORDPRESS_DB_HOST",
          "value": "RDS_ADDRESS"
        }
      ],
      "essential": true,
      "mountPoints": [
        {
          "containerPath": "/var/www/html",
          "sourceVolume": "wordpress-vol",
          "readOnly": false
        }
      ],
      "memory": 256,
      "name": "wordpress",
      "cpu": 128,
      "image": "wordpress"
    }
  ],
  "volumes": [
    {
      "host": {
        "sourcePath": "/ecs/webdata/wordpress"
      },
      "name": "wordpress-vol"
    }
  ],
  "family": "wordpress"
}
```

This will define one container running WordPress that will connect to a defined
RDS MySQL database with the user/password combination you created previously. It
will also set up a persisted data volume that will map the WordPress installation
to `/ecs/webdata/wordpress` path on the EC2 instance. That way the data will
persist between the container being stopped and started. So that's the definition
for the container.

###Creating ECS Cluster

Go to EC2 Container Service / Clusters and click 'Create Cluster'. That's it,
this will create an empty cluster that you need to build up.

###Define Service

Create a service that will run the WordPress task that we created earlier. The
settings are pretty simple: Task Definition, Cluster, Service Name, Number of
Tasks, and Elastic Load Balancing. Most of those are self explanatory. We only
need on task to run. The ELB, if you run through the initial set up it will
automatically start one for you, otherwise you have to create an ELB and point
it to this group.

###ECS Instances

The last thing you will need is to setup is a EC2 instance to actually run this.
I'll admit, this was setup automatically for me when I build the initial ECS
cluster using the wizard. To create this instance, you will need to tag it with
the correct cluster instance so it can be found by ECS. Simple enough, but it's
probably better to put it behind an auto scaling group of 1 so that if the
machine ever dies it will spin right back up.

###Start the service

Once you have the infrastructure in place and setup all of the permission needed
to talk to RDS, you can activate the service and then visit the ELB public DNS
address to setup your blog.

But before you do that, I suggest you setup the routing to alias the ELB to a
domain name. I used Route 53, but feel free to use whatever service you have.
Then visit the address to run the installation and get WordPress up and running.

One thing I found is that the ELB cannot route to the site before WordPress is
installed, because it redirects to a different port, so it thinks the site is
not running. To fix this, visit the public address of the EC2 instance and
install WordPress. Once the ELB is reporting a healthy service status, change
the WordPress setting for site address so that everything is showing up
correctly.

##Additional Notes

Congrats you now have a running WordPress instance, but it's not very stable. If
the EC2 instance dies, you loose your content. This is bad. Let's try and fix
that a bit.

###Set up CloudFront and S3

Use W3 Total Cache plug in to set up a CDN using CloudFront and S3. This will
upload your files to S3 and host them through the CloudFront CDN. I'm not
providing detailed instructions, but make sure that you set up CloudFront to
forward headers.

###Create an AMI Image of the EC2 instance

Create an AMI image so you have a backup of the current state of the image. This
will help when the server crashes and you need to restore the files that are not
part of the W3 Total Cache CDN.

So once you get all of that running you're still not fully protected. EC2
instances could die, it doesn't happen very often, but it can happen. I'm still
not sure how I'll protect against that so that there is very limited down time,
but I'll need to figure out a good way to store the WordPress configuration and
automatically load it to the AMI. This should all be doable with CloudFormation,
but one step at a time.
