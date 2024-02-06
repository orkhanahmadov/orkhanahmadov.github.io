---
layout: post
title: Using pre-built Docker images with Laravel Vapor to reduce deployment time
---

Vapor's custom Docker runtimes are great to customize environment dependencies, adding PHP extensions, changing configuration, etc.

But doing all of this requires building the deployment image from scratch, on every single deployment.
If you are using tools like GitHub Actions to automate the deployment, increased deployment time will eat up your GitHub Actions minutes quickly.

<!--more-->

Perfect example to this would be using the `imagick` PHP extension. Adding `imagick` extension to Dockerfile requires building it from source, which can take a lot of time.

Consider the following `production.Dockerfile`:

```Dockerfile
FROM laravelphp/vapor:php82

RUN apk add --update --no-cache autoconf g++ imagemagick-dev \
    && pecl install imagick \
    && docker-php-ext-enable imagick 

COPY . /var/task
```

On this post I will show how to use pre-built Docker images with Laravel Vapor and use it as base image for your application's deployments.

We'll cover the following steps:

- Building the base Docker image
- Pushing the base Docker image to a container registry. It could be Docker Hub, GitHub Container Registry, AWS ECR, etc. We'll use AWS ECR in this example.
- Using the base Docker image in your Laravel Vapor project

## Building the base Docker image

Create `Dockerfile` in the root of your project with similar content with the `production.Dockerfile` but without the `COPY` command:

```Dockerfile
FROM laravelphp/vapor:php82

RUN apk add --update --no-cache autoconf g++ imagemagick-dev \
    && pecl install imagick \
    && docker-php-ext-enable imagick 
```

The goal is to build a Docker image with all the dependencies, extensions and configurations necessary without adding the project files.

Build this Docker image with a tag. I'm using `vapor-base` in this example:

```shell
docker build -t vapor-base .
```

## Pushing the base Docker image to AWS ECR

Create a new repository in AWS ECR. You can choose either public and private repository type.

Once you create the repository, repository URI will look like this:

```shell
1234567890.dkr.ecr.AWS-REGION-1.amazonaws.com/vapor-base
```

Replace `1234567890` with your AWS account ID and `AWS-REGION-1` with your AWS region.

Use AWS CLI, login to your account and retrieve the authentication token to authenticate your Docker client to the registry:

```shell
aws ecr get-login-password --region AWS-REGION-1 | docker login --username AWS --password-stdin 1234567890.dkr.ecr.AWS-REGION-1.amazonaws.com
```

Tag the previously built Docker image with the repository URI:

```shell
docker tag vapor-base:latest 1234567890.dkr.ecr.AWS-REGION-1.amazonaws.com/vapor-base:latest
```

Lastly, push the Docker image to the AWS ECR repository:

```shell
docker push 1234567890.dkr.ecr.AWS-REGION-1.amazonaws.com/vapor-base:latest
```

## Using the base Docker image in your Laravel Vapor project

Now that we have the base Docker image in AWS ECR, we can use it in our Laravel Vapor project. Open `production.Dockerfile` and do the following changes:

```Dockerfile
FROM 1234567890.dkr.ecr.AWS-REGION-1.amazonaws.com/vapor-base:latest 

COPY . /var/task
```

As you can see here, with this kind of setup we are no longer building the base Docker image on every deployment.
Instead, we are using our pre-built Docker image as a base image and just adding the project files on top of it for every deployment.

This will significantly reduce the deployment time.