# Dockerfile

The Dockerfile is essentially the build instructions to build the image. The advantage of a Dockerfile over just storing the binary image \(or a snapshot / template in other virtualisation systems\) is that the automatic builds will ensure you have the latest version available. This is a good thing from a security perspective, as you want to ensure you’re not installing any vulnerable software.

### Dockerfile Keywords

#### FROM

every dockerfile starts with from command. in the from command, we will mention base image for the docker image.

Example:

`From node:latest`

#### MAINTAINER

`MAINTAINER decodingdevops@gmail.com`

maintainer command is used to represent the author of the docker file.

#### RUN

the run command is used to run any shell commands. these commands will run on top of the base image layer. run commands executed during the build time. you can use this command any number of times in dockerfile.

#### CMD

the cmd command doesn’t execute during the build time it will execute after the creation of the container. there can be only one cmd command in dockerfile. if you add more than one cmd commands the last one will be executed and remaining all will be skipped. whatever you are mentioning commands with cmd command in dockefile can be overwritten with docker run command. if there is entrypint command in the dockerfile the cmd command will be executed after entry point command. with cmd command you can pass arguments to entrypoint command.

#### EXPOSE

after creating the container, the container will listen on this port

#### ENV

this command is used to set environment variables in the container.

#### COPY

`COPY  <SRC> <DEST>`

it will copy the files and directories from the host machine to the container. if the destination does not exist. it will create the directory.

`COPY  .  /usr/src/app`

here dot\(.\) means all files in host machine dockerfile directory, will be copied into container /usr/src/copy directory.

#### ADD

`ADD  <SRC>  <DEST>`

it will copy the files and directories from the host machine to the container and its support copy the files from remote URLs.

#### ENTRYPOINT

The first command that will execute in the container is entrypoint command. Entrypoint command will be executed before the cmd command.  If you mentioned more than one, only the last one will be executed and remaining all will be skipped. cmd command paraments are arguments to entrypoint command.

