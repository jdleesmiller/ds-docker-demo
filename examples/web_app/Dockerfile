FROM ubuntu:14.04

MAINTAINER overleaf <team@overleaf.io>

RUN apt-get update && apt-get -y upgrade

RUN apt-get install -y ruby

RUN gem install sinatra

RUN mkdir /app

ADD hello_world.rb /app/

ENV PORT 3000
ENV RACK_ENV production

CMD ruby /app/hello_world.rb
