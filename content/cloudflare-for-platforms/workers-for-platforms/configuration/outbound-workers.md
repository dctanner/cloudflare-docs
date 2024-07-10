---
pcx_content_type: concept
title: Outbound Workers
---

# Outbound Workers

Outbound Workers sit between your customer’s Workers and the public Internet. They give you visibility into all outgoing `fetch()` requests from user Workers.

![Outbound Workers diagram information](/images/cloudflare-for-platforms/outbound-worker-diagram.png)

## General Use Cases

Outbound Workers can be used to:

- Log all subrequests to identify malicious domains or usage patterns.
- Create, allow, or block lists for hostnames requested by user Workers.
- Configure authentication to your APIs behind the scenes (without end developers needing to set credentials).

## Use Outbound Workers

To use Outbound Workers:

Make sure that you have `wrangler@3.3.0` or later [installed](/workers/wrangler/install-and-update/).

1. Create a Worker intended to serve as your Outbound Worker: In this example, the Outbound Worker is called `my-outbound-worker`.

```sh
$ npm create cloudflare@latest my-outbound-worker
```

2. The Outbound Worker will be invoked on any `fetch()` requests from a User Worker. The User Worker will trigger a [FetchEvent](/workers/runtime-apis/handlers/fetch/) on the Outbound Worker.

Edit the src/index.js file of the Outbound Worker. The following is an example of an Outbound Worker that logs the url of any fetch requests made from the User Worker.

```js
---
filename: index.js
---
export default {
  // this event is fired when a User Worker makes a subrequest
  async fetch(request, env, ctx) {
    // log the request.url
    console.log('Outbound Worker handling request to: ' + request.url)

    // env contains optional key value pairs we can set in `dispatcher.get()` in the Dispatcher Worker code
    console.log('user_worker_name: ' + env.user_worker_name)
    console.log('user_worker_url: ' + env.user_worker_url)

    // Continue the original request from the User Worker
    return fetch(request)
  }
};
```

{{<Aside type ="note">}}

Outbound Workers do not intercept fetch requests made from [Durable Objects](/durable-objects/) or [mTLS certificate bindings](/workers/runtime-apis/bindings/mtls/).

{{</Aside>}}

3. Deploy the Outbound Worker as you would deploy a regular worker (no special config is required in its worker.toml):

```sh
$ npx wrangler deploy
```

4. In your Dispatch Worker's [wrangler.toml](/workers/wrangler/configuration/), add an `outbound` configuration line in the [dispatch namespaces](/cloudflare-for-platforms/workers-for-platforms/get-started/configuration/#2-create-a-dispatch-namespace) section.

Optionally, you can pass key value pairs from your Dispatch Worker to the Outbound Worker, the key names must be specified in an array in the **parameters** of the outbound worker configuration line.

```toml
---
filename: wrangler.toml
---
[[dispatch_namespaces]]
binding = "dispatcher"
namespace = "testing" # This is the dispatch namespace used in the [Configure Workers for Platforms](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/get-started/configuration/#2-create-a-dispatch-namespace) guide
outbound = {service = "my-outbound-worker", parameters = ["user_worker_name", "user_worker_url"]}
```

5. Edit your Dispatch Worker to call the Outbound Worker, by adding the outbound option when calling `dispatcher.get()`. You can specify the optional key value pair data to pass onto the Outbound Worker here.
```js
---
filename: index.js
---
export default {
  async fetch(request, env) {
    try {
      // parse the URL, read the subdomain
      let workerName = new URL(request.url).host.split(".")[0];

      let userWorker = env.dispatcher.get(
        workerName,
        {},
        {
          outbound: {
            user_worker_name: workerName,
            user_worker_url: request.url,
          },
        }
      );
      return await userWorker.fetch(request);
    } catch (e) {
      if (e.message.startsWith("Worker not found")) {
        // we tried to get a worker that doesn't exist in our dispatch namespace
        return new Response("", { status: 404 });
      }
      return new Response(e.message, { status: 500 });
    }
  },
};

```

Remember to deploy this updated Dispatch Worker.
