# Dockerfile Best Practices

#### Tip \#1: Order matters for caching

![](../.gitbook/assets/image.png)

However, the order of the build steps \(Dockerfile instructions\) matters, because when a step’s cache is invalidated by changing files or modifying lines in the Dockerfile, subsequent steps of their cache will break. Order your steps from least to most frequently changing steps to optimize caching.

**Tip \#2: More specific COPY to limit cache busts**

![](https://i1.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/0c1d0c4e-406c-468c-b6ba-b71ac68b9c84.jpg?ssl=1)

Only copy what’s needed. If possible, avoid “COPY  .” When copying files into your image, make sure you are very specific about what you want to copy. Any changes to the files being copied will break the cache. In the example above, only the pre-built jar application is needed inside the image, so only copy that. That way unrelated file changes will not affect the cache.

**Tip \#3: Identify cacheable units such as apt-get update & install**

![](https://i2.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/2322a39e-bd7e-4a2b-9a8f-548a97dbacb4.jpg?ssl=1)

Each RUN instruction can be seen as a cacheable unit of execution. Too many of them can be unnecessary, while chaining all commands into one RUN instruction can bust the cache easily, hurting the development cycle. When installing packages from package managers, you always want to update the index and install packages in the same RUN: they form together one cacheable unit. Otherwise you risk installing outdated packages.

### Reduce Image size

Image size can be important because smaller images equal faster deployments and a smaller attack surface.

**Tip \#4: Remove unnecessary dependencies**

![](https://i1.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/a1b36f64-1a30-45bf-8fcd-4f88437c189e.jpg?ssl=1)

Remove unnecessary dependencies and do not install debugging tools. If needed debugging tools can always be installed later. Certain package managers such as apt, automatically install packages that are recommended by the user-specified package, unnecessarily increasing the footprint. Apt has the –no-install-recommends flag which ensures that dependencies that were not actually needed are not installed. If they are needed, add them explicitly.

**Tip \#5: Remove package manager cache**

![](https://i0.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/363961a4-005e-46fc-963b-f7b690be12ef.jpg?ssl=1)

Package managers maintain their own cache which may end up in the image. One way to deal with it is to remove the cache in the same RUN instruction that installed packages. Removing it in another RUN instruction would not reduce the image size.

There are further ways to reduce image size such as multi-stage builds which will be covered at the end of this blog post. The next set of best practices will look at how we can optimize for maintainability, security, and repeatability of the Dockerfile.

### Maintainability

**Tip \#6: Use official images when possible**

![](https://i1.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/f336014d-d2aa-4c1b-a2bd-e1d5d6ed0d93.jpg?ssl=1)

Official images can save a lot of time spent on maintenance because all the installation steps are done and best practices are applied. If you have multiple projects, they can share those layers because they use exactly the same base image.

**Tip \#7: Use more specific tags**

![](https://i1.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/9d991da9-bdb9-4108-8b36-296a5a3772aa.jpg?ssl=1)

Do not use the latest tag. It has the convenience of always being available for official images on Docker Hub but there can be breaking changes over time. Depending on how far apart in time you rebuild the Dockerfile without cache, you may have failing builds.

Instead, use more specific tags for your base images. In this case, we’re using openjdk. There are a lot more tags available so check out the [Docker Hub documentation](https://hub.docker.com/_/openjdk) for that image which lists all the existing variants.

**Tip \#8: Look for minimal flavors**

![](https://i2.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/6c486200-5198-4457-86c0-b5275e70e699.jpg?ssl=1)

Some of those tags have minimal flavors which means they are even smaller images. The slim variant is based on a stripped down Debian, while the alpine variant is based on the even smaller Alpine Linux distribution image. A notable difference is that debian still uses GNU libc while alpine uses musl libc which, although much smaller, may in some cases cause compatibility issues. In the case of openjdk, the jre flavor only contains the java runtime, not the sdk; this also drastically reduces the image size.

### Reproducibility <a id="h.hsy4wrgavnlq"></a>

So far the Dockerfiles above have assumed that your jar artifact was built on the host. This is not ideal because you lose the benefits of the consistent environment provided by containers. For instance if your Java application depends on specific libraries it may introduce unwelcome inconsistencies depending on which computer the application is built.

**Tip \#9: Build from source in a consistent environment**

The source code is the source of truth from which you want to build a Docker image. The Dockerfile is simply the blueprint.

![](https://i0.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/f393ad07-c25d-4241-a40f-c6168e0ba4dd.jpg?ssl=1)

You should start by identifying all that’s needed to build your application. Our simple Java application requires Maven and the JDK, so let’s base our Dockerfile off of a specific minimal official maven image from Docker Hub, that includes the JDK. If you needed to install more dependencies, you could do so in a RUN step.

The pom.xml and src folders are copied in as they are needed for the final RUN step that produces the app.jar application with `mvn package`. \(The -e flag is to show errors and -B to run in non-interactive aka “batch” mode\).

We solved the inconsistent environment problem, but introduced another one: every time the code is changed, all the dependencies described in pom.xml are fetched. Hence the next tip.

**Tip \#10: Fetch dependencies in a separate step**

![](https://i0.wp.com/www.docker.com/blog/wp-content/uploads/2019/07/41ea71ce-11c3-42a3-8d2b-05fe20901745.jpg?ssl=1)

By again thinking in terms of cacheable units of execution, we can decide that fetching dependencies is a separate cacheable unit that only needs to depend on changes to pom.xml and not the source code. The RUN step between the two COPY steps tells Maven to only fetch the dependencies.

There is one more problem that got introduced by building in consistent environments: our image is way bigger than before because it includes all the build-time dependencies that are not needed at runtime.

**Tip \#11: Use multi-stage builds to remove build dependencies \(recommended Dockerfile\)**

```text
FROM golang:1.10 as builder

WORKDIR /tmp/go
COPY hello.go ./
RUN CGO_ENABLED=0 go build -a -ldflags '-s' -o hello

FROM scratch
CMD [ "/hello" ]
COPY --from=builder /tmp/go/hello /hello
```

Multi-stage builds are recognizable by the multiple FROM statements. Each FROM starts a new stage. They can be named with the AS keyword which we use to name our first stage “builder” to be referenced later. It will include all our build dependencies in a consistent environment.

The second stage is our final stage which will result in the final image. It will include the strict necessary for the runtime, in this case a minimal JRE \(Java Runtime\) based on Alpine. The intermediary builder stage will be cached but not present in the final image. In order to get build artifacts into our final image, use `COPY --from=STAGE_NAME`. In this case, STAGE\_NAME is builder.

Multi-stage builds is the go-to solution to remove build-time dependencies.

#### Tip \#12: Use a linter

Adopt the use of a linter to avoid common mistakes and establish best practice guidelines that engineers can follow in an automated way.

One such linter is [hadolint](https://github.com/hadolint/hadolint). It parses a Dockerfile and shows a warning for any errors that do not match its best practice rules.

#### Tip \#13: Use metadata labels

Image labels provide metadata for the image you’re building. This help users understand how to use the image easily. The most common label is “maintainer”, which specifies the email address and the name of the person maintaining this image. Add metadata with the following `LABEL` command:

```text
LABEL maintainer="me@acme.com"
```

In addition to a maintainer contact, add any metadata that is important to you. This metadata could contain: a commit hash, a link to the relevant build, quality status \(did all tests pass?\), source code, a reference to your SECURITY.TXT file location and so on.

It is good practice to adopt a [SECURITY.TXT](https://securitytxt.org/) \(RFC5785\) file that points to your responsible disclosure policy for your Docker label schema when adding labels, such as the following:

```text
LABEL securitytxt="https://www.example.com/.well-known/security.txt"
```

See more information about labels for Docker images: https://label-schema.org/rc1/

#### Tip \#14: Use COPY instead of ADD

Docker provides two commands for copying files from the host to the Docker image when building it: `COPY` and `ADD`. The instructions are similar in nature, but differ in their functionality:

* COPY — copies local files recursively, given explicit source and destination files or directories. With COPY, you must declare the locations.
* ADD — copies local files recursively, implicitly creates the destination directory when it doesn’t exist, and accepts archives as local or remote URLs as its source, which it expands or downloads respectively into the destination directory.

While subtle, the differences between ADD and COPY are important. Be aware of these differences to avoid potential security issues:

* When remote URLs are used to download data directly into a source location, they could result in man-in-the-middle attacks that modify the content of the file being downloaded. Moreover, the origin and authenticity of remote URLs need to be further validated. When using COPY the source for the files to be downloaded from remote URLs should be declared over a secure TLS connection and their origins need to be validated as well.
* Space and image layer considerations: using COPY allows separating the addition of an archive from remote locations and unpacking it as different layers, which optimizes the image cache. If remote files are needed, combining all of them into one RUN command that downloads, extracts, and cleans-up afterwards optimizes a single layer operation over several layers that would be required if ADD were used.
* When local archives are used, ADD automatically extracts them to the destination directory. While this may be acceptable, it adds the risk of zip bombs and [Zip Slip vulnerabilities](https://snyk.io/research/zip-slip-vulnerability) that could then be triggered automatically.

#### Tip \#15: Use a .dockerignore file

Just like `.gitignore` , `.dockerignore` helps us not add things that we dont want or need in our docker image.

#### Tip \#16: Least privileged user

When a `Dockerfile` doesn’t specify a `USER`, it defaults to executing the container using the root user. In practice, there are very few reasons why the container should have root privileges. Docker defaults to running containers using the root user. When that namespace is then mapped to the root user in the running container, it means that the container potentially has root access on the Docker host. Having an application on the container run with the root user further broadens the attack surface and enables an easy path to privilege escalation if the application itself is vulnerable to exploitation.

To minimize exposure, opt-in to create a dedicated user and a dedicated group in the Docker image for the application; use the `USER` directive in the `Dockerfile` to ensure the container runs the application with the least privileged access possible.

A specific user might not exist in the image; create that user using the instructions in the `Dockerfile`.  
The following demonstrates a complete example of how to do this for a generic Ubuntu image:

```text
FROM ubuntu
RUN mkdir /app
RUN groupadd -r lirantal && useradd -r -s /bin/false -g lirantal lirantal
WORKDIR /app
COPY . /app
RUN chown -R lirantal:lirantal /app
USER lirantal
CMD node index.js
```

 

  


