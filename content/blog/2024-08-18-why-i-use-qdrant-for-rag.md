+++
title = "Why I use Qdrant for RAG"
description = "What makes Qdrant better than the competition? Let's break it down."
template = "blog-page.html"
tags = ["qdrant", "miscellaneous"]

[extra]
thumb = "why-i-use-qdrant-for-rag-thumb.png"
+++

## Introduction

In the short amount of time that I've used AWS Bedrock and other related AWS AI/ML tools for professional work, I've learned that AWS does not particularly excel in the vector search field. There's a few things that I have used with AWS for AI/ML purposes, with varying degrees of results. The common thread between all of the options is that while they're *good*, they're not great. For example: it's not obvious that a lot of the Bedrock models (outside of Claude) are pretty mediocre. OpenSearch, while technically being usable for RAG, is more of a general search engine and can fit other purposes like log analytics and full-text search (and therefore does not cater to the particularities of RAG systems).

Enter Qdrant, a [vector storage database that aims to excel at high-performance vector search at scale.](https://qdrant.tech/) Compared to other solutions (and of course the previously mentioned AWS offering), I found their offering to be pretty compelling:
- They [use Rust under the hood](https://github.com/qdrant/qdrant)
- They have a Docker image for trying Qdrant out locally! No account registration required to get going.
- They're open source and are active open source contributors (see: `fastembed` library)

I realise that playing the Rust card here, as someone who is already heavily invested in Rust, is not particularly imaginative. However, the fact remains that Rust is likely to continue to see future use in database programs. SurrealDB and TiKV (a part of the TiDB project by PingCap) both use Rust under the hood, with Databend and ScyllaDB using Rust in some part. It should be mentioned of course, that Qdrant has database client libraries in languages other than Rust (notably Python).

## What's so good about Qdrant?

In my opinion, Qdrant makes vector search extremely easy (and is really good at it!). All you need to do is configure and create a collection (selecting things like the embedding dimension size and search method, as well as whether to store embeddings on disk or in memory). Then you insert embeddings alongside relevant JSON payloads and you're basically ready to go! If you have used Weaviate or Pinecone, it is mostly the same process.

You can also store multiple embeddings under a single embedding record so that you can search for all of them at the same time. For example, if you have two relevant snippets (like a code snippet and a JSON payload snippet that contains metadata about the code snippet) that go together and you want your application to search for cosine similarity for both of them, it is quite simple to just put them together in an array and then upsert the relevant embeddings.

Their free tier is also pretty generous and I have found that even after inserting quite a few embeddings, I'm still within the free tier bounds. As previously mentioned of course, you can also run Qdrant's Docker image should you want to try things locally. However, for web applications that I've created in the past, it works pretty well for the use cases I have. If I *were* to use Qdrant for a business use case though, I'd be happy to do so. Judging by [this Reddit thread](https://www.reddit.com/r/vectordatabase/comments/170j6zd/my_strategy_for_picking_a_vector_database_a/), Qdrant is the cheapest in terms of pricing and it's not close, except for the pricing at 20M vectors where it comes out a bit more expensive than Pinecone normally, but is much cheaper for high perf).

The Qdrant team are also quite active on their Discord server. Whether this is a benefit or not mostly depends on whether or not you use Discord, but if you have any questions you can always drop a mention and they tend to respond pretty quickly. They also hold some of their events there so if you want insider insight, that's the place to be.

## Why not Pinecone?

This answer boils down to a few things, currently:
- Pinecone requires account registration on the cloud before you can do anything with it (huge no-no in my opinion)
- Pinecone's Rust SDK is currently in alpha at the moment, and doesn't have full functionality. This is somewhat painful as it means I have to switch to Python or another language just to use Pinecone... not ideal.
- How am I supposed to do local dev testing if there is no Docker image for me to pull?

Pinecone being closed-source is neither here nor there, but having to register just to do anything is already a non-starter for me personally. However, the RBAC feature of Pinecone is something that Qdrant lacks and could maybe benefit from. Though if you're using Qdrant from AWS Marketplace, you probably don't need to worry about this.

## Final thoughts

Although vector databases are quite specialised, they're very useful for the right use case - and Qdrant is no exception. For engineers who primarily work with (or want to work with!) Rust, using Qdrant is mostly a no-brainer as it's super easy to get started. Interested? Check out [Qdrant](https://qdrant.tech/) and try building something cool with it!

You can also, of course, check out the Shuttle articles I've written on Qdrant if you want to follow a tutorial:
- [Build an agentic RAG workflow with Qdrant and Rust](https://www.shuttle.rs/blog/2024/05/23/building-agentic-rag-rust-qdrant)
- [Implement semantic caching with Qdrant and Rust](https://www.shuttle.rs/blog/2024/05/30/semantic-caching-qdrant-rust)
- [Building a RAG web service with Qdrant and Rust](https://www.shuttle.rs/blog/2024/02/28/rag-llm-rust)
