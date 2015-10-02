---
layout: post
comments: true
title:  "Consolidate AWS servers by using Docker"
date:   2015-07-18 15:11:09
categories: docker
---

## Story

Amazon Web Services is good and I love it because it's easy to use, flexible and
cost-effective.

But, at a point, at least for some companies, AWS becomes too clunky and too
expensive.

That's what is happening to BlackCitrus.

At this moment, we have four onging rails projects and they all have their own
staging server, namely four EC2 instances (m3.medium).
And they are all running 24x7.
However, the utilization of those staging servers are really not much at all.

We recently initialized another rails project and we decided to limit our spend
on AWS. However we still want all our applications are running 24x7 and
don't want to downsize the instance because of the poor performanace of small
EC2 instances.

## Docker to the rescue

### What is Docker

#### Software Container

Docker is a tool that can package an application and its dependencies into a
virtual container that can run on any Linux server.

In a way, Docker is a bit like a virtual machine. But unlike a virtual machine,
rather than creating a whole virtual operating system, Docker allows
applications to use the same Linux kernel as the system that they're running on
and only requires applications be shipped with things not already running on the
host computer.

### Why Docker

#### 1. Easy Setup

What you'll need is just one linux server runs Docker and a Dockerfile for each
of your projects. Maybe an extra docker-compose.yml file if your application
requires multiple containers.

#### 2. Portability

Docker containers are independent from the host version of Linux Kernel,
platform distribution or deployment model. They can be easily transfered to
another machine that runs Docker, and executed there without compatibility
issues.

#### 3. App Isolation

By using containers, resources can be isolated, services restricted, and
processes provisioned to have an almost completely private view of the operating
system with their own process ID space, file system structure, and network
interfaces. Docker lets you safely run multiple applications on the same cloud
instance by abstracting and isolating their dependencies.

#### 4. Lightweight, minimal overhead, high utilization

Docker images are typically very small and docker containners also require much
less overhead - really no more expensive than a process. You can easily spin up
1000's of docker containers on an ordinary PC. By deploying multiple Dockerized
applications on to a single cloud instance, you can get much closer to achieving
100% utilization.

### How docker changes things

#### Before Docker

1. Create an new EC2 intance every time when you initialize an new project
2. Install dependencies (i.e. ruby, mysql, redis). Nomarlly we do this by using
   [ Chef ]( https://www.chef.io/ ).
3. Setup the deployment tool (i.e. [Capistrano](http://capistranorb.com/))

#### After Docker

1. Create an EC2 instance (only once) and install Docker
2. Dockerize the application by adding a Dockerfile and docker-compose.yml (if
   required)

## What's Next
The new rails app is now running smoothly in a docker container on a m3.medium
EC2 instance. And we'll migrate all other existing apps onto the same instance
soon in the future.

Why pay the money for 5 (or more) instances if you can get things done in just
one intance?

