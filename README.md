# Modulus Docker Images
This repository is the entry point for all Modulus build and runtime images. It provides a high-level overview of the Modulus Docker image standard as well as links to all language-specific images.

## The Modulus Docker Standard
Operating in a PaaS environment requires a well-defined external interface, a high degree of resource control, and security. The standard we’ve developed allows us to meet these requirements while providing an extensible and maintainable set of Docker images.

Below is a diagram of our current Docker inheritance tree.

``` text
phusion/baseimage-docker
  |- onmodulus/docker-base
      |- onmodulus/docker-build-base
          |- onmodulus/docker-build-node
          |- onmodulus/docker-build-java
          |- onmodulus/docker-build-php
          |- onmodulus/docker-build-static
      |- onmodulus/docker-run-base
          |- onmodulus/docker-run-node
          |- onmodulus/docker-run-java
          |- onmodulus/docker-run-php
          |- onmodulus/docker-run-static
```

## Base Image
All Modulus images inherit from [Phusion’s base image](https://github.com/phusion/baseimage-docker). Because we can’t control whether or not child processes will be created at runtime, we needed a base image that solved the zombie process problem. Phusion replaced the init system of the standard Debian image so zombie processes won’t prevent a Docker container from stopping.

The [Modulus base image](https://github.com/onmodulus/docker-base), which inherits from the Phusion base image, is responsible for creating the user (mop) in which all in-container processes are run. Modulus does not run any external code or services as root within the container. The mop user is also prohibited from writing anywhere on the Docker filesystem. The volume it’s allowed to write to is mounted in from the host filesystem. This allows us to control the disk resources used by the mop user and the processes it’s running.

## Build Images
Build images inherit from the Modulus base image and are used during the build phase of a Modulus deploy. Currently only the [Node.js build image](https://github.com/onmodulus/docker-build-node) has anything interesting in its build script. The standard for a build image is simple: there must be an executable binary somewhere in the PATH named “build”. When Modulus invokes a build this is the command that is run. Build images only live for as long as the build takes to run (with a 15 minute timeout). Modulus also injects INPUT and OUTPUT environment variables that can be used by the script to know where the source is and where to put the compiled output. The input and output directories are created on the host using an LVM thin-provisioned volume with a 2GB limit. The only place the mop user can write to is this volume. 

## Run Images
The [Modulus run images](https://github.com/onmodulus/docker-run-base) contain most of the complexity. The base run image installs [supervisord](http://supervisord.org/) and several libraries that our customer applications would need access to. Because we cater to arbitrary applications, our runtime images are much larger than typical Docker images.

Like the build image, our standard only requires a single executable in the PATH, named “start”. However unlike the build image, the start executable will be run with supervisord.

The volume mounted inside our runtime containers must have the following layout:

```text
/mnt/
  |- app/
  |- home/
  |- log/
  |- notifications/
  |- tmp/
  |- supervisor.conf
```

The global supervisord configuration loads configuration files from /mnt/supervisor.conf. Modulus generates this file automatically as a way to load environment variables. This configuration file also sends all log data to /mnt/log/app.log.
The customer’s application source is located in /mnt/app. The language-specific images contain different start scripts that know how to deal with its source code. For example, the Node.js start script reads the package.json and initializes the correct version of Node.

The HOME and TEMP_DIR environment variables are set to /mnt/home and /mnt/tmp respectively. Having a properly writeable home and temp location is important for many environments and the base image automatically defines these variables for all inherited images.

Modulus includes a tool named “Navi” inside all run images. This tool hooks into supervisord’s notification system and generates json files that are picked up by our higher-level system. These files are placed in /mnt/notifications. Currently the notifications that are generated are crash and out-of-memory.

Modulus controls the customer’s process by using the [docker exec](https://docs.docker.com/reference/api/docker_remote_api_v1.19/#exec-create) api to send commands to supervisorctl. 

# License
Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.


