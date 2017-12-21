---
layout: post
author: Kévin Maschtaler
title: "Efficient Deployments Thanks To Configurable Artifacts"
date: 2017-12-21 16:00:00
image: /blog/content/partial-deployment-pipeline.png
tags:
- devops
---

As your project and your team grows, feature deployment becomes a critical part of your development process.
Knowing how to deploy quickly and safely to production can be a lifesaver skill for your company.
To this end, let me introduce you the artifact-based deployment process!
<!--more-->


## What Is A Deployment Artifact

Artifacts aren't a new thing, but like every good practice, it's better when written down.
This article aims to help teams that haven't automated their deployment yet.
So, let's start with the basics.

A deployment artifact (or a `build`) is your application code as it runs on production: compiled, built, bundled, minified, optimized and so on ; most often in a compressed folder or a binary.

In such a state, you can store and version it. The ultimate goal of an artifact is to be downloaded as fast as possible on a server and ran immediately with no service interruption.

Also, it should be configurable (we'll see how later) in order to be deployable on any environment. It is the same artifact that must be deployed on staging then on production, for example.

Yes, you read it right: Only the configuration must change, not the artifact itself. It can seem harmful or difficult, but it's the main feature of a deployment artifact.
If you have to build twice your artifact for two environments, you are missing the whole point.


## Lifecycle of an Artifact

Say we have a simple project to deploy, like a basic HTTP proxy in order to add a random HTTP header to every requests.
The code is available in this gist: [Kmaschta/0920c6a7781cdf15c37a51b370e4fb66](https://gist.github.com/Kmaschta/0920c6a7781cdf15c37a51b370e4fb66)

First install the dependencies and write the little proxy:
```bash
$ mkdir deployment-example && cd deployment example
$ npm install --save express axios reify
```

```js
// server.js
import uuid from 'uuid';
import axios from 'axios';
import express from 'express';

const port = process.env.NODE_PORT || 3000;
const baseURL = process.env.PROXY_HOST || 'http://perdu.com/';

const client = axios.create({ baseURL });
const app = express();

app.get('/', (req, res) => {
    const requestOptions = {
        url: req.path,
        method: req.method.toLowerCase(),
        headers: { 'X-Random': uuid.v4() },
    };

    return client(requestOptions).then(response => res.send(response.data));
});

app.listen(port);
console.log(`Proxy is running on port ${port}. Press Ctrl+C to stop.`);
```

```makefile
# Makefile
start:
    # reify is a small lib to fully support import/export without having to install the babel suite
    node --require reify server.js
```

The server is runnable on local environment, it's all good, "it works on my machine™":
```bash
$ make start
> Proxy is running on port 3000. Press Ctrl+C to stop.
```

Now let's ship this code on a staging environment to test it with somewhat real conditions. At this point, it's a good idea to freeze a version of this code with a GitHub release for example.

The simplest way to prepare the deployment is to **build** a zip file with the source code and all its dependencies.

```makefile
.PHONY: build

build:
    mkdir -p build
    zip 'build/$(shell git log -1 --pretty="%h").zip' Makefile package.json -R . '*.js' -R . '*.json'
```

```bash
$ make build
$ ls build/
> 4dd370f.zip
```

The build step is simplified here, but on real projects you can use a bundler, a transpiler, a minifier, and so on.
All these lengthy tasks should be done at the build step.

The resulting zip file is what we can call an **artifact**.
It can be deployed on an external server or be stored in a S3 bucket for later usage.

Once you find the build process that fits you, automate it!
Usually, a Continuous Integration / Continuous Delivery (CI/CD) system like Travis or Jenkins runs the tests, and if they pass, build the artifact in order to store it.

Now to deploy the artifact, just copy this file on the server, extract it and run the code.

```makefile
TAG ?=
SERVER ?= staging-server

deploy:
    scp build/$(TAG).zip $(SERVER):/data/www/deployment-example/
    ssh $(SERVER) " \
        cd /data/www/deployment-example/ \
        unzip $(TAG).zip -d $(TAG)/ && rm $(TAG).zip \      # unzip the code in a folder
        cd current/ && make stop \                          # stop the current server
        cd ../ && rm current/ && ln -s $(TAG)/ current/ \   # move the symbolic link to the new version
        cd current/ && make start \                         # restart the server
    "
    echo 'Deployed $(TAG) version to $(SERVER)'
```

```bash
$ TAG=4dd370f make deploy
> Deployed 4dd370f version to staging-server
```

**Tip:** Use your local <code>~/.ssh/config</code> to save your server credentials.
This way you can securely share them with your co-workers and keep these credentials outside of the repository while having the deployment instructions in the the Makefile.

```config
Host staging-server
    Hostname staging.domain.me
    User ubuntu
    IdentityFile ~/.ssh/keys/staging.pem
```

As you can see, the deployment is actually very fast. What if we want to deploy the same build on production with a different port?
Nothing could be simpler. All we need is to be sure that when the code is run, the environment variable `NODE_PORT` is set to the correct value.

There is many ways to achieve this **configuration**, the simplest one is to directly write your environment variables in the file `~/.ssh/environment` of your server.
In this case, you don't need to remember how or when to retrieve your configuration: it's loaded automatically each time you log into the machine.

```bash
> ssh production-server "echo 'NODE_PORT=8000' >> ~/.ssh/environment"
> ssh production-server "echo 'PROXY_HOST=https://host.to.proxy' >> ~/.ssh/environment"
> ssh production-server
> env
NODE_PORT=8000
PROXY_HOST=https://host.to.proxy
```

Now you just have to deploy on the production server the same artifact that you've used for the staging, no need to rebuild.

```bash
$ TAG=4dd370f SERVER=production-server make deploy
> Deployed 4dd370f version to production-server
```

That's it. The **rollback** is as simple as a deployment of the previous artifact.

At this point, it is easy to automate the process: let the CI build an artifact each time a PR is merged or on every push on master (Travis, Jenkins, every CI allows to implement a build phase) and store this build somewhere with a specific tag such as the commit hash or the release tag.

When somebody want to deploy, just run a script on the server that will download the artifact, configure it thanks to the environment variables and run it.


**Tip:** If you don't want to write your environment variables on every production server you have, you can take a look at a configuration manager like [comfygure](https://github.com/marmelab/comfygure)!
You can also read <a href="https://12factor.net/config">what the Twelve-Factor App guide says about configuration</a>.


## Cool! Should I Use This Method On My Project Right Now?

Artifact-based deployment comes with many advantages. It is quick to deploy, instant to rollback, the backups are available and ready to run.

It is the same code that runs on every environments, in fact the deployment becomes a configuration issue. If a bunch of code works on staging, there is no reason that it fails on production, unless there is a mistake in the configuration or your environments aren't the same. For that reason, it's a good idea to invest in a staging environment that is strictly equal to the production.

Feature flagging is easier with such a deployment process, just check the environment variables.

And last but not least, the deployment can be so automated that it can be done by a product owner and not by a developer. We do it, it's really great.

But automate such a process comes with a substantial cost: it takes time to install and maintain, and the extra space to store the artifacts and the backups cost money.

Moreover, this technique makes **stateless application** mandatory and the code must not depend of an environment or another. Try to find all `if (env === 'production')` in your code. That said, a stateless application is much easier to debug and scale.

No need of such a deployment process for a POC, an early project or a solo developer. All the advantages come with a team who works on a mature project.

Ask yourself how many time do you spend on deployment and take a look at this chart!

![xkcd: Is it worth the time?](https://imgs.xkcd.com/comics/is_it_worth_the_time.png)

**But my project is an SPA / PWA. I must configure my bundle at the build phase!**

You don't have to. There is many ways to configure a SPA later in the process, like read a `config.json` located elsewhere than your static assets or write a `window.__CONFIG__` through server side rendering, and so on.

Don't be afraid to display the frontend config, it'll be easier to debug. If you have sensible informations in your frontend configuration hidden in your minified and cryptic webpack build, you're already doing it wrong.

**And what about database migrations?**

Migrations don't have to be run with the application deployment but should have their own pipeline that can be triggered at the appropriate moment.
This way, you can handle each deployment on their own and rollback them independently.

It clearly implies to do backups a lot and have revertable migrations.
In case of big or painful migrations, don't hesitate to do it step by step. For example, how to move a table: first create the new table, then copy the data and finally delete the resulting table in three different migrations.
The application can be deployed between the copy and the table deletion. If something goes wrong, the application can be reverted quickly without touching the database.

## Conclusion

Once this process is correctly installed it allows teams and product owners to be more confident about deployment that becomes quicker and safer.

Artifact-based deployment is a powerful technique that is truly worth considering when a project gets close to its maturity.
It matches perfectly with agile methodologies like SCRUM, even more with Kanban, and it's a prerequisite to continuous delivery.

To go further with deployment techniques, I recommend huge article of Nick Craver: [Stack Overflow: How We Do Deployment - 2016 Edition](https://nickcraver.com/blog/2016/05/03/stack-overflow-how-we-do-deployment-2016-edition/).
