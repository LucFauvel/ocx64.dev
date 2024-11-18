---
title: 'Using Rust inside a Slack Workflow app with WASM'
description: "Slack workflows are pretty cool, they use deno and I'm going to show you how you can use Rust!"
pubDate: 'Nov 17 2024'
heroImage: '/rust-slack-wasm.png'
---

### If you're wondering why you'd want to use WASM inside a slack workflow, well there's lots of reasons! 
In my case I wanted to do some ✨ encryption ✨ within the workflow to encrypt data before storing it and I didn't want to make an API just for that so WASM it was!

### Chapter 1: wasm-bindgen

I'll skim through this part because there like 1000 tutorials on how to setup a wasm-bindgen in a rust project so for the sake of this blog post I'll keep it simple.

Whats important here is how you package your Rust project and for that we'll be using wasm-pack.

For the sake of example here is a function in your Rust project.

```rust
#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```
