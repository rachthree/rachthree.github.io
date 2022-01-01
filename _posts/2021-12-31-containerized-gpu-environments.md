---
layout: post
title: "Containerized GPU AI/ML Dev Environments"
tags:
- Docker
- TensorFlow
- PyTorch
- GPU
- Containers
- Environment
- CUDA
- Linux
- Windows
thumbnail_path: "blog/docker-gpu-dev-env/docker_tf_pytorch.png"
---

A year has flown by. A lot happened which put my posting and side projects on hold. Now with the holidays upon us, I finally have some time to revisit my projects. Might as well post after updating this for some security vulnerabilities!

At my last job, I helped set up AWS EC2 instances for AI/ML teams. These teams mainly used TensorFlow, and [TensorFlow has its own CUDA and CuDNN requirements per version](https://www.tensorflow.org/install/source#gpu). This led to environment headaches as getting CUDA right for everyone could become rather catastrophic depending on the installation method used. However, these teams always started fresh, only needed one version of TensorFlow, and had the luxury of spinning up new EC2 instances. We could start from scratch, ensuring a fresh, working environment for all.

Why couldn't we use Docker? Long story short, it was not approved for our use at the time. Had I known then what I know now, I would have pushed for it and solved future headaches. At my next role, there were already multiple users with their own versions of TensorFlow and PyTorch on a multi-GPU machine. CUDA and TensorFlow worked for some but not for others. Athough there are ways to work with [multiple CUDA versions on the same machine](https://medium.com/@peterjussi/multicuda-multiple-versions-of-cuda-on-one-machine-4b6ccda6faae), I needed to find a way to ensure things worked with little hassle for new team members without bringing down the hammer on the environment.

Thankfully, Docker was available. Having exec'd into app containers to debug and test code in the app's compatible environment, I wanted to bring a containerized environment to development. The end goal was to have people stop saying "_But it works for my environment!_".

### Requirements
After getting familiar with the different workflows of my teammates, I came up with the following requirements:
1. The user shall be able to edit code, run code, and use Git from `/home` as normal on a remote multi-GPU Linux machine.
2. The user shall be able to access datasets on the remote machine as normal.
3. The user shall not rely on IDEs for remote container development (otherwise [VS Code would have been great](https://code.visualstudio.com/docs/remote/containers)).
4. The user shall not need to be familiar with Docker commands.
5. The user shall be able to use GPUs for TensorFlow and PyTorch without worrying about CUDA and CuDNN versions.
6. The user shall be able to run Jupyter Lab/notebooks.


### Solution
I defintely do not claim to be an expert. With a little Googling, this was my solution:
1. Base the image from a TensorFlow one since it is already built with the CUDA and CuDNN it needs. Luckily, PyTorch comes bundled with its own CUDA through pip, so it can be installed afterwards.
2. Use a `requirements.txt` file to install additional Python libraries the dev may need off the bat.
3. Use a Docker Compose file to:
    * Bind mounts for `/home` and the datasets directory
    * Expose a port to the host machine for Jupyter
    * Hacky: bind `/etc/passwd` so that the user and group ID will show as normal and the user can use Git, pip, etc. as normal.
4. Use a Makefile to simplify Docker commands for:
    * Building a dev image
    * Starting a dev container

To not reveal anything pertaining to company code, I adapted and simplified this for my own use at home, shown below.

### Files
All code here can be found on my GitHub [here](https://github.com/rachthree/gpu-docker-dev-example). For this example, this is the directory structure of the files we need:
```
📦gpu-docker-dev
 ┣ 📂.build
 ┃ ┗ 📜Dockerfile
 ┣ 📜docker-compose-linux.yml
 ┣ 📜Makefile
 ┗ 📜requirements.txt
 ```

Placing `Dockerfile` in a `.build` folder does not matter, but this is done for potential future organization if there are more Dockerfiles.

### Dockerfile
The Dockerfile is as below, with sections numbered as comments:
```
# 1
ARG TF_VER
FROM tensorflow/tensorflow:$TF_VER

# 2
RUN apt-get update && apt-get install -yq \
  build-essential \
  curl \
  git \
  nano \
  vim \
  wget \
  graphviz

# 3
COPY requirements.txt /tmp/requirements.txt

WORKDIR /tmp
RUN pip3 install --upgrade pip
RUN pip3 install -r requirements.txt -v

# 4
ARG PYTORCH_VER
RUN pip3 install $PYTORCH_VER

# 5
EXPOSE 8888
```
Above:
1. The `TF_VER` argument is used to specify from which TensorFlow Docker image version to base this image. `FROM` will then use that image as your base image.
    * The Docker Compose YAML will have defaults for `TF_VER`.
    * Alternatively, you can build TensorFlow from source if you depend on another image for other uses. See [here](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/tools/dockerfiles) to check out how the TensorFlow images were built, and you can leverage the same instructions to your own Dockerfile.
2. This updates `apt-get` and installs packages into the image's OS that may be needed for development.
3. `requirements.txt` is the usual requirements file for Python and lists libraries such as `jupyterlab`, `transformers`, etc. It is copied into the image's `/tmp` folder. Once changing the working directory to `/tmp`, `pip install` is used (after upgrading) to install those requirements.
4. The `PYTORCH_VER` is used to specify which PyTorch version to install, and then PyTorch is installed via `pip`
    * The Docker Compose YAML will have defaults for `PYTORCH_VER`
5. Container port 8888 is exposed to the host machine, and will be used by Jupyter since it is the default port.

### Docker Compose
`docker-compose-linux.yml` contains:
```
version: "3.8"
services:
  build-env:
    build:
      context: .
      dockerfile: .build/Dockerfile
      args:
        TF_VER: ${TF_VER:-latest-gpu}
        PYTORCH_VER: ${PYTORCH_VER:-torch==1.10.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html}
    image: dev-env

  dev-env:
    image: dev-env
    container_name: dev-env
    restart: always
    volumes:
      - type: bind
        source: /home
        target: /home
      - type: bind
        source: /mnt
        target: /mnt
      - type: bind
        source: /etc/passwd
        target: /etc/passwd
        read_only: true
    user: ${UID}:${GID}
    ports:
      - "127.0.0.1:${PORT:-8888}:8888"
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
    shm_size: 2gb
```
Above:
* The `build-env` service is used just for building the image as the image does not need to know the user/group ID, home and other directories, host port to forward until containers are deployed, and resources to allocate.
  * `context` gives the build context so that `requirements.txt` is accessible. `dockerfile` is given the path to the Dockerfile from before.
  * The `args` define the `TF_VER` and `PYTORCH_VER` defaults that are being used in the Dockerfile. The user can override this using `TF_VER=<TF Docker version>` and `PYTORCH_VER=<PyTorch version>` when building the image.
  * The image name here is `dev-env` as it is just for my personal use. However, for multiple users, using something like `dev-env:$USER` so that the tag is the username will help differentiate images between different users on the same host machine.
* The `dev-env` service is what is getting deployed for the user. It relies on the image built by the `build-env` service.
  * `/home` is bound to the homonymous directory on the container so that the user can continue developing as normal.
  * `/mnt` is bound to the homonymous directory on the container. This file can be changed to anything the user needs, but for the sake of this example, we assume datasets are on separate storage devices mounted to the host machine. Alternatively, if using WSL, the user can access their Windows files through here.
  * `/etc/passwd` is bound as read-only and this is so the user ID and group ID will show as normal. This also helps with using git.
  * `user` assigns the user ID and group ID so that the user can show as themself. The Makefile below automates assigning these values.
  * `ports` forwards a specified `PORT` (default 8888) to the container's port 8888. This way, Jupyter can be access on the host machine's `PORT` with `--ip 0.0.0.0` (required when using a container)
  * Under `deploy`, `reservations` are made to allow usage of the GPU and allocating memory.
  * `shm_size` allocates the shared memory size. This may need adjusting depending on your needs.

### Makefile
The Makefile is as below:

```
SHELL = /bin/sh

UID := $(shell id -u)
GID := $(shell id -g)

DC_DEV_VARS := UID=$(UID) GID=$(GID)

build:
	docker-compose -f docker-compose-linux.yml build \
		build-env

dev:
	$(DC_DEV_VARS) docker-compose -f docker-compose-linux.yml run \
		--service-ports \
		dev-env bash -c "cd;bash"
```

The Makefile defines the tasks that can be executed. It also helps with setting the current user and group ID numbrs so that the user can appear as themselves when using the container. The user however can override any of the environment variables used in the Docker Compose YAML using `make <task> <VAR1>=<value1> <VAR2>=<value2>`. Here, the `docker-compose` commands are used with arguments so that the user does not need to remember how to use Docker commands, and instead can just use:
* `make build`: build the image. The `docker-compose` command points to the YAML from before and specifies buildling the `build-env` service.
* `make dev`: deploy the container and start up a terminal session within the container.
  * Pointing to the same YAML as before, this tine `dev-env` is the service requested, and that is where all the other variables come into play.
  * `--service-ports` allows the container port to be exposed.
  * `bash -c "cd;bash"` is the workaround to immediately start off in the user home directory. If only `bash` was used, the user would start off in the last working directory of the build process. The `/etc/passwd` file that was bounded from before helps with getting to the correct user home directory. 

### A Note on TensorFlow VRAM
TensorFlow by default allocates the entire GPU VRAM, which does not allow having multiple users on the same GPU and prevents from using PyTorch at the same time. Enabling memory growth will allow for this. To do this, the below should be run after importing TensorFlow, before anything else from TF is imported:

```
gpu_list = tf.config.list_physical_devices('GPU')
for gpu in gpu_list:
    try:
        tf.config.experimental.set_memory_growth(gpu, True)
    except Exception as e:
        print(f"Unable to enable memory growth for {gpu}. Error: {e}")
```

Hope this helps! May the New Year be free of environmental inconsistencies and CUDA problems!