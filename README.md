# Frontend Masters Docker

## Introduction

Containers are much "lighterweight" than a traditional VM. Why manage a whole VM (i.e ensuring your operating system is up to date and your security is good (i.e ssh)) when you could just provide your code, define the environment it needs to run and then you are good.

It costs cloud providers much less to provide infrastructure for containers because you can run many containers on the same VM and keep them isolated from one another. It costs much more to spin up multiple VMs on one host operating system. And many apps in the cloud do not need to have a whole operating system available to run them. Containers allow you to be more fine-grained about the computing resources your company needs in the cloud.

"Containers give us many of the security and resource-management features of VMs but without the cost of having to run a whole other operating system."

Containers use the linux tools `chroot`, `namespace`, and `cgroup` to separate a group of processes from each other. `chroot` allows you to set the root directory of a new process. This "jails" off a process. Once a new "root" is established, you can't reference binaries that live outside of your new "root" directory. Technically what you can do is, for example, `cp /bin/bash /my-new-root`. Then you `cd /my-new-root && chroot . /bin/bash`, then it could work if you have also added the correct libraries that `bash` depends on. Many of these binaries themselves have library dependencies, for example `.so` files (shared object files). These are dynamically linked dependencies shared across linux tools. You can use `ldd` to inspect the `.so` libraries a given dependency relies on. For example, `ldd /bin/ls`. Then you can `cp /lib/{libname}.so /my-new-root/lib`. This is essentially what Docker does to simulate an "image" of a linux vm. 

`debootstrap` can add the minimum files needed for a debian/ubuntu linux environment to a given path. You could then `chroot` inside of it to make something akin to a container. This tool is something that Docker actually uses. The `unshare` tool can be used to make a child process (`chroot`) not share certain "namespaces." `chroot` is about unsharing the file system, and namespaces are about unsharing capabilities, for example, seeing the process system. 

Use `htop` to see how many cores your environment has access to, as well as how much memory is being used by each running process. Examine each process. 

`cgroup` stands for "control group" and it is meant to limit the amout of resources given to a dedicated process, for example CPU and memory. This allows you to run multiple containerized processes on the same server without fear that a runaway process (i.e memory leak) will take down the whole physical server and therefore any other running processes. This allows you to create `chroot`-ed `unshared`-ed environments and assign control groups to them to truly keep these environments "jailed" off. `cgroup` can be installed with `apt` and was created by google. 

Once the `unshared` `chroot`-ed environment running `bash`, for example, is created on the host machine, on the host, you can inspect the environment's "parent" bash process and assign a control group to this bash process' PID (the host "knows" about and can inspect these environment processes). The bash process will obviously be spawning children processes, and all children processes to this bash process (aka a Process Tree) will be subject to the same environment and control group. 

Setting control group rules can be very arcane and difficult to parse. This is just an academic exercise, it's something Docker does for you. For example, this command assigns a rule to an example cgroup called "sandbox" that stipulates that any process tree subject to this cgroup can only use 10% of available processing power on the host machine: 

```bash
cgset -r cpu.cfs_period_us=100000 -r cpu.cfs_quota_us=$[ 5000 * $(getconf _NPROCESSORS_ONLN) ] sandbox
```

And this will limit the total number of bytes of RAM available to any processes (process trees) subject to the sandbox control group:

```bash
cgset -r memory.limit_in_bytes=80M sandbox
```

This is essentially what Docker accomplishes for you "under the hood." Docker does use `cgroup`.

## Using Docker

Docker images are basically just a large zip file of sorts. A docker image dumps out the state of a partcular container and tars it (or zips it) together. When you download a docker image, you're just pulling down the zip. When you hand off an image to docker and tell it to run it as a container, docker unzips it and performs tasks similar to those outlined in the introduction to create the container. 

Containers are made to be ephemeral. They are made to be spun up and destroyed frequently. When you're using a dockerfile, just assume nothing is permanent.

To create an alpine linux container and drop into an interactive shell, you can run this: 

`docker run --interactive --tty alpine:3.10` or for short, `docker run -it alpine:3.10`. `docker run alpine:3.10` will simply start the container and then destroy it. You can also pass one or more shell commands simply `docker run alpine:3.10 ls`. 

For example:

```
➜  frontendmasters-docker git:(main) ✗ docker run --interactive --tty alpine:3.10

Unable to find image 'alpine:3.10' locally
3.10: Pulling from library/alpine
396c31837116: Pull complete 
Digest: sha256:451eee8bedcb2f029756dc3e9d73bab0e7943c1ac55cff3a4861c52a0fdd3e98
Status: Downloaded newer image for alpine:3.10
/ # cat /etc/issue 
Welcome to Alpine Linux 3.10
Kernel \r on an \m (\l)

/ # exit
```

To run an ubuntu image, run `docker run -it ubuntu:bionic`. If you haven't already pulled down this ubuntu image from docker hub, docker does it for you. 

`docker image prune` will remove images you've downloaded, since images can be quite large. 

Run `docker run ...` with the `--detach` flag to have the docker daemon start up the container in the background. Run `docker ps` to view running docker processes and use `docker attach` to "attach" to them. Kill a docker process with `docker kill <dockerPID>`.

Pull the official latest node image with: `docker run -it node` or `docker run -it node:latest`. At the time of taking these notes, that pulled a debian linux image with:

```
PS C:\Users\pszujewski> docker run -it node bash
root@7506750cc201:/# cat /etc/issue
Debian GNU/Linux 11 \n \l
```

And with node version 17 installed.

Use `docker inspect` to inspect metadata on a container image that you've pulled. Use `docker exec` to run a shell command against an existing container that is already running in a detached state.

`docker info` is useful if you, for example, `ssh` into a remote vm somewhere in the cloud and you want to inspect the host machine that your docker container(s) are running on. It provides info on the host machine. 

`docker run -dit mongo` is a command that will run a mongo image container in a detached and interactive state (in the background, as a background docker process). The "d" is for `--detached`.

`docker container prune` will prune all exited containers. `docker image list` shows all downloaded images. Prune them with `docker image prune`. `docker container list` will show all exited containers.

## Dockerfiles

This file contains instructions that Docker will execute line by line. The `FROM` instruction in a Dockerfile tells Docker which container image to use. 

`docker build <pathtofolderwithdockerfile>` will "build" your docker container. That will produce a long id, which you can use to run the contiainer with `docker run <sha256>`. The CMD command in the dockerfile will overwrite whatever the image's default command is. 

Build with a tag by executing, for example, `docker build --tag my-node-app .` where `.` means the current directory. The you simply execute `docker run my-node-app`. You can tag your docker builds with different versions of sorts for example, `docker build --tag my-node-app:2 .` You can also just write `-t` instead of `--tag`.

`docker run --init --rm my-node-app`. `--init` will ensure that SIGTERM passed to docker with ctrl+C kills all child processes (spwaned docker containers). 

To expose the network (allow host computer to share the network with a container), you can use `EXPOSE` in the dockerfile, or add this on the cmd line, `docker run --init --rm --publish 3000:3000 my-node-app`. Caution, `EXPOSE` doesn't work exactly the same as this publish`.

You should also not allow for root user in your containers. Add `USER node` to your Dockerfile. The node container ships with a user named node already. The creators of the container set up this user in the container. Note `--chown=node:node` means the file will be copied and owned by the node user in the "node user group". 

Where does docker copy the source code? `WORKDIR /home/node/src`. Use `ls -lsah` to (in addition to listing all files...) see what user and user group owns each file, and if they can "rwx" on that file.

## Important note

You cannot bind a http server to `localhost` in a container. `localhost` is a hard loop back that you cannot get around. That's why in the hapi configuration, `0.0.0.0` is expressly written instead of "localhost". 

## Multi-stage builds

This allows you to create new containers from the same base image as you go from installing app dependencies, to building the app, to finally running the app. For example, for your final Production image, you probably don't need to include your source code from which your app was built, allowing your image to be smaller. Read more here, https://docs.docker.com/develop/develop-images/multistage-build/

"...let's build something first, copy the output into a new container and go from there..." So you're building one container from previous containers that are thrown away.

To name a stage do something like this:

```
# First container is named "build"
# This is the "build" stage
FROM node:12-stretch AS build
WORKDIR /build
COPY package-json package-lock.json ./
RUN npm ci
COPY . .

# Second container is for the "runtime" stage, so we'll
# call it "runner"
# Notice that builder and runner are using entirely different base images
FROM alpine:3.10
RUN apk add --update nodejs
RUN addgroup -s node && adduser -S node -G node
USER node

RUN mkdir /home/node/code
WORKDIR /home/node/code
COPY --from=build --chown=node:node /build . 
# Note "--from=build", meaning from the previous "build" stage.
# etc, etc, etc...
```

## Different Dockerfiles for different environments

For example, you could create a dev.Dockerfile and use that as your dev environment instead of your production Dockerfile. To build with a different Dockerfile, execute, for example 

`docker build -t my-node-app -f dev.Dockerfile .`
