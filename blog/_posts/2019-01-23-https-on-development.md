---
layout: post
author: Kévin Maschtaler
title: "HTTPS In Development: A Practical Guide"
excerpt: "Why and how to setup HTTPS certificates, with or without reverse proxy, in a JavaScript development environment."
date: 2019-01-23 18:00:00
image: /blog/content/https-on-dev/berlin-wall.jpeg
canonical: https://marmelab.com/blog/2019/01/23/https-in-development.html
tags:
- js
- node-js
- react
- security
- tutorial
---

According to Firefox Telemetry, [76% of web pages are loaded with HTTPS](https://letsencrypt.org/stats/#percent-pageloads), and this number is growing.

Sooner or later, software engineers have to deal with HTTPS, and the sooner the better. Keep reading to know why and how to serve a JavaScript application with HTTPS on your development environment.

[![HTTPS adoption according to Firefox Telemetry](/blog/content/https-on-dev//https-adoption-firefox.png)](https://letsencrypt.org/stats/#percent-pageloads)

<!--more-->

## Why Use HTTPS On a Development Environment?

First, should you serve a website in production through HTTPS at all? Unless you really know what your are doing, [the default answer is **yes**](https://doesmysiteneedhttps.com). It improves your website at so many levels: security, performance, SEO, and so on.

How to setup HTTPS is often adressed during the first release, and brings a lot of other questions. Should traffic be encrypted from end to end, or is encryption until the reverse proxy enough? How should the certificate be generated? Where should it be stored? What about [<abbr title="HTTP Strict Transport Security">HSTS</abbr>](https://developer.mozilla.org/fr/docs/S%C3%A9curit%C3%A9/HTTP_Strict_Transport_Security)?

The development team should be able to answer these questions early. If you fail to do so, you might [end up like Stack Overflow wasting a lot of time](https://nickcraver.com/blog/2017/05/22/https-on-stack-overflow/).

Besides, having a development environment as close as possible from the production reduces risks that bugs reach the production environment, and also tend to decrease the time to debug those bugs. It's also true for end-to-end tests.

In addition, there are features that only work on a page served by HTTPS, such as [Service Workers](https://developer.mozilla.org/fr/docs/Web/API/Service_Worker_API).

**But HTTPS is slow!** Many people believe that encryption is complicated and in a certain way must be slow to be efficient. But with modern hardware and protocols, [this is not true anymore](https://istlsfastyet.com/).

## How To Generate A Valid Certificate For A Development Environment?

For production systems, it's easy to get a <abbr title="Transport Layer Security">TLS</abbr> certificate: generate one from [Let's Encrypt](https://letsencrypt.org/) or buy one from a paid provider.

For the development environment, it seems trickier, but it isn't that hard.

### Mkcert: The No Brainer CLI

[Filippo Valsorda](https://blog.filippo.io/hi/) recently published [`mkcert`](https://github.com/FiloSottile/mkcert), a simple cli to generate locally-trusted development certificates. You just have to run a one-line command:

```bash
mkcert -install
mkcert example.com
```

The fully supported certificate will be available where your ran the command, namely at `./example.com-key.pem`.

### Manual Installation With OpenSSL

`mkcert` should fulfill all of your needs, unless you have to share the same certificate with your coworkers, or through other systems than your local env. In that case, you can generate your own certificate thanks to `openssl`.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt
```

The certificate (`server.crt`) and its key (`server.key`) will be valid but *self-signed*. This certificate will be unknown to any [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority). But all browsers ask well-known certificate authorities to validate certificates in order to accept encrypted connections. For a self-signed certificate, they can't validate it, so they display an annoying warning:

![Self-Signed Certificate Error](/blog/content/https-on-dev/self-signed-certificate-warning.png)

You can accept that inconvenience and manually ignore the warning each time it shows up. But it's very cumbersome, and it may block e2e tests in a CI environment. A better solution is to create your own local **certificate authority**, add this custom authority to your browser and generate a certificate from it.

That's what `mkcert` does for you under the hood, but if you want to do it yourself, I wrote a gist that may help you: [Kmaschta/205a67e42421e779edd3530a0efe5945](https://gist.github.com/Kmaschta/205a67e42421e779edd3530a0efe5945).

## HTTPS From a Reverse Proxy Or A Third-Party App

Usually, end-users don't directly reach the application server. Instead, user requests are handled by a load balancer or a reverse proxy that distributes requests across backends, stores the cache, protects from unwanted requests, and so on. It's not uncommon to see these proxies take the role of decrypting requests and encrypting responses as well.

On a development environment, we can use a reverse proxy, too!

### Encryption via Traefik and Docker Compose

[Traefik](https://traefik.io/) is a reverse proxy that comes with a lot of advantages for developers. Among others, it's simple to configure and it comes with a GUI. Also, there is an official docker image [available on docker hub](https://hub.docker.com/_/traefik).

So, let's use it inside the `docker-compose.yml` of a hypothetical application that only serves static files:

```yaml
version: '3.4'

services:
    reverse-proxy:
        image: traefik # The official Traefik docker image
        command: --docker --api # Enables the web UI and tells Traefik to listen to docker
        ports:
            - '3000:443'  # Proxy entrypoint
            - '8000:8080' # Dashboard
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
            - ./certs/server.crt:/sslcerts/server.crt
            - ./certs/server.key:/sslcerts/server.key
            - ./traefik.toml:/traefik.toml # Traefik configuration file (see below)
        labels:
            - 'traefik.enable=false'
        depends_on:
            - static-files
    static-files:
        image: halverneus/static-file-server
        volumes:
            - ./static:/web
        labels:
            - 'traefik.enable=true'
            - 'traefik.frontend.rule=Host:localhost'
            - 'traefik.port=8080'
            - 'traefik.protocol=http'
        ports:
            - 8080:8080
```

In this example, our static file server listens on port 8080 and serves files in HTTP. This configuration tells Traefik to handle HTTPS requests to `https://localhost` and proxy each of them to `http://localhost:8080` in order to serve static files.

We also have to add a `traefik.toml` to configure the Traefik entry points:

```ini
debug = false

logLevel = "ERROR"
defaultEntryPoints = ["https","http"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
  [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "/sslcerts/server.crt"
      keyFile = "/sslcerts/server.key"

```

Here, we have two entry points: `http` and `https`, listening respectively to ports 80 and 443. The first one redirects to the HTTPS, and the second is configured to encrypt requests thanks to the specified TLS certificates.

![Traefik Dashboard](/blog/content/https-on-dev/traefik-dashboard.png)

### Encryption From Docker Compose via Nginx

Obviously, we can do exactly the same with the popular Nginx reverse proxy. As Nginx can also directly serve static files itself, the setup is simpler. Again, the first step is the `docker-compose.yml`:

```yaml
version: '3'

services:
    web:
        image: nginx:alpine
        volumes:
            - ./static:/var/www
            - ./default.conf:/etc/nginx/conf.d/default.conf
            - ../../certs/server.crt:/etc/nginx/conf.d/server.crt
            - ../../certs/server.key:/etc/nginx/conf.d/server.key
        ports:
            - "3000:443"
```

And the nginx configuration at `default.conf`:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;

    server_name ~.;

    ssl_certificate /etc/nginx/conf.d/server.crt;
    ssl_certificate_key /etc/nginx/conf.d/server.key;

    location / {
        root /var/www;
    }

    ## If the static server was another docker service,
    ## It is possible to forward requests to its port:
    # location / {
    #     proxy_set_header Host $host;
    #     proxy_set_header X-Real-IP $remote_addr;
    #     proxy_pass http://web:3000/;
    # }
}
```

## Serving HTTPS Directly From The Application

Sometimes security requirements demand end-to-end encryption, or having a reverse proxy just might seem to be overkill on a development environment. Most of the time, it's possible to serve HTTPS directly from your everyday development environment.

Let's take the example of a common stack: a React application with a REST API using [express](http://expressjs.com/).

### Using Create React App or Webpack Dev Server

Your average React app is bootstraped by `create-react-app`. This awesome tool comes with a lot of built-in features and can handle HTTPS out of the box. To do so, you just have to specify a `HTTPS=true` environment variable when starting the app:

```bash
HTTPS=true npm run start
# or
HTTPS=true yarn start
```

This command will serve your app from `https://localhost:3000` instead of `http://localhost:3000` with an auto-generated certificate. But it's a self-signed certificate, so the developer experience is poor. 

If you want to use your own HTTPS certificate (signed with an authority that your browser trusts), `create-react-app` doesn't let you configure it without ejecting the app (`npm run eject`).

Fortunately, if you do eject CRA, or if your project is bundled with webpack, `webpack-dev-server` is as straightforward as `create-react-app` when it comes to serve HTTPS! It's possible to configure a custom HTTPS certificate with two lines in the Webpack configuration:

```js
const fs = require('fs');
const path = require('path');

module.exports = {
    mode: 'production',
    // ...
    devServer: {
        https: {
            key: fs.readFileSync(path.resolve(__dirname, '../../certs/server.key')),
            cert: fs.readFileSync(path.resolve(__dirname, '../../certs/server.crt')),
        },
        port: 3000,
    },
};
```

The next time you'll run `webpack-dev-server`, it will handle HTTPS requests to `https://localhost:3000`.

![Example App - Static Site](/blog/content/https-on-dev/example-app.png)

### Encrypted HTTP/2 With Express And SPDY

Now that we have our frontend part of the app that is served through HTTPS, we have to do the same with our backend.

For this purpose, let's use [express](https://www.npmjs.com/package/express) and [spdy](https://www.npmjs.com/package/spdy). No wonder why these two libraries names are about SPEED, it's because they are fast to setup!

```js
const fs = require('fs');
const path = require('path');
const express = require('express');
const spdy = require('spdy');

const CERTS_ROOT = '../../certs/';

const app = express();

app.use(express.static('static'));

const config = {
    cert: fs.readFileSync(path.resolve(CERTS_ROOT, 'server.crt')),
    key: fs.readFileSync(path.resolve(CERTS_ROOT, 'server.key')),
};

spdy.createServer(config, app).listen(3000, (err) => {
    if (err) {
        console.error('An error occured', error);
        return;
    }

    console.log('Server listening on https://localhost:3000.')
});
```

HTTP/2 isn't required to serve HTTPS, [it's possible to serve encrypted content with HTTP first of the name](https://stackoverflow.com/a/11745114/3868326), but while we are at serving HTTPS, we can upgrade the HTTP protocol. If you want to know more about the advantages of HTTP/2, you can read [this quick FAQ](https://http2.github.io/faq/).

## Conclusion

Modern tooling allows to build applications that are safer and faster for end-users and, now, easy to bootstrap. I hope that I convinced you to use these libraries and technologies starting from your project inception, when they are still cheap to install.

All the examples I used in this blog post are gathered on the following repo: [marmelab/https-on-dev](https://github.com/marmelab/https-on-dev). Feel free to play with and add your own HTTPS development experience!
