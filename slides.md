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

Sort of like a playbook (ansible) or recipe (chef).

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

### Images are like Commits

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

### Images form a Tree

Only the differences between images are stored.

<pre>
$ docker images -ta
└─511136ea3c5a Virtual Size: 0 B
  ├─d497ad3926c8 Virtual Size: 192.5 MB
    ...
           └─5506de2b643b Virtual Size: <b>199.3 MB</b> Tags: ubuntu:14.04
              └─0bf3ae3662b4 Virtual Size: <b>199.3 MB</b>     # <i>MAINTAINER ...</i>
                └─296a361125bd Virtual Size: <b>232.7 MB</b>   # <i>RUN apt-get update ...</i>
                  └─60d6e9ae2318 Virtual Size: <b>250.4 MB</b> # <i>RUN apt-get install ...</i>
                    └─a6447b00f7df Virtual Size: <b>267.1 MB</b>
                      ........
                              └─203c183acb03 Virtual Size: <b>267.1 MB</b> Tags: hello_world:latest
</pre>

The tagged image is the result of the build.

--

### Run a Command in a Container

Run a command in a container using our image:

```
docker run -it --publish 3000:3000 hello_world ruby /app/hello_world.rb
                                 # ^           ^ command in the container
                                 # ^ image tag
```

`--publish` port 3000 on the host from port 3000 in the container

`-it` means "interactive tty" (terminal); this just lets us use Ctrl-C to interrupt the server.

--

### `docker ps` Lists Running Containers

TODO
