---
layout: post
author: Kévin Maschtaler
title: "GitHub Oauth authentication for SPA without server"
date: 2017-03-04 12:00:00
lang: en
tag:
- github
- oauth
- SPA
- serverless
---
For a Single-Page Application (SPA), you often need an authentication from a trusted service like Google, Facebook, Twitter, Github and so on.
Most of the time, you have a server with an API to handle this feature. But what if it's not the case?

While creating a simple page for [Sedy](https://marmelab.com/sedy/), I was in this position: I just wanted that users can log from GitHub and have a button to install Sedy on their repositories. But I didn't want a server just for this.

Let's see how we can implement a GitHub Oauth authentication with another approach.
<!--more-->

## How the GitHub Oauth implementation works
The [Oauth2 specification](https://oauth.net/2/) offers multiple authorization methods, but GitHub doesn't supports the `Implicit Grant` which works well with client-only applications such as a Single-Page Application.

The only method supported by GitHub is the `Application Code Grant`:

<center style="padding-bottom: 1rem;">
	<a href="http://www.bubblecode.net/fr/2016/01/22/comprendre-oauth2/">
		<img alt="Application Code Grant Flow" src="/blog/content/authorization_code_grant.png" />
	</a>
</center>

As you can see, the **Exchange Code for Access Token** step requires to send your `client secret`. The problem is that you can't publish any secret token in a SPA since the code is accessible to every user. Please don't do that.

Instead, you need to have another server that works like a proxy, receive your request, ask GitHub to transform your code into an access token and give to the SPA. We can represent that behavior into a simple function:

```js
function transformCodeToAccessToken(code) {
    return askGitHub(code, clientId, clientSecret, redirectUri);
}
```

For a while, I was using a third-party service to this job: [hello.js](https://adodson.com/hello.js/), but it comes with some big issues:

- You need to trust this service by give it your GitHub secret token
- It's a critical dependency of your project. What if the hello.js server is down?
- It doesn't work well with HTTPS

But hey, can we make this function work in an AWS Lambda? Of course, we can.


## Let's code that function
First, what do we need?

- Retrieve the client request
- Send this request to GitHub API with the secret token
- Secure your function by verifying the request origin

In order to deploy the code into an Lambda Function, you just need write a JavaScript file which exports a `handler` function.

{% raw %}
```js
// index.js
const request = require('request');

const config = {
    clientId: 'GITHUB CLIENT ID',
    clientSecret: 'GITHUB CLIENT SECRET',
    redirectUri: 'GITHUB REDIRECT URI',
    allowedOrigins: ['http://your.website.spa', 'https://your.website.spa'],
};

const handler = function (event, context, callback) {
    // Retrieve the request, more details about the event variable later
    const headers = event.headers;
    const body = event.body;
    const origin = headers.origin || headers.Origin;

    // Check for malicious request
    if (!config.allowedOrigins.includes(origin)) {
        throw new Error(`${headers.origin} is not an allowed origin.`);
    }

    const url = 'https://github.com/login/oauth/access_token';
    const options = {
        headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
        },
        body: JSON.stringify({
            code: body.code,
            client_id: config.clientId,
            client_secret: config.clientSecret,
            redirect_uri: config.redirectUri,
        }),
    };

    // Request to GitHub with the given code
    request(url, options, function(err, response) {
        if (err) {
            callback({ success: false, error: err });
            return;
        }

        callback(null, {
            success: true,
            // Access token should be stored in response.body
            body: JSON.parse(response.body),
        });
    });
};

module.exports = { handler: handler };
```
{% endraw %}

## Deploy the function and expose it to the web
Even if the so hyped [Serverless Framework](https://github.com/serverless/serverless) allows you to deploy your function on all the providers you want, it comes with a overwhelming complexity. I choose AWS Lambda because I have an AWS account, but you can do the same with Google CloudFunctions or Azure Functions.

So, log into to your AWS account and create a new Lambda Function for Node 4.3, configure `index.handler` as the handler and copy-paste your code in the `Code` tab.

I don't explain the whole process here but when it comes to chose a trigger, you have to create an **API Gateway** in order to expose an endpoint which will trigger your Lambda Function.

There are two tricky points with your API Gateway that you need to address:

- Enable CORS headers in order to allow browsers to request your new endpoint
- Bind the request headers to the `event` argument

For the first point, I let you check [the documentation about API Gateway and CORS](https://github.com/serverless/serverless).

About the `event` argument, in the case that the trigger is an API Gateway, it's the representation of the HTTP request sent to the endpoint.
By default, the representation only contains the request body, but we need to have both the body and the HTTP headers.

To do so, go to the API Gateway menu, click on the new API and select the HTTP verb, in my case it's `ANY`.
You should have different panels shown here. In our case, we need to update the **Integration Request**.

![Sedy API Integration](/blog/content/sedy_api_gateway.jpg)

What we want to change here is a new **Body Mapping Template** for all `application/json` requests. Here is the template that I use:

```json
{
  "body" : $input.json('$'),
  "headers": {
    #foreach($header in $input.params().header.keySet())
    "$header": "$util.escapeJavaScript($input.params().header.get($header))" #if($foreach.hasNext),#end

    #end
  }
}
```

Hence, we have a `body` and a `headers` variables that contains all the informations we retrieve as the first argument of our `handler` function. Don't forget to save your changes and deploy the new API configuration.

Here we are! You can now use your new endpoint to transform your GitHub code into an access token from your Single-Page App.

```
POST https://[ID].execute-api.[REGION].amazonaws.com/prod/[FUNCTION]
Accept: application/json
Content-Type: application/json

{
    "code": "GITHUB_CODE"
}
```

If you want more details about the usage or the integration of such a function, you can take a look on [the Sedy repository](https://github.com/marmelab/sedy).
