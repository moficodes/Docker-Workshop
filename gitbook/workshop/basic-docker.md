# Basic Docker

#### Run "The" Hello World Container

```text
docker run hello-world
```

#### RUN a "Hello World" container

```text
docker run alpine echo "Hello World"
```

* If the Image is not cached, it pulls it automatically
* It prints `Hello World` and exits

#### RUN an interactive Container

```text
docker run -it alpine sh
cat /etc/os-release
```

* **-i**: Keep stdin open even if not attached
* **-t**: Allocate a pseudo-tty

#### RUN a Container with pipeline

```text
cat /etc/resolv.conf | docker run -i alpine wc -l
```

#### SEARCH a Container

```text
docker search --filter=stars=50 nginx
```

* **--filter=stars=x**: Only displays with at least x stars

#### Expose a port

```text
docker container run --detach --publish 8080:80 --name nginx nginx
```

* --detach, -d = detach
* --publish, -p = publish
* --name, -n = name the container \(by default they get auto generated name\)

