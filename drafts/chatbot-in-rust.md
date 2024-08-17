+++
title = "Building a Chatbot in Rust"
description = "Let's look at building a Chatbot in Rust backed by Qdrant and OpenAI."
template = "blog-page.html"
tags = ["rust"]

[extra]
repository = "joshua-mo-143/www"
+++

## Introduction

Today, we're going to look at building a customer support chatbot in Rust, backed by Qdrant. This article will be one of a series of articles detailing the building of complex web services (or "chatbots") to help you understand how you can create more complex systems that leverage LLMs to streamline and improve processes.

## Pre-requisites
This article assumes you have basic working knowledge of Rust, although a best-effort attempt will be made to explain most of the code and where things can go wrong for educational purposes.

You'll also want to make sure you have an instance of [Qdrant](https://qdrant.tech/) that you can use. The easiest way to try Qdrant if you are not acquainted with it is to use Docker to [spin up an instance locally](https://qdrant.tech/documentation/quickstart/) and try it out. In development, I would recommend running it with the `-d` flag so you don't have to spin up a container every time you're running the application. **Note that the Qdrant client in Rust attaches to the gRPC port (6334) - not to be confused with port 6333, which is the HTTP port!**

## Preparing data for RAG ingestion
Before we start, you'll need to prepare your data. If you don't already have a dataset, you can find one [on Kaggle](https://www.kaggle.com/datasets/aimack/customer-service-chat-data-30k-rows). 

Generally, most of the work for writing RAG-based applications can be found in data cleaning and preparing your data. Good results from LLMs primarily rely on high-quality, well-formatted data. 

Data cleaning is a large topic and your cleaning strategy generally depends on the quality of the data. Regardless of your strategy however, the result should be a document that can be semantically matched against from natural language queries. We can express a suitable result for what the data to be embedded looks like as a JSON object:

```json
{
	"question": "What is the status of my order?",
	"answer": "Your delivery should be two days away."
}
```

You can see this is a pretty simple embedding with not a lot of words. The idea is that by splitting conversations into many different (de-duplicated) questions and answers, the bot will have a much better understanding of the response style of a customer success agent as well as dealing with various issues. For example, if a customer puts their credit card number in the chat, we obviously want to advise against them doing that.

Some best practices to keep in mind:
- Labelling the different parts of your embeddings is quite beneficial as it gives semantic meaning to each part of a document.
- Removing special characters and things that do not necessarily add relevant semantic meaning to the document isimportant, as characters that aren't adding useful semantic meaning can actually detract from overall data quality.
- Make sure to remove duplicates! Duplicate embeddings can significantly hamper any attempts later on when you need to carry out data classification, retrieval or recommendations.
- Have a plan for monitoring application and LLM performance if you are planning to write a similar application and take it to production.

This is not an exhaustive list, but following the above points should help you move significantly closer towards making the most of LLMs.

## Setup

The next thing to do is actually initialising our project, which we'll do below (and switch directory into it):
```bash
cargo init my-chatbot
cd my-chatbot
```

Next, we'll initialise our dependencies. You can add everything with the following one-liner:
```bash
cargo add serde tokio async-openai qdrant-client \
-F serde/derive,tokio/macros,tokio/rt-multi-thread
```
This adds the following:
- `serde` (with the `derive` feature for easy de/serialization implementation)
- `tokio` for async runtime
- `async-openai` for OpenAI usage
- `qdrant-client` for using the Qdrant client in Rust
- `axum` for spinning up a web service easily

A simple `main.rs` should look like this:

```rust
// main.rs

#[tokio::main]
async fn main() {
    // router moved out of main function to allow testing later on
    let router = init_router();
	
    let tcpl = TcpListener::bind("0.0.0.0:8000").await.unwrap();
	
    axum::serve(tcpl, router).await.unwrap();
}

fn init_router() -> Router {
    Router::new()
}
```
## Finishing Up
Thanks for reading!
