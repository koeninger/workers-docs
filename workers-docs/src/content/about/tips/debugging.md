---
title: "Debugging Tips"
weight: 1
---

Debugging your Workers application, whether locally or in production, is a common request from Workers developers. Here's some tips on how to effectively debug your application, as well as some code samples to help you get up and running:

<iframe width="560" height="315" src="https://www.youtube.com/embed/8iPmy7ePYDE" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# Test your app on localhost using wrangler dev

When you're developing your Workers application, `wrangler dev` can significantly reduce the time it takes to testing and debugging new features.

To get started, run `wrangler dev` in your Workers project directory. `wrangler` will deploy your application to our preview service, and make it available for access on `localhost`:

```bash
$ wrangler dev

  Built successfully, built project size is 27 KiB.
  Using namespace for Workers Site "__app-workers_sites_assets_preview"
  Uploading site files
  Listening on http://localhost:8787

[2020-05-28 10:42:33] GET example.com/ HTTP/1.1 200 OK
[2020-05-28 10:42:35] GET example.com/static/nav-7cb303.png HTTP/1.1 200 OK
[2020-05-28 10:42:36] GET example.com/sw.js HTTP/1.1 200 OK
```

`wrangler dev` also supports `console.log` statements, so you can see output from your application in your local terminal:

```javascript
// index.js

addEventListener('fetch', event => {
  console.log(`Received new request: ${event.request.url}`)
  event.respondWith(handleEvent(event))
})
```

```bash
$ wrangler dev

[2020-05-28 10:42:33] GET example.com/ HTTP/1.1 200 OK
Received new request to url: https://example.com/
```

You can customize how `wrangler dev` works to fit your needs: see [the docs](/tooling/wrangler/commands/#dev-alpha-) for available configuration options.

# Tail production logs with wrangler tail

With your Workers application deployed, you may want to inspect incoming traffic. This can be useful for situations where a user is running into issues or errors that you can't reproduce. `wrangler tail` allows developers to "tail" their Workers application's logs, giving you real-time access to information about incoming requests.

To get started, run `wrangler tail` in your Workers project directory. This will log any incoming requests to your application available in your local terminal.

The output of each `wrangler tail` log is a structured JSON object:

```json
{
  "outcome": "ok",
  "scriptName": null,
  "exceptions": [],
  "logs": [],
  "eventTimestamp": 1590680082349,
  "event": {
    "request": {
      "url": "https://www.bytesized.xyz/",
      "method": "GET",
      "headers": {},
      "cf": {}
    }
  }
}
```

By piping the output to tools like [`jq`](https://stedolan.github.io/jq/), you can query and manipulate the requests to look for specific information:

```bash
$ wrangler tail | jq .event.request.url
"https://www.bytesized.xyz/"
"https://www.bytesized.xyz/component---src-pages-index-js-a77e385e3bde5b78dbf6.js"
"https://www.bytesized.xyz/page-data/app-data.json"
```

You can customize how `wrangler tail` works to fit your needs: see [the docs](/tooling/wrangler/commands/#tail) for available configuration options.

# Miscellaneous tips

## Error pages generated by Workers

When a Worker running in production has an error that prevents it from returning a response, the client will receive an error page with an error code, defined as follows:

| Error code | Meaning                                                                            |
| ---------- | ---------------------------------------------------------------------------------- |
| 1101       | Worker threw a JavaScript exception.                                               |
| 1102       | Worker exceeded CPU time limit. See: [Resource Limits](/about/limits)              |
| 1015       | Your client IP is being rate limited.                                              |
| 1027       | Worker exceeded free tier [daily request limit](/about/limits#Daily-Request-Limit) |

<br/>Other 11xx errors generally indicate a problem with the Workers runtime itself - please check our [status page](https://www.cloudflarestatus.com/) if you see one.

## Wrap your whole event handler in a try/catch

If your Workers application is returning an 1101 error code, your code is throwing an exception. You can catch the exception in your code using a `try`/`catch` block:

```javascript
async function handleRequest(request) {
  try {
    // ... some logic that might error ..
    return await fetch(newUrl, request)
  } catch (err) {
    // Return the error stack as the response
    return new Response(err.stack || err)
  }
}
```

## Setup a logging service

A Worker can make HTTP requests to any site on the public internet. You can use a service like [Sentry](https://sentry.io/) to collect error logs from your Worker, by making an HTTP request to the service to report the error. Refer to your service's API documentation for details on what kind of request to make.

When logging using this strategy, remember that outstanding asynchronous tasks are canceled as soon as a Worker finishes sending its main response body to the client. To ensure that a logging subrequest completes, you can pass the request promise to [`event.waitUntil()`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil). For example:

```javascript
   ...
  // Without event.waitUntil(), our fetch() to our logging service may
  // or may not complete.
  event.waitUntil(postLog(stack))
  return fetch(request)
}

function postLog(data) {
  return fetch('https://log-service.example.com/', {
    method: 'POST',
    body: data,
  })
}
```

## Go to Origin on Error

By using `event.passThroughOnException`, your Workers application will pass requests to your origin if it throws an exception. This allows you to add logging, tracking, or other features with Workers, without degrading your website's functionality.

```javascript
addEventListener('fetch', event => {
  event.passThroughOnException()

  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // An error here will return the origin response, as if the Worker wasn’t present.
  ...
  return fetch(request)
}
```
