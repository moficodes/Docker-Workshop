# ADD vs COPY

For the most part you can get the same result from both ADD and COPY. There are some differences though.

### Docker File Copy Command 

if you want to copy the files from local machine to docker image you can use copy command

`COPY  <src>  <dest>`

`COPY  /path/to/the/file/in/your/system   /destination/path/in/docker/image`

`COPY . /project`

  
Here we are copying all files into project folder in docker image

### Docker File ADD Command 

If you want copy or download the files from remote location to docker image you can use add command.

`ADD  <src>  <dest>`

`ADD /source/file/url/path       /destination/path/in/docker/image`

Ex1:

`ADD http://mirrors.estointernet.in/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz /opt`

Here maven tar file will be downloaded from above URL and it will be placed in opt directory in docker image.

And also it will extract the maven tar file in opt directory in docker image. So add command we can use in two ways  
one is to download the files from remote URL to docker image and to extract the files.

Ex2:

In the previous example we downloaded a maven tar file and extracted into docker opt directory. Let’s say that if you have zip or any tar files in your system and if you want send those tar or zip files to docker image and extract. in this case also we can use same ADD command.

`ADD <src> <dest>`

`ADD abcd.tar.gz   /opt`



