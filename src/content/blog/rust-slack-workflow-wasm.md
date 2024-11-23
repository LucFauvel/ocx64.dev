---
title: 'Using Rust inside a Slack Workflow app with WASM'
description: "Slack workflows are pretty cool, they use deno and I'm going to show you how you can use Rust!"
pubDate: 'Nov 17 2024'
heroImage: '/rust-slack-wasm.png'
---

### If you're wondering why you'd want to use WASM inside a slack workflow, well there's lots of reasons! 
In my case I wanted to do some ✨ encryption ✨ within the workflow to encrypt data before storing it and I didn't want to make an API just for that so WASM it was!

### Chapter 1: Generating the Slack workflow app

There is an important destinction with slack apps. There are currently 2 ways to make an app, the new type called Workflow app, which are hosted in slack's infrastructure for free, and a classic app which uses webhooks.

In our case, we're not interested in using a classic app as that uses webhooks and we want to use a slack hosted app and have it run as close to the slack client as possible. With workflow apps, slack will run your javascript/typescript code in the [Deno runtime](https://deno.com/).

One downside of this approach is that we cannot publish the app on the slack marketplace as that is not supported now (author note, look for refrences if they are planning to release this feature)

The best way to create a slack workflow is using the slack CLI. Its very well made (kudos to the team!)

In my case, I just used the deno starter template and deleted what ever I didn't need/renamed everything.

```sh
slack create your-app-name --template https://github.com/slack-samples/deno-starter-template
```

### Chapter 2: wasm-bindgen

I'll skim through this part because there like [1000 tutorials on how to setup wasm-bindgen in a rust project](https://rustwasm.github.io/book/game-of-life/setup.html) so for the sake of this blog post I'll keep it simple. Though I do strongly recommend [wasm-pack](https://rustwasm.github.io/wasm-pack/book/).

You can create a wasn library with the following:

```sh
wasm-pack new my-wasm-lib
```

Lets say you have this function inside your lib.rs file.

```rust
#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```

In order to use this function within Deno, you need to pack your Rust library with wasm-pack using the fitting Deno option (web can work too with a few work arounds):

```sh
wasm-pack build --target Deno
```

Once your project is packed in web mode you should be able to load it as a dependency in your import_map.json. In order to do so you can either import it as a file or an NPM package. The simplest is to use an npm package like so:

```json
{
  "imports": {
    "deno-slack-sdk/": "https://deno.land/x/deno_slack_sdk@2.14.2/",
    "deno-slack-api/": "https://deno.land/x/deno_slack_api@2.8.0/",
    "std/": "https://deno.land/std@0.224.0/",
    "mock-fetch/": "https://deno.land/x/mock_fetch@0.3.0/",
    "wasm-library": "npm:my-wasm-lib@0.1.0/my-wasm-lib.js"
  }
}
```
You might have noticed that I added an `my-wasm-lib.js` file in the path. One restriction I found while working on this is that Deno will look for an index.js file inside your npm library. In my project that file didn't exist so I had to manually specify the file. Deno seemed happy with that.

Once your dependency is loaded, you should have your types ready

### Chapter 3: How do workflow apps?

Workflow apps are well documented in the [Slack API docs](https://slack) but to save you trouble of going to an LLM and asking it to summarize, here are the main points.

They have 4 different components: Workflow, Function, Trigger and Datastore. We will be skipping datastore for now as we don't really need it for this blog.

The workflow component is the sum of everything. In the generated project there should already be a workflow setup inside of sample_workflow.ts. It should look like this:

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

In that file you will see that the steps are defined below. You can open a form and use a slew of functions provided by slack. You will notice that a custom function defined in sample_function.ts is called here.

```ts
const sampleFunctionStep = SampleWorkflow.addStep(SampleFunctionDefinition, {
  message: inputForm.outputs.fields.message,
  user: SampleWorkflow.inputs.user,
});
```

This leads us to the meat of things, functions, they are where you define your business logic and in our case, load up our WASM module.

```ts
FIGURE OUT IF WE NEED THE INIT LINE IN DENO MODE
```

Now that we have setup our function to use the WASM module. All that's left is calling it. This is where the last component comes in, triggers. There are multiple trigger types each with their role. The shortcut type is to be able to invoke the workflow from a link or slash command. The event type where we can hook on message sent events and things of the sort. LUC NOTE: Add more details for the types of triggers.

### Chapter 4: Deploy

Now that we have everything setup, last step is to check out manifest file and deploy the app. The manifest file contains the refrences to the different components of the workflow app and additional data like the name and icon of the app. This is also where we're going to define the trigger of the workflpw app.

If you look at your sample app, you should see a manifest.ts file like this:

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
