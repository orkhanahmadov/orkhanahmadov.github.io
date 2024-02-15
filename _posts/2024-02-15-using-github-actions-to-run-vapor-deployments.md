---
layout: post
title: Using GitHub Actions to run Vapor deployments
---

In the previous post, we discussed how to use pre-built Docker images with Laravel Vapor to reduce deployment time.
There I also mentioned using GitHub Actions to automate the deployment.

In this post, I want to showcase some examples, including one that uses a private AWS ECR repository as a base image to deploy Vapor applications using GitHub Actions.

<!--more-->

First, create a GitHub action workflow file in the `.github/workflows` directory.
For example, `deploy.yml`. If you have more than one environment, you can create separate workflow files for each environment.
`deploy-production.yml` for production, `deploy-staging.yml` for staging, etc.

One important note, before everything else, in the examples below whenever you see `${secrets.X}` make sure to replace `${` with <img src="https://i.imgur.com/4vdxMuO.png" height="18" style="display:inline;margin:0"/> and `}` with <img src="https://i.imgur.com/oZXvoRU.png" height="19" style="display:inline;margin:0"/> in your actual workflow file.
GitHub automatically removes any mentioned variables when building the site, so I had to place them without variable wrappers to avoid this.

That's actually really smart from GitHub to protect against accidental exposure of secrets!

## Basic deployment using GitHub Actions without custom Docker image

Here's a basic example of a GitHub Actions workflow file that deploys a Laravel Vapor application to the production environment without using anything custom:

```yaml
name: Deploy Production

env:
  VAPOR_API_TOKEN: ${secrets.VAPOR_API_TOKEN}

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install Yarn dependencies
        run: yarn install

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.3

      - name: Install Composer dependencies
        run: composer install --no-dev

      - name: Deploy Environment
        run: ./vendor/bin/vapor deploy production
```

Here we give the action a name, and instruct it to run on every push to the `master` branch, which includes both direct pushes and pull request merges to `master`.

The important part here is the `VAPOR_API_TOKEN` secret.
First, go to your [Vapor Profile, API Token](https://vapor.laravel.com/app/account/api-tokens) and generate a new token.
Then go to your repository "Settings" > "Secrets and variables" > "Actions" on GitHub, and add a new repository secret with the name `VAPOR_API_TOKEN` and the value of the generated token.

Else should be self-explanatory:

- We install Node.js dependencies using `yarn`. Use NPM or whatever you use, or delete this step completely if your project does not need to build assets on deployment
- We install PHP dependencies using `composer`
- Finally, we deploy the application using `vapor deploy production`

Another important note here, I like to add `laravel/vapor-cli` as a dependency to the project and lock the version. That's why we use `./vendor/bin/vapor` instead of `vapor` directly.
If you prefer to use the global `vapor` command, you can replace the dependency installing step with `composer global require laravel/vapor-cli` and on the deployment step use `vapor deploy production`.

## Using custom Docker image from private AWS ECR repository

In the [previous post](https://orkhan.dev/2024/02/06/using-pre-built-docker-images-with-laravel-vapor-to-reduce-deployment-time/) we talked about using pre-built Docker images with Vapor deployments.

If you used a public repository on AWS ECR or Docker Hub, you should be able to use the above GitHub Actions workflow without any changes.

But if you use a private repository, you need to authenticate with the registry before you can push or pull images. This requires adding a few more steps to the workflow file.

Before running the deployment command we need to:

- Configure AWS credentials using AWS access key and secret
- Login to the AWS ECR registry
- Authenticate Docker client to the ECR registry

Luckily for us, there are official GitHub Actions runners for each of these steps. Our updated workflow file will look like this:

```yaml
name: Deploy Production

env:
  VAPOR_API_TOKEN: ${secrets.VAPOR_API_TOKEN}

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install Yarn dependencies
        run: yarn install

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.3

      - name: Install Composer dependencies
        run: composer install --no-dev

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-access-key-id: ${secrets.AWS_ACCESS_KEY_ID}
          aws-secret-access-key: ${secrets.AWS_SECRET_ACCESS_KEY}
          aws-region: AWS-REGION-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Login Docker to ECR
        uses: docker/login-action@v3
        with:
          registry: ${steps.login-ecr.outputs.registry}
          username: ${steps.login-ecr.outputs.docker_username_1234567890_dkr_ecr_AWS-REGION-1_amazonaws_com}
          password: ${steps.login-ecr.outputs.docker_password_1234567890_dkr_ecr_AWS-REGION-1_amazonaws_com}

      - name: Deploy Environment
        run: ./vendor/bin/vapor deploy production
```

### Configure AWS credentials

In this step, we use the `aws-actions/configure-aws-credentials` action to configure AWS credentials using the AWS access key and secret.
As you can notice we provide this action with `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` secrets. This means you need to add these secrets to your repository settings as well, similar to `VAPOR_API_TOKEN`.

I recommend you create a dedicated IAM user on your AWS account with the minimum required permissions and use its access key and secret here.
AWS has a special user policy called `AWSAppRunnerServicePolicyForECRAccess`, which allows listing and reading from ECR repositories, it is a perfect policy for this use case.

Also, don't forget to replace `AWS-REGION-1` with your AWS region.

### Login to Amazon ECR

This step uses the `aws-actions/amazon-ecr-login` action to log in to the AWS ECR registry.
It does not require any configuration, but it outputs some values that we use in the next step, so we give it an `id` to reference later.

### Login Docker to ECR

Finally, we use the `docker/login-action` action to authenticate the Docker client to the ECR registry with the credentials provided in the previous step.

This step also uses some configuration.
You can leave `registry` as is, but you need to replace `1234567890` with your AWS account ID and `AWS-REGION-1` with your AWS region.
Do not modify the rest of the string, including `docker_username` and `docker_password`.

If everything is correctly configured, once this workflow runs, it will authenticate with the AWS ECR registry, pull the image, and deploy the application using the pre-built Docker image from the private AWS ECR repository.
