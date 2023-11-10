---
layout: post
locale: en_US
title: "How To: Build Docker Image for Phoenix Application"
description: Allowing developers to package their applications along with all the necessary dependencies into a single container and making it easier to deploy and run the application on any platform docker has become an industry standard.In this article, we will discuss how to create a Docker image for a Phoenix application, which is a popular web framework in Elixir.
image: /assets/images/posts/image.png
tags:
  - devops
  - elixir
  - phoenix
  - docker
  - docker-image
  - Dockerfile
  - phoenix-framework
status: in progress
categories:
  - en
  - devops
  - elixir
published: true
---
Deploying your project to a bare metal server or VPS like described in [the previous article]({% link  _posts/2021-05-15-how-to-deploying-phoenix-application-on-ubuntu-20.04LTS.md %}) is fine at the very begining. But when it comes to thousands or even better millions of users it becomes necessary to scale the deployment. This is where docker comes into play. It has gained immense popularity in recent years due to its ability to simplify the deployment process and create isolated environments for applications. Allowing developers to package their applications along with all the necessary dependencies into a single container and making it easier to deploy and run the application on any platform docker has become an industry standard.

In this article, we will discuss how to create a Docker image for a Phoenix application, which is a popular web framework in Elixir.

## Prerequisite
Before we begin, make sure you have installed and working
- [Erlang](http://www.erlang.org) and [Elixir](http://elixir-lang.org)  as runtime environment for Phoenix
- [Docker](https://docs.docker.com/desktop/) for building and testing the image
- [Phoenix application](https://hexdocs.pm/phoenix/up_and_running.html) you are going to create the image from, eg. [Midas app](https://github.com/T0ha/midas)

In code examples we use these placeholders
- `$PROJECT$` ---  your project name, eg. my_project
- `$ELIXIR_VSN$` --- Elixir version used for the project
- `$ALPINE_VSN$` --- Alpine linux image version used as base for Elixir image. See below.

## Creating a Dockerfile
Next, we need to create a `Dockerfile` in the root directory of our Phoenix application. The `Dockerfile` is used to define the instructions for building the Docker image.

Open a text editor and create a new file named `Dockerfile` (without any file extension). Add the following content to it:

```dockerfile
# Use an official Elixir runtime as the base image
FROM elixir:$ELIXIR_VSN$-alpine as builder



ENV MIX_ENV prod

RUN apk add --update --no-cache bash git openssh openssl
RUN apk add --update --no-cache --virtual .gyp g++ make

RUN mix local.hex --force && \
    mix local.rebar --force
    
WORKDIR /$PROJECT$

COPY mix.exs mix.lock ./
COPY config config
COPY assets assets
COPY priv priv
COPY lib lib

RUN mix do deps.get --only ${MIX_ENV}, deps.compile
ENV PATH="/root/.mix/escripts:${PATH}"

RUN mix assets.deploy

RUN mix release $PROJECT$

# Run Release
FROM alpine:$ALPINE_VSN$

RUN apk add --update --no-cache bash openssl postgresql-client libstdc++

ENV MIX_ENV prod

EXPOSE 5000

WORKDIR /$PROJECT$
COPY --from=builder /$PROJECT$/dist/ .

HEALTHCHECK CMD curl --fail http://localhost:4000/ || exit 1

CMD ["sh", "-c", "/$PROJECT$/bin/$PROJECT$ start"]

```

Let's go through the `Dockerfile` step by step:
- The first line specifies the base image to use, which is an official Elixir runtime image with a specific version.
- We set mix environment to `prod` with `MIX_ENV` environment variable.
- Next, we install the necessary build dependencies using the Alpine package manager (apk).
- We create a working directory inside the container and copy the mix.exs and mix.lock files, as well as the config directory.
- Then, we copy the remaining application files (lib and priv, etc).
- We fetch and compile the project dependencies.
- After that, we build js and other assets for the project.
- We compile the project and build a release using the mix release command.
- Finally, we use a minimal Alpine image as the final base image and copy the release from the previous stage. There is a trick here. Alpine image version (`$ALPINE_VSN$`) should be exactly the same as in the image the Elixir and Erlang images are based. You can look this version on [elixir image tags page](https://hub.docker.com/_/elixir/tags). Go to the tag you use. Navigate to "Images" tab. And look one for alpine.

## Building the Image
With the `Dockerfile` in place, we can now build our Docker image. Open a terminal in the project directory and run the following command:

```bash
$ docker build -t $PROJECT$ .
```

This command instructs Docker to build an image with the tag `$PROJECT$` using the current directory as the build context.
The build process may take some time, as Docker needs to download and install all the necessary dependencies specified in the `Dockerfile`.
## Running the Docker Image
Once the build process is complete, we can run our Phoenix application inside a Docker container. Run the following command:

```bash
$ docker run -p 4000:4000 $PROJECT$
```

This command starts a new Docker container based on our image and maps port 4000 from the container to port 4000 on the host machine.

Now, you can access your Phoenix application by opening a web browser and navigating to http://localhost:4000.

