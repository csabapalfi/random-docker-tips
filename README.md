# 24 random docker tips

[Csaba Palfi](https://csabapalfi.github.io), Dec 2014

We love docker and had it in production since 0.8 at [TES Global](http://www.tesglobal.com/) (my current client). Couple of us could attend the trainings at dockerConEU thanks to [Contino](http://contino.co.uk/). Here are some of the tips and tricks that will hopefully be useful for anyone who is already familiar with docker basics.

## 1. CLI

### 1.1. nice docker ps output

Just pipe docker ps output to less -S so that the table rows are not wrapped:

```sh
docker ps -a | less -S
```

### 1.2 follow the logs

docker logs won't watch the logs by default unless use the -f flag:

```sh
docker logs <containerid> -f
```

### 1.3 a single value from docker inspect

docker inspect spits out a lots of JSON by default. You can just use jq to extract a single key. Or you can use the builtin go templating in docker inspect like below:

```sh
# is the last run container still running?
docker inspect --format '{{.State.Running}}' $(docker ps -lq)
```

### 1.4 ```docker exec``` instead of sshd or nsenter

This one is pretty well-known if you follow the docker releases. ```exec``` was introduced in 1.3 and allow you to run a new process within a container. There's no need for running sshd in the container or installing nsenter on the host anymore.

## 2. Dockerfiles

### 2.1 docker build supports git repos

You can not only build images from local Dockerfiles but can simply give docker build a git repo URL and it takes care of the rest.

### 2.2. no package lists

Default images (e.g. ubuntu) don't include package lists to keep them smaller hence the ```apt-get update``` in pretty much any base Dockerfile.

### 2.3 watch out for package versions

Be careful with package installations as those commands are cached as well. Meaning you may get different version when busting the cache or lagging behind on security updates when caching these for too long.

### 2.4 small base images

There's an official, truly empty docker image on docker hub. It's called [scratch](https://registry.hub.docker.com/_/scratch/). If you want you can start your images ```FROM scratch```. Most of time you're better off starting from [alpine](https://registry.hub.docker.com/_/alpine/) if you want a really small base image (few MBs) as it has a shell and nice package manager, too.

### 2.5 FROM is latest by default

If you don't specify a version in your tag then the FROM keyword will just use latest. Be careful with that and make sure you specify a version if you can.

### 2.6. shell or exec mode

There are a few places where you can specify commands in your Dockerfile. (e.g. ```CMD```, ```RUN```). docker supports two ways of doing that. If you just write the command then docker will wrap it in ```sh -c```. You can also write them as an array of strings (e.g ```[ "ls", "-a"]```). The array notation won't need a shell to be available within the container (as it uses go's exec) and that's the preferred syntax according to the docker guys.

### 2.7. ADD vs COPY

Both ```ADD``` and ```COPY``` adds local files when building a container but ADD does some additional magic like adding remote files and ungzipping and untaring archives. Only use ```ADD``` if you understand this difference.

### 2.8 WORKDIR and ENV

Each command will create a new temporary image and runs in a new shell hence if you do a `cd directory` or `export var=value` in your Dockerfile it won't work. Use `WORKDIR` to set your working directory across multiple commands and ENV to set environment variables.

### 2.9 CMD and ENTRYPOINT

`CMD` is the default command to execute when an image is run. The default `ENTRYPOINT` is `/bin/sh -c` and `CMD` is passed into that as an argument. We can override ENTRYPOINT in our Dockerfile and make our container behave like an executable taking command line arguments (with default arguments in CMD in our Dockerfile).

```
# in Dockerfile
ENTRYPOINT /bin/ls
CMD ["-a"]

# we're overriding the command but entrypoint remains ls
docker run training/ls -l
```

### 2.10 ```ADD``` your code last

ADD invalidates your cache if files have changed. Don't invalidate the cache by adding frequently changing stuff too high up in your Dockerfile. Add your code last, libraries and dependencies first. For node.js apps that means adding your package.json first, running npm install and only then adding your code.

## 3. docker networking

Docker has an internal pool of IPs which it uses for container IP addresses. These are invisible to the outside by default and accessible via a bridge interface.

### 3.1 looking up port mappings

```docker run``` accepts explicit port mappings as parameters or you can specify ```-P``` to map all ports automatically. The latter has the advantage of preventing conflicts and looking up the assigned ports can be done as follows:

```sh
docker port <containerId> <portNumber>
# or
docker inspect --format '{{.NetworkSettings.Ports}}' <containerId>
```

### 3.2 container IPs

Each container has it's IP in a private subnet (which is 172.17.0.0/16 by default). The IP can change with restart but can be looked up should you need it:

```
docker inspect --format '{{.NetworkSettings.IPAddress}}' <containerId>
```

docker tries to detect conflicts and will use a different subnet if needed.

### 3.3 taking over the hosts network stack

```docker run --net=host``` allows reusing the network stack of the host. Don't do this.

## 4. volumes

A way to bypass copy-on-write filesystem for a directory or a single file with close to zero overhead (bind mounting).

### 4.1 volume contents are not saved on docker commit

There's not much point in writing to your volumes when the image is built.

### 4.2 volumes are read-write by default

but there's an :ro flag

### 4.3 volumes exists separately from containers

And available until at least container references them. Can be shared between container with ```--volumes-from```.

### 4.4 mount your docker.sock

You can just mount your docker.sock to provide a container access to the docker API. You can then just run docker commands from within that container. This way a container can even kill itself. There's no need to run the docker daemon within a container.


## 5. security

### 5.1. docker runs as root...

...treat it accordingly. Docker API access gives full root access as you can map ```/``` as a volume, read, write. Or you can just take over the host's network with ```--net host```. Don't expose docker API to public or use TLS if you do.

### 5.2 USER in Dockerfiles

By default docker runs everything as root but you can use USER in Dockerfiles. There's no user namespacing in docker so the container sees the users on the host but only uids hence you need the add the users in the container.

### 5.3 use TLS on the docker API

There was no access control on the docker API until 1.3 when they added TLS. They use mutual authentication: the client and server both has a key. Treat keys as root passwords.

Boot2docker has TLS as default since 1.3 and also generates the keys for you.

Otherwise generating keys requires OpenSSL 1.0.1 then the docker daemon needs to be run with --tls-verify and will use the secure docker port (2376).

We're hopefully getting more granular access control soon instead of all or nothing.

![](https://ga-beacon.appspot.com/UA-29212656-1/random-docker-tips?pixel)
