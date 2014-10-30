title: Docker Demo for Digital Science Tech Forum
author:
  name: John Lees-Miller
  twitter: jdleesmiller
  url: http://jdlm.info
output: index.html
controls: true
progress: true

--

# Docker Demo

## Digital Science Tech Forum
## 30 Oct 2014

--

### The Plan

1. A brief overview of docker.

1. Run a simple web app in docker.

1. Run some scientific software in docker.

--

### Mental Model

Here's my (oversimplified) model for using docker.

1. You build a disk **image**.

1. You create a **container** to run a **command** in the context of the image.

1. When you're finished, you throw away the container (but you'll probably reuse the image).

--

### Use a `Dockerfile` to Build an Image

Sort of like an ansible playbook or chef recipe.

```
FROM ubuntu:14.04                          # base image

MAINTAINER overleaf <team@overleaf.io>     # who are you?

RUN apt-get update && apt-get -y upgrade   # image setup commands
RUN apt-get install -y ruby
RUN gem install sinatra
RUN mkdir /app

ADD hello_world.rb /app/                   # copy files to image

ENV PORT 8000                              # set environment vars
ENV RACK_ENV production
```

--

### Building an Image

To build the example web app:

```
$ cd examples/web_app
$ docker build --tag hello_world .
```

`--tag` is how we'll refer to the image later.

`.` tells docker that the `Dockerfile` and `ADD`ed files are in the current directory.

--

### Images are like git Commits

Each command in the Dockerfile makes a new image.

<pre>
Step 0 : FROM ubuntu:14.04
...
Status: Downloaded newer image for ubuntu:14.04
 ---> <b>5506de2b643b</b>

Step 1 : MAINTAINER overleaf <team@overleaf.io>
...
 ---> <b>0bf3ae3662b4</b>

Step 2 : RUN apt-get update && apt-get -y upgrade
...
 ---> <b>296a361125bd</b>
</pre>

Each image is identified by a **hash** of its *content* and its *ancestor*, like a git commit.

--

### Only "Diffs" of Images are Stored

```
$ docker history hello_world
IMAGE               CREATED             CREATED BY                      SIZE
203c183acb03        2 minutes ago       ... #(nop) CMD [/bin/sh ...      0 B
d1df6bf66499        2 minutes ago       ... #(nop) ENV RACK_ENV=...      0 B
24a3db493063        2 minutes ago       ... #(nop) ENV PORT=3000...      0 B
e567e577d823        2 minutes ago       ... #(nop) ADD file:4164...     52 B
5695586ee7a3        2 minutes ago       ... mkdir /app          ...      0 B
a6447b00f7df        2 minutes ago       ... gem install sinatra ...   16.7 MB
60d6e9ae2318        3 minutes ago       ... apt-get install -y r...  17.67 MB
296a361125bd        4 minutes ago       ... apt-get update && ap...  33.49 MB
0bf3ae3662b4        5 minutes ago       ... #(nop) MAINTAINER ov...      0 B
5506de2b643b        5 days ago          ... #(nop) CMD [/bin/bas...      0 B
22093c35d77b        5 days ago          ... apt-get update && ap...  6.558 MB
3680052c0f5c        5 days ago          ... sed -i 's/^#\s*\(deb...  1.895 kB
e791be0477f2        5 days ago          ... rm -rf /var/lib/apt/...      0 B
ccb62158e970        5 days ago          ... echo '#!/bin/sh' > /...  194.8 kB
d497ad3926c8        8 days ago          ... #(nop) ADD file:3996...  192.5 MB
511136ea3c5a        16 months ago                                        0 B
```

The `hello_world` tag points to image `203c183acb03`.

--

### Run a Command in a Container

```
docker run -it --rm --publish 3000:3000 hello_world ruby /app/hello_world.rb
                                      # ^           ^ command in the container
                                      # ^ image tag
```

'--rm' removes the container once the command has exited

`--publish` forwards port 3000 on the host from port 3000 in the container

`-it` means interactive tty (terminal); this just lets us use Ctrl-C to interrupt the server.

--

### `docker ps` Lists Running Containers

<pre>
$ docker ps
CONTAINER ID        IMAGE                COMMAND                CREATED       \
fa23530d3502        hello_world:latest   "ruby /app/hello_wor   4 seconds ago \

STATUS              PORTS                    NAMES
Up 3 seconds        0.0.0.0:3000->3000/tcp   backstabbing_shockley
</pre>

The name is randomly generated for easy reference, e.g. to stop the container:

```
docker kill backstabbing_shockley
```

--

### Incremental Rebuilds are Cheap

If we change `examples/web_app/hello_world.rb` to output a different message and rebuild the container, docker reuses cached images for the preceeding steps.

<pre>
$ sed -i 's/World/Docker/' hello_world.rb
$ docker images -ta
...
Step 4 : RUN gem install sinatra
 ---> <b>Using cache</b>
 ---> a6447b00f7df
Step 5 : RUN mkdir /app
 ---> <b>Using cache</b>
 ---> 5695586ee7a3
Step 6 : ADD hello_world.rb /app/
 ---> 4c535a0a4a01 <em># now things have changed</em>
...
</pre>

--

### Images form a Tree

<pre>
$ docker images -ta
└─511136ea3c5a Virtual Size: 0 B
  ├─d497ad3926c8 Virtual Size: 192.5 MB
    ...
      └─5506de2b643b Virtual Size: 199.3 MB Tags: ubuntu:14.04
        ...
          └─5695586ee7a3 Virtual Size: 267.1 MB         <em># RUN mkdir /app</em>
            ├─4c535a0a4a01 Virtual Size: 267.1 MB       <em># ADD hello_world...</em>
            │ └─e7352ee9bedd Virtual Size: 267.1 MB
            │   └─fb17a307e866 Virtual Size: 267.1 MB Tags: hello_world:latest
            └─e567e577d823 Virtual Size: 267.1 MB       <em># ADD hello_world...</em>
              └─24a3db493063 Virtual Size: 267.1 MB
                └─d1df6bf66499 Virtual Size: 267.1 MB
                  └─203c183acb03 Virtual Size: 267.1 MB <em># <-- the old hello_world</em>
</pre>

--

### More About Containers

You can run a shell in a container to explore or use an image interactively:

    docker run -it hello_world /bin/bash

If you make changes in the container and kill it (but don't remove it), you can restart it later, and your changes will persist.
This can be useful in development, but docker has other solutions for data persistence that are better in production; see [data volumes](https://docs.docker.com/userguide/dockervolumes/).

--

### Web App Example Improvements

* The process runs as root inside our container, which is bad. Docker can run the process as an [unprivileged user](http://docs.docker.com/reference/run/#user) instead (exercise).

* We're using system ruby on Debian, which is far out of date. It's usually better to install with [RVM](http://rvm.io/) (exercise) or [find a docker image in the registry](https://registry.hub.docker.com/) with ruby installed.

--

### LaTeX in a Container

Because creating and destroying containers is very fast, we can use them for short-lived processes, too.

Example (`examples/latex/Dockerfile`):

```
FROM ubuntu:14.04

MAINTAINER overleaf <team@overleaf.io>

RUN apt-get update && apt-get -y upgrade

RUN apt-get install -y texlive-latex-base

WORKDIR /tmp # by default, start commands in the container in /tmp
```

--

### LaTeX in a Container

To build and run:

```
cd examples/latex
docker build --tag texlive .
docker run -it --rm --volume `pwd`:/tmp texlive pdflatex main
```

`--volume` mounts the current working directory on the host as `/tmp` inside the container.

`pdflatex main` is the command that runs inside the container, to compile a file called `main.tex`.

--

### LaTeX Example Improvements

* Again, we shouldn't be running the compile as root in the container, but that requires getting the permissions on the host to match up with those in the container.

* We could install a lot more software. A full install of TeX Live is over 4GB, and there are many related packages.

--

### Docker at writeLaTeX

* We maintain a Dockerfile that builds an image with latex and lots of related tools.

* Each compile job gets its own short-lived container.

* This adds another layer to our security system.
