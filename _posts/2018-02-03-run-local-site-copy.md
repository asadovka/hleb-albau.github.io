---
layout: post
title:  "Run a jekyll blog with custom plugins locally using Docker"
date:   2018-01-28
excerpt: ""
---

## What's the problem ?

Github pages supports only small subset of jekyll gems. In order to enable custom gems, you should build site yourself 
and provide to github "compiled" **_site** folder with ready to serve htmls. 
[Here](http://joshfrankel.me/blog/deploying-a-jekyll-blog-to-github-pages-with-custom-plugins-and-travisci/) you can 
find very nice article on how to automatize your build of blog with custom plugins using 
[TravisCi](https://travis-ci.org/). But, how can I test changes locally, before publishing them?

 Jekyll is written using ruby language. So, to build your site, you should install some build-deps, ruby and plugin gems
  to your environment machine. But... what if I am not a ruby developer. All I want is just to test my blog on changes 
  locally and keep my system clean?


## Build your site using Docker

 Let's use [Docker](https://www.docker.com/) as a sandbox for local site assembly. We will use 2 images to speed up 
 serve process. One will contain all dependencies and requires rebuild only on plugins configurations
  change. The second will serve generated site. 

### Image with required dependencies

  The first thing we will do, is to define image, containing all required dependencies. 

{% prism docker %}
# local-dependencies file
FROM ruby:2.5-alpine

RUN apk --update add --virtual build_deps build-base ruby-dev libc-dev linux-headers

COPY . /jekyll-build
WORKDIR /jekyll-build

RUN bundle install
RUN bundle list
{% endprism %} 

Within default ruby image we install required dev tools, copy our repo with gem definitions(Gemfile) inside container
 and download listed gems. Now, lets build it.
 
{% prism bash %}
# context is your git repo with site sources
docker build -f local-dependencies -t hleb-albau-local-dependencies .
{% endprism %}
 
As a result, your local docker registry will contain image with name *hleb-albau-local-dependencies*.
 
> You should rebuild **local-dependencies** image each time you add new jekyll gems


### Image with jekyll serve

Next step is to define image serving generated site.

{% prism docker %}
# local-run file
FROM hleb-albau-local-dependencies

WORKDIR /jekyll-build

EXPOSE 4000
CMD jekyll serve -d /_site --watch --force_polling -H 0.0.0.0 -P 4000
{% endprism %}

Here we use previously created image **local-dependencies** as a base image, expose 4000 port and run `jekyll serve`
 command. It's all. Now, we are ready to browser our site on **localhost:4000** using next command:
 
{% prism bash %}
# context is your git repo with site sources  
docker build -f local-run -t hleb-albau-local-run . \
&& docker run -it --rm -v ${PWD}:/jekyll-build -p "4000:4000" hleb-albau-local-run
{% endprism %}
  
Also, `jekyll serve` command regenerates your site on every change you made, so there is no need to restart image. You
 can test your changes on the fly. Happy blogging!   