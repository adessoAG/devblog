---
layout: 		[post, post-xml]     
title:  		"A Series of Rust: Exploring Rust for AWS Lambda - Part 1"
date:   		2021-05-17 16:15
modified_date: 	2021-05-17 16:15
author: 		felix_marezki
categories: 	[Softwareentwicklung]
tags: 			[AWS, AWS Lambda, Cloud, Rust, A Series of Rust]
---

The demand for a high performance, yet secure programming language is increasing.
Evidence for that claim is found not only in statistics about security bugs but also in the ground and momentum in general that Rust has been able to gain, lately. 
I take that as well as a bit of my own curiosity as cause for a series of blog posts about Rust and applications in real world scenarios.

# The Intro

With all the arguments around Rust considering safety, usability and performance already being established, I would like to take the opportunity here to show the current state of effort around Rust that has or is being made to make Rust a suitable candidate for cloud development.
Since Amazon seems to have taken Rusts proposal very seriously and are pursuing Rust already in a variety of their own services as well as offerings to their cloud customers, projects like [lambda-runtime](https://github.com/awslabs/aws-lambda-rust-runtime) or [aws-rust-sdk](https://github.com/awslabs/aws-sdk-rust) have caught my attention.

This Post is therefore the first of what I plan to be three of this kind.

# The Contents

In this post we will develop a function for AWS lambda using Rust that we will be able to upload and test on AWS.

I presume you are somewhat familiar with one AWS service or the other but just in case you have not come across lambda or the AWS CLI yet you can find links for further information on both these topics below:

* [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

# The Groundwork

This is what I'm gonna be using:

- [Rust (Up)](https://www.rust-lang.org/learn/get-started)
- [Visual Studio Code](https://code.visualstudio.com)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

Just follow the links for installation instructions.

The musl target is required for our rust installation:

```bash
rustup target add x86_64-unknown-linux-musl
```

We need one additional thing in case we are trying to run this on OS X. Which is the musl library to statically link to:

```bash
brew install filosottile/musl-cross/musl-cross
```

```bash
mkdir .cargo
echo $'[target.x86_64-unknown-linux-musl]\nlinker = "x86_64-linux-musl-gcc"' > .cargo/config
```

For a quick explanation: Musl is a libc implementation that is optimized for static linking. This library together with the rust target enables us to produce a binary that needs no further attention from the Operating System to link dependencies at runtime. You can find further information about musl [here](https://musl.libc.org).

#  The Account

To try this we need an AWS Account and configure it as described below.

You have to set up a user in your `AWS Management Console` that has permissions to at least create new lambda functions. 
Also, define it to be used programmatically and generate an `access key` for it. 
For the permissions part I chose to attach an `AWS Managed Policy` to to my user, which you can find by the name `AWSLambda_FullAccess`.



# The Project

To start with our project we can use cargo to generate an initial crate:

```bash
cargo new adesso-echo
```

Now, we can open the folder that cargo created in Visual Studio Code:

```bash
code adesso-echo
```

# The Code

There are already several projects in place that ease some of the typical things we would like to do with rust in an AWS context. 
We are going to leverage the lambda_runtime crate to easily integrate our code into the lambda infrastructure.
So lets add a copuple of dependencies.
Our `Cargo.toml` should now look something like this:

```toml
[package]
name = "adesso-echo"
version = "0.1.0"
authors = ["felix.marezki"]
edition = "2018"

[dependencies]
lambda_runtime = "0.3.0"
serde_json = "1.0"
tokio = { version = "1.5.0", features = ["full"] }
```

In our `main.rs` we are going to do the following:

```rust
use lambda_runtime::{handler_fn, Context, Error};
use serde_json::{Value, json};

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler_fn(func);
    lambda_runtime::run(func).await?;
    Ok(())
}

async fn func(event: Value, _: Context) -> Result<Value, Error> {
    Ok(json!({ 
        "message": format!("Hello, {}!", event["name"].as_str().unwrap_or("world"))
    }))
}
```

# The Build

We can now build and create our function, assuming you have your aws CLI configured locally with correct permissions and everything.

Lets build our application:

```bash
cargo build --release --target x86_64-unknown-linux-musl
```

Now, AWS Lambda runs an executable named `bootstrap` that resides within a zip file that we will supply to it.

We are going to execute these three commands to generate our zip file:

```bash
cp ./target/x86_64-unknown-linux-musl/release/adesso-echo ./bootstrap
```

```bash
zip lambda.zip ./bootstrap
```

```bash
rm ./bootstrap
```

# The Deployment

To get our function going on `AWS` we need to do one last thing:

```bash
aws lambda create-function --function-name adesso-echo --handler doesnt.matter --zip-file fileb://lambda.zip --runtime provided --role arn:aws:iam::XXXXXXXXXXXX:role/lambda-role --environment Variables={RUST_BACKTRACE=1}
```
* [--function-name] denotes the name of the function we wish to create \
* [--handler] the handler is usually a reference to the corresponding handler function. In our case we provide the runtime (and poll on the API our selfs as you will see in the next blog post) in our executable so this wouldn't matter so we set it to `doesnt.matter` \
* [--zip-file] the zip file to use \
* [--role] the role our function should assume for execution \
* [--environment] sets the environment attribute on you lambda resource

This will upload our function to `AWS`.

# The Test

We can now test our function using the `AWS CLI`. To do so run the following command:

```bash
aws lambda invoke --function-name adesso-echo --payload '{"name": "Felix"}' --cli-binary-format raw-in-base64-out out.txt
```

and

```bash
cat out.txt
```
Output: {"message": "Hello, Felix!"}

# The End

So we have seen how to build and deploy lambda functions written in rust to AWS. 
In the next post I wanna see if we can test our experiments locally so we can spare us the roundtrip to AWS every time we want to try something new.
