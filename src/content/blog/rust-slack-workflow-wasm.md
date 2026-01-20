---
title: 'Using Rust inside a Slack Workflow app with WASM'
description: "Slack workflows are pretty cool, they use deno and I'm going to show you how you can use Rust!"
pubDate: 'Nov 17 2024'
heroImage: '/rust-slack-wasm.jpg'
---

### If you're wondering why you'd want to use WASM inside a slack workflow, well there's lots of reasons! 
In my case I wanted to do some ✨ ChaCha20-Poly1305 encryption ✨ within the workflow to encrypt data before storing it and I didn't want to see the data before encrypting it so WASM it was!

### Chapter 1: Generating the Slack workflow app

There is an important distinction with slack apps. There are currently 2 ways to make an app, the new type called Workflow app, which are hosted in slack's infrastructure for free, and a classic app which uses webhooks.

In our case, we're not interested in using a classic app as that uses webhooks and we want to use a slack hosted app and have it run as close to the slack client as possible. With workflow apps, slack will run your javascript/typescript code in the [Deno runtime](https://deno.com/).

One downside of this approach is that we cannot publish the app on the slack marketplace as that is not currently supported.

The best way to create a slack workflow app is using the slack CLI. It's very well made (kudos to the team!)

In my case, I just used the deno starter template and deleted whatever I didn't need and renamed everything else.

```sh
slack create your-app-name --template https://github.com/slack-samples/deno-starter-template
```

### Chapter 2: wasm-bindgen

I'll skim through this part because there's like [1000 tutorials on how to setup wasm-bindgen in a rust project](https://rustwasm.github.io/book/game-of-life/setup.html), so for the sake of this blog post I'll keep it simple. Though I do strongly recommend [wasm-pack](https://rustwasm.github.io/wasm-pack/book/).

You can create a wasm library with the following:

```sh
wasm-pack new my-wasm-lib
```

Lets say you have this function inside your lib.rs file.

```rust
#[wasm_bindgen]
pub fn greet(name: Option<String>) -> String {
    format!("Hello from Rust, {}!", name.unwrap_or_default())
}
```

In order to use this function within the slack workflow infrastructure, you'll need to use the `web` target (I haven't tried with the `deno` target, let me know if it works!):

```sh
wasm-pack build --target web
```

Once your project is packed in web mode you should be able to load it as a dependency in your import_map.json. In order to do so you can either import it as a file or an NPM package. Fair warning, the file method won't work if you're trying to deploy the app as it needs a way to get to that file. The simplest is to use an npm package like so:

```json
{
  "imports": {
    "deno-slack-sdk/": "https://deno.land/x/deno_slack_sdk@2.14.2/",
    "deno-slack-api/": "https://deno.land/x/deno_slack_api@2.8.0/",
    "std/": "https://deno.land/std@0.224.0/",
    "mock-fetch/": "https://deno.land/x/mock_fetch@0.3.0/",
    "wasm-lib": "npm:slack-wasm-example@1.0.0/slack_wasm_example.js"
  }
}
```
You might have noticed that I've added a `slack_wasm_example.js` file in the path. One restriction I found while working on this is that Deno will look for an index.js file inside your npm library. In my project that file didn't exist so I had to manually specify the file. Deno seemed happy with that.

Once your dependency is loaded, you should have your types ready and usable.

### Chapter 3: How do workflow apps?

Workflow apps are well documented in the [Slack API docs](https://api.slack.com/automation/create) but to save you trouble of going to an LLM and asking it to summarize, here are the main points.

They have 4 different components: Workflow, Function, Trigger and Datastore. We will be skipping datastore for now as we don't really need it for this blog.

The workflow component is the sum of everything. In the generated project there should already be a workflow definition inside of `sample_workflow.ts`. It should look like this:

```ts
const SampleWorkflow = DefineWorkflow({
  callback_id: "sample_workflow",
  title: "Sample workflow",
  description: "A sample workflow",
  input_parameters: {
    properties: {
      interactivity: {
        type: Schema.slack.types.interactivity,
      },
      channel: {
        type: Schema.slack.types.channel_id,
      },
      user: {
        type: Schema.slack.types.user_id,
      },
    },
    required: ["interactivity", "channel", "user"],
  },
});
```

In that file you will see that the steps are defined below. You can open a form and use a slew of functions provided by slack. You will notice that a custom function defined in `sample_function.ts` is called here.

```ts
const sampleFunctionStep = SampleWorkflow.addStep(SampleFunctionDefinition, {
  message: inputForm.outputs.fields.message,
  user: SampleWorkflow.inputs.user,
});
```

This leads us to the meat of things, functions, they are where you define your business logic and in our case, load up our WASM module. We'll tweak the default `sample_function.ts` with our own special sauce.

```ts
export default SlackFunction(
  SampleFunctionDefinition,
  async ({ inputs, client }) => {
    await init("https://url/to/your/wasm-file.wasm");
    const userProfile = await client.users.profile.get({ user: inputs.user });
    return { outputs: { updatedMsg: greet(userProfile?.profile?.display_name) } };
  },
);
```

So here, you might notice the `init` call, this function is imported from our npm package like this: 

```ts
import init, { greet } from "wasm-lib";
```

This is how we are going to load the WASM file into the Deno runtime. Be warned, this will run every time the function is called with no caching, so I urge you to use a CDN of some sorts to upload your wasm file and prevent any rate-limiting.

Now that we have set up our function to use the WASM module, all that's left is calling it. This is where the last component comes in, triggers. There are multiple trigger types each with their role. The shortcut type is to be able to invoke the workflow from a link or slash command. The event type is where we can hook on message sent events and things of the sort, the scheduled trigger is like a cron job and the webhook trigger is a way of triggering a workflow from an external service with a webhook provided by slack.

In our case a shortcut trigger will work just fine. You can reference any workflow using a trigger, so lets put the ID of our own workflow in the trigger, like this:

```ts
const sampleTrigger: Trigger<typeof SampleWorkflow.definition> = {
  type: TriggerTypes.Shortcut,
  name: "Sample trigger",
  description: "A sample trigger",
  workflow: `#/workflows/${SampleWorkflow.definition.callback_id}`,
  inputs: {
    interactivity: {
      value: TriggerContextData.Shortcut.interactivity,
    },
    channel: {
      value: TriggerContextData.Shortcut.channel_id,
    },
    user: {
      value: TriggerContextData.Shortcut.user_id,
    },
  },
};
```

### Chapter 4: Deploy

Now that we have everything setup, last step is to check out our manifest file and deploy the app. The manifest file contains the references to the different components of the workflow app and additional data like the name and icon of the app.

If you look at your sample app, you should see a `manifest.ts` file like this:

```ts
export default Manifest({
  name: "slack-wasm-example",
  description: "A template for building Slack apps with Deno",
  icon: "assets/default_new_app_icon.png",
  workflows: [SampleWorkflow],
  outgoingDomains: [],
  datastores: [SampleObjectDatastore],
  botScopes: [
    "commands",
    "chat:write",
    "chat:write.public",
    "datastore:read",
    "datastore:write",
  ],
});
```

We're going to change it to look like this:

```ts
export default Manifest({
  name: "slack-wasm-example",
  description: "A template for building Slack apps with Deno",
  icon: "assets/default_new_app_icon.png",
  workflows: [SampleWorkflow],
  outgoingDomains: ["domain-of-your-wasm-file.com"],
  botScopes: [
    "commands",
    "chat:write",
    "chat:write.public",
    "users.profile:read"
  ],
});
```

I've added the `users.profile:read` scope for the user profile call in our function and I've added the url to our CDN in `outgoingDomains`, if the domain of your WASM file is not present, it will fail.

All that's left now is to run it with: 

```sh
slack run
```

This will make you go through a bunch of prompts and guide you through installing the local app in your workspace. Once that's done, it should print a URL to your workflow trigger that you can then use to call your workflow from slack.

test it out using that URL and if everything works, you can use this:

```sh
slack deploy
```

To deploy your app to slack's infrastructure and voilà, Rust in your slack!

Everything I've detailed here is available in this repo: [https://github.com/LucFauvel/slack-wasm-example](https://github.com/LucFauvel/slack-wasm-example)

Thanks for making it this far in my post! Damn you must've really needed help with this huh, well I don't have anything else to add, sorry. Feel free to hit me up on my socials if there's anything!
