---
layout: post
title: "Hubot: A bot for automating daily tasks or anything else"
date: 2015-01-02 19:23:50 +0100
comments: true
categories: [ hubot, devops, chatops, git, slack, docker ]
---


> Hubot is your company's robot. Install him in your company to dramatically improve and reduce employee efficiency.

In other words, [hubot](https://hubot.github.com/) is a chat bot crafted by github team that can run custom written scritps. This allow us to automatize any kind of tasks like merging branches, deploying releases, do monitoring queries, inform (it listen on a port), etc. It has a lot of adapters (where it reads from and writes to), even shell, so we can enjoy its company almost everywhere.

It is written in [coffescript](http://coffeescript.org/) and ran with nodejs.

## Hubot setup

In order to ease this process, we are going to setup and deploy it with docker. Following Dockerfile allowed me to run and test Hubot.

```text hubot_dockerfile https://gist.github.com/jancorg/2d14912f35f756b97912/
FROM dockerfile/nodejs
MAINTAINER  Marvin
 
WORKDIR /root
RUN npm install -g yo generator-hubot
 
RUN useradd -ms /bin/bash marvin
ENV HOME /home/marvin
# variables needed by hubot scripts 
ADD env-vars.sh /home/marvin/.profile
RUN chown marvin /home/marvin/.profile

USER marvin
WORDIR /home/marvin
RUN echo n | yo hubot --defaults
RUN npm install hubot-slack hubot-scripts githubot --save
 
# enable plugins
RUN echo [ \'github-merge.coffee\' ] > hubot-scripts.json
 
CMD ["/home/marvin/bin/hubot", "--name", "marvin"]
 
EXPOSE 8080
```

Build an image with `docker build -t cname/hubot .` or just use the commands for testing it. Nodejs setup instructions can be found on [joyent's nodejs repo](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager)

This also installs [slack adapter](https://github.com/slackhq/hubot-slack), [hubot-scripts](https://github.com/github/hubot-scripts) and [githubot](https://github.com/iangreenleaf/githubot) npm packages, hence you can figure out how next steps will be.

Let me introduce you to Marvin, my hubot instance, named like Hitchhicker's guide to galaxy assistant robot.

``` text 
                     _____________________________  
                    /                             \ 
   //\              |      Extracting input for    |
  ////\    _____    |   self-replication process   |
 //////\  /_____\   \                             / 
 ======= |[^_/\_]|   /----------------------------  
  |   | _|___@@__|__                                
  +===+/  ///     \_\                               
   | |_\ /// HUBOT/\\                             
   |___/\//      /  \\                            
         \      /   +---+                            
          \____/    |   |                            
           | //|    +===+                            
            \//      |xx|                            

marvin> marvin: the rules
1. A robot may not injure a human being or, through inaction, allow a human being to come to harm.
2. A robot must obey any orders given to it by human beings, except where such orders would conflict with the First Law.
3. A robot must protect its own existence as long as such protection does not conflict with the First or Second Law.
```
[Janky](https://github.com/github/janky) is a hithub project that implements CI on top Jenkins and Hubot. Chatops term is used to refer chat based deployments and monitoring.  

## Github integration

One of the things I found interesting using hubot for is easy merging and deploy process.
In this article I am just going to treat merging as an example, but concepts shown here could be easily used for integrating it with jenkins or any other tools.

We can find some git related scripts under scripts folder in [hubot-scripts](https://github.com/github/hubot-scripts) github project. 
This time I have tested [github-merge.coffee](https://github.com/github/hubot-scripts/blob/master/src/scripts/github-merge.coffee) plugin. It uses [githubot](https://github.com/iangreenleaf/githubot) project for github API access.

We need to set some variables in order this to work. They are made available to hubot as environment variables, so you can set them on `env-vars.sh` script mentioned in Dockerfile.

``` sh 
export HUBOT_GITHUB_TOKEN=$token
export HUBOT_GITHUB_API_VERSION=$api_version
export HUBOT_GITHUB_USER=$gh_user
export HUBOT_GITHUB_API=$gh_url
export HUBOT_CONCURRENT_REQUESTS=$req_limit
```

Enabling a plugin is as easy as adding it to `hubot-scripts.json` or `external-scripts.json`.
``` json 
[ 'github-merge.coffee' ]
``` 

Once configured we can check if plugin was loaded sucessfuly and works properly.
``` text
marvin> marvin help
[...]
marvin merge project_name/<head> into <base> - merges the selected branches or SHA commits
[...]
marvin> marvin merge nevermind/PXTRM-test-switch-bug into develop
marvin> Merge PXTRM-test-switch-bug into develop
marvin> marvin merge nevermind/PXTRM-merge-conflict-on-purpose into develop
marvin> [Fri Jan 02 2015 17:03:46 GMT+0000 (UTC)] ERROR 409 Merge conflict
```

![hubot_merged_branch](/images/2015-01-02-hubot/hubot_merged_branch.png)

It seems to work, and since it is using [github merge API](https://developer.github.com/v3/repos/merging/) we are not messing any branch up.


## Slack integration

I have chosen slack as default adapter since it is chat tool we are using at office. Hubots default one is [campfire](https://github.com/github/hubot/tree/master/src/adapters).

An organization, if you already don't have one, needs to be created on slack. You can do it on sign up process.
When this is done, just configure integrations and look for hubot there. You will get a token and even you can set a custom avatar for you shinny bot.
Make this token availabe as environment varaible, and... that's it.
```sh
export HUBOT_SLACK_TOKEN=$slack_token
```

Both processes are really straighforward.

Let's see the results.

![joins_channel](/images/2015-01-02-hubot/hubot_joins_channel.png)

![hubot_merge_and_plugin_example](/images/2015-01-02-hubot/hubot_merge_and_plugin_example.png)


Enjoy your bot! 







