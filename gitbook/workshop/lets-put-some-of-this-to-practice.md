# Lets put some of this to practice

We will put what we learned in the step prior to test and start making a dockerfile for an go application from bad to great.

We have a few versions of the Dockerfile.

The App we are dealing with is a go application that gives back a list of prime numbers.

### Version 1:

```text
cat Dockerfile-1
```

```text
# start with a base image
FROM ubuntu:xenial

RUN apt-get update -y
RUN apt-get upgrade -y

# need some things installed
RUN apt-get install -y wget
RUN apt-get install -y build-essential
RUN apt-get install -y ca-certificates
RUN apt-get install -y git

RUN wget https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz
RUN tar -xvf go1.12.6.linux-amd64.tar.gz
RUN mv go /usr/local
RUN rm go1.12.6.linux-amd64.tar.gz

ENV GOROOT /usr/local/go

RUN mkdir go

RUN GOPATH=$HOME/go

ENV PATH $GOPATH/bin:$GOROOT/bin:$PATH

WORKDIR /src

COPY ./go.mod ./go.sum ./

RUN go mod download

COPY ./ ./

RUN go build -o /app .

EXPOSE 8080

ENTRYPOINT [ "/app" ]
```

Lets read through this Dockerfile. Most command should be self explanatory. But lets make sure we understand what its doing.

```text
FROM ubuntu:xenial
```

We start with `ubuntu` as our base image.

Then we have 

```text
RUN apt-get update -y
RUN apt-get upgrade -y

# need some things installed
RUN apt-get install -y wget
RUN apt-get install -y build-essential
RUN apt-get install -y ca-certificates
RUN apt-get install -y git

RUN wget https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz
RUN tar -xvf go1.12.6.linux-amd64.tar.gz
RUN mv go /usr/local
RUN rm go1.12.6.linux-amd64.tar.gz
```

This is installing all the things we need. We are also being mindful of our download and deleting the go binary. 

We then setup our go env variables.

Copy our code, Expose the port needed and finally set the binary as the `ENTRYPOINT` .

Lets see if this works.

```text
docker build -t prime:v1 -f Dockerfile-1 .
```

After the build is complete lets run it.

```text
docker run -d -p 8080:8080 prime:v1
```

Lets see if it worked.

```text
curl http://localhost:8080/primes/10
```

We should see 

```text
{
"data": [
"2",
"3",
"5",
"7"
]
}
```

Stop the container

```text
docker stop <container-id>
```

Lets use some of our learnings to improve this 

### Version 2

```text
cat Dockerfile-2
```

```text
FROM ubuntu:xenial

RUN apt-get -qq update -y && apt-get -qq upgrade -y

# need some things installed
RUN apt-get -qq install -y wget build-essential ca-certificates git


RUN wget -q https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz && tar -xf go1.12.6.linux-amd64.tar.gz && mv go /usr/local && rm go1.12.6.linux-amd64.tar.gz

ENV GOROOT /usr/local/go

RUN mkdir go

RUN GOPATH=$HOME/go

ENV PATH $GOPATH/bin:$GOROOT/bin:$PATH

WORKDIR /src

COPY ./go.mod ./go.sum ./

RUN go mod download

COPY ./ ./

RUN go build -o /app .

EXPOSE 8080

ENTRYPOINT [ "/app" ]
```

Its the same dockerfile in content except some `RUN` commands have been packed together with &&.

```text
docker build -t prime:v2 -f Dockerfile-2 .
```

Lets run it just to make sure its still working.

```text
docker run -d -p 8080:8080 prime:v2
```

It should be the same

Lets see how much we improved in size

```text
docker images
```

```text
prime                 v2                  8d370068e716        37 seconds ago      730MB
prime                 v1                  5159d258a5ee        7 minutes ago       1.2GB
```

Just by combining `Run` commands we shaved around 500MB. Not bad. Lets keep going.

### Version 3

```text
cat Dockerfile-3
```

```text
FROM ubuntu:xenial

RUN apt-get -qq update -y && apt-get -qq upgrade -y

# need some things installed
RUN apt-get -qq install -y wget build-essential ca-certificates git

# Create the user and group files that will be used in the running container to
# run the process as an unprivileged user.
RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group

RUN wget -q https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz && tar -xf go1.12.6.linux-amd64.tar.gz && mv go /usr/local && rm go1.12.6.linux-amd64.tar.gz

ENV GOROOT /usr/local/go

RUN mkdir go

RUN GOPATH=$HOME/go

ENV PATH $GOPATH/bin:$GOROOT/bin:$PATH

WORKDIR /src

COPY ./go.mod ./go.sum ./

RUN go mod download

COPY ./ ./

RUN go build -o /app .

EXPOSE 8080

# Perform any further action as an unprivileged user.
USER nobody:nobody

ENTRYPOINT [ "/app" ]
```

This ones identical to the last version. But adds a valuable security feature. We are setting a user. Because by default docker sets the root user which is all sorts of bad.

Feel free to build and run this docker images as v3.

### Version 4

```text
cat Dockerfile-4
```

```text
# Accept the Go version for the image to be set as a build argument.
# Default to Go 1.11
ARG GO_VERSION=1.12

# First stage: build the executable.
FROM golang:${GO_VERSION}-alpine AS builder

# Create the user and group files that will be used in the running container to
# run the process as an unprivileged user.
RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group

# Install the Certificate-Authority certificates for the app to be able to make
# calls to HTTPS endpoints.
# Git is required for fetching the dependencies.
RUN apk add --no-cache git

# Set the working directory outside $GOPATH to enable the support for modules.
WORKDIR /src

# Fetch dependencies first; they are less susceptible to change on every build
# and will therefore be cached for speeding up the next build
COPY ./go.mod ./go.sum ./
RUN go mod download

# Import the code from the context.
COPY ./ ./

# Build the executable to `/app`. Mark the build as statically linked.
RUN CGO_ENABLED=0 go build \
    -installsuffix 'static' \
    -o /app .

# Declare the port on which the webserver will be exposed.
# As we're going to run the executable as an unprivileged user, we can't bind
# to ports below 1024.
EXPOSE 8080

# Perform any further action as an unprivileged user.
USER nobody:nobody

# Run the compiled binary.
ENTRYPOINT ["/app"]
```

We are using a official docker image for golang on this one.

```text
docker build -t prime:v4 -f Dockerfile-4 .
```

Test the application still works by running it.

But I am more interested in the size improvement.

```text
prime                 v4                  da72555bbd19        54 seconds ago      400MB
prime                 v2                  8d370068e716        14 minutes ago      730MB
prime                 v1                  5159d258a5ee        21 minutes ago      1.2GB
```

Now we are sitting at around 400MB.

### Version 5

This is the final version of this dockerfile and the one I am using.

```text
cat Dockerfile-final
```

```text
# Accept the Go version for the image to be set as a build argument.
# Default to Go 1.11
ARG GO_VERSION=1.12

# First stage: build the executable.
FROM golang:${GO_VERSION}-alpine AS builder

LABEL maintainer="moficodes@gmail.com"
# Create the user and group files that will be used in the running container to
# run the process as an unprivileged user.
RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group && \
    apk add --no-cache git=2.22.0-r0

# Install the Certificate-Authority certificates for the app to be able to make
# calls to HTTPS endpoints.
# Git is required for fetching the dependencies.


# Set the working directory outside $GOPATH to enable the support for modules.
WORKDIR /src

# Fetch dependencies first; they are less susceptible to change on every build
# and will therefore be cached for speeding up the next build
COPY ./go.mod ./go.sum ./
RUN go mod download

# Import the code from the context.
COPY ./ ./

# Build the executable to `/app`. Mark the build as statically linked.
RUN CGO_ENABLED=0 go build \
    -installsuffix 'static' \
    -o /app .

# Final stage: the running container.
FROM scratch AS final

# Import the user and group files from the first stage.
COPY --from=builder /user/group /user/passwd /etc/

# Import the compiled executable from the first stage.
COPY --from=builder /app /app

# Declare the port on which the webserver will be exposed.
# As we're going to run the executable as an unprivileged user, we can't bind
# to ports below 1024.
EXPOSE 8080

# Perform any further action as an unprivileged user.
USER nobody:nobody

# Run the compiled binary.
ENTRYPOINT ["/app"]
```

This look almost identical except for until the `go build` 

```text
# Final stage: the running container.
FROM scratch AS final

# Import the user and group files from the first stage.
COPY --from=builder /user/group /user/passwd /etc/

# Import the compiled executable from the first stage.
COPY --from=builder /app /app

# Declare the port on which the webserver will be exposed.
# As we're going to run the executable as an unprivileged user, we can't bind
# to ports below 1024.
EXPOSE 8080

# Perform any further action as an unprivileged user.
USER nobody:nobody

# Run the compiled binary.
ENTRYPOINT ["/app"]
```

So we have a multistage build now.

What this allows us to do is to use the binary build on a previous stage and just copy the binary. 

Lets see the improvement.

```text
docker build -t prime:v5 -f Dockerfile-final .
```

Lets see the size improvement with multistage build

```text
prime                 v5                  0680bdacb9de        2 minutes ago       7.89MB
prime                 v4                  da72555bbd19        15 minutes ago      400MB
prime                 v2                  8d370068e716        29 minutes ago      730MB
prime                 v1                  5159d258a5ee        35 minutes ago      1.2GB
```

7.89 MB, Thats all. Compare that to the 1.2 GB, thats over 150x improvement.

