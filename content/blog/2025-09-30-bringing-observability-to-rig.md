+++
title = "Bringing Observability to Rig"
description = "Observability is necessary in production. Let's talk about how Rig implements it."
template = "blog-page.html"
tags = ["observability", "rig"]
+++
With the new Rig 0.21.0 release comes significantly upgraded providers in terms of observability. We have added compatibility for OpenTelemetry GenAI Semantic Conventions, as well as instrumenting the system such that you can either essentially use Rig's inbuilt spans for whatever you want, or alternatively making your own spans and instrumenting those on top of our pre-defined functions.

In this article, we'll go over what Rig is, why we decided to instrument the library directly, how we did it and what the challenges were, how we did it and our learnings. This article will also mostly speak from my own perspective on building out the integration.

## What is Rig?
[Rig](https://github.com/0xPlaygrounds/rig) is an AI framework in Rust that tries to make it as easy as possible to create functional, lightweight, modular agents. That is to say, we provide boilerplate for you to use LLMs in a loop that will automatically use tools for you. Don't worry, we know - there's no magic here.

One might say Rig is the "leading" framework. However, that's an extremely low bar given the state of the Rust AI/ML ecosystem - so let me raise that a bit for you:
- We've been mentioned in Solana dev docs and SurrealDB
- We're being noticed and actively contributed to by members of popular Python AI frameworks like Agno
- Companies are reaching out to us to do talks on Rig

While we're not quite at the level of AI frameworks in other languages yet, the hope is that one day we will be as big as them in our own right and we will compete on a truly level playing field (and that people will still want to use Rig in production by then!).
### The problem statement
Most Python AI frameworks do not actually have observability out of the gate. Typically because of this, a lot of observability providers use their own SDK to monkey patch the original SDK.

It is an interesting solution that perhaps fits the nature of the Python ecosystem quite well. However, in Rust this is not an option[^1]. You either build the library with observability in mind, or you simply just don't do it. There is no in-between on this.

Because AI (agent) frameworks typically have many layers of abstraction, this compounds the problem quite a lot and makes it extremely difficult to get any serious visibility into the framework as an end user. Additionally, being able to keep track of model metrics is quite important. Teams and companies will likely want to keep track of statistics like token usage, model latency, time to first token, and more - as model inference tends to become a very significant part of a team's spending budget.

Of course, we have also received GitHub issues about this exact issue. Without time to investigate time allocation for it though, we'd left it on the shelf for a while as the ticket was basically just "improve observability" without any real leads. After Swiftide implemented their Langfuse integration however, I was pretty motivated to get our observability integration going.

[1] Okay, I mean there's the [guerilla](https://github.com/mehcode/guerrilla) crate. Doesn't mean you should do it, though.

### What our options were
So, our options were primarily as follows:
- Do nothing (people were using Rig in production before this update, they'll probably be using it in production after the update)
- Tie our observability to a specific integration like Langfuse... which while not being a bad idea, only ever allows our users to effectively use Langfuse for proper observability.
- Use a generic standard like OTel GenAI SemConv, which while a bit early is still supported by many of the larger general observability providers like Grafana

Of course, there's only really one real answer here, and it's the third one. Being able to support as many providers as possible should be a goal that we want to achieve for the sake of being as flexible as possible and allowing an easy integration from one provider to another.

### How we implemented it
The implementation side of it was actually relatively simple, using `tracing`. However given that Rig supports approximately 20(!) model providers now, it's still tedious work.

My first attempt at this was to try and fiddle around with `#[tracing::instrument]` macros. However, early attempts showed that this was not going to be the final iteration for a couple of reasons:
- What happens when users want to add extra fields? They won't be able to add any.
- I wanted to be able to use tracing spans with streams... which this particular macro didn't seem super compatible for.

In essence, what we did was conditionally spawning a pre-defined span if there is no previous span (but if the overriding span exists, just use that instead):

```rust
let span = if tracing::Span::current().is_disabled() {
    info_span!(
        target: "rig::completions",
        "chat",
        gen_ai.operation.name = "chat",
        gen_ai.provider.name = "huggingface",
        gen_ai.request.model = self.model,
        gen_ai.system_instructions = &completion_request.preamble,
        gen_ai.response.id = tracing::field::Empty,
        gen_ai.response.model = tracing::field::Empty,
        gen_ai.usage.output_tokens = tracing::field::Empty,
        gen_ai.usage.input_tokens = tracing::field::Empty,
        gen_ai.input.messages = tracing::field::Empty,
        gen_ai.output.messages = tracing::field::Empty,
        )
    } else {
    tracing::Span::current()
};
```

We ended up setting the actual span names to the name of the operation being carried out because formatted strings don't appear to be allowed in the span macros. Not that it matters because we can just re-format it in OTel Collector when it gets sent elsewhere, but it's a rather small detail.

So essentially we have this span, and then we wrap our request and response (a heavily async part of the function) in an async block and instrument it with the span. Within the async block, we then retrieve the span using `tracing::Span::current()` and fill in the required fields by using `span.record()`.

Effectively this allows us to carry out our default behaviour while (hopefully!) allowing users to be able to create basically whatever span they want and slapping it on top. For example, let's say you want to add a user ID or session ID to your span. You can just create your own span, then just instrument the future with it.

So far, so good.

However, streams are a trickier beast. They are lazy by default in Rust, so occasionally you can get issues where your span isn't tracked properly... even after you've used `tracing_futures` with the `futures-03` feature and instrumented the stream. This was primarily an issue with multi-turn agent streaming, where you are essentially using creating streams within a stream.

I pretty much ended up just trying to enter the span inside of the stream and it worked.

### Implications for end users
Now onto the fun part: how this affects end users (namely, you).

So obviously the pre-defined span is not particularly small, having about 10 fields each. If you have an agent layer on top of that, that's roughly about another 10 fields. Unless you are in production and actually need those metrics, it's not really a fun time for you.

If you don't care about Rig's observability at all, you can use the following in your `EnvFilter` (or as the `RUST_LOG` environment variable):
```bash
rig=none,debug
```
This deletes all `rig` logs, as well as requiring all other logs to a minimum log level of `debug` before being tracked.

However if you want the event logs but not the spans/trace spam, one of the solutions to this would be to create your own tracing subscriber that actively removes everything *except* the log message itself (and log level) from the print-out. Such an implementation can be found below:

```rust
#[derive(Clone)]
struct MessageOnlyLayer;

impl<S> Layer<S> for MessageOnlyLayer
where
    S: Subscriber + for<'a> LookupSpan<'a>,
{
    fn on_event(&self, event: &tracing::Event<'_>, _ctx: Context<'_, S>) {
        use tracing::field::{Field, Visit};

        struct MessageVisitor {
            message: Option<String>,
        }

        impl Visit for MessageVisitor {
            fn record_debug(&mut self, field: &Field, value: &dyn std::fmt::Debug) {
                if field.name() == "message" {
                    self.message = Some(format!("{:?}", value));
                }
            }
        }

        let mut visitor = MessageVisitor { message: None };
        event.record(&mut visitor);

        if let Some(msg) = visitor.message {
            let msg = msg.trim_matches('"');
            let metadata = event.metadata();

            let colored_level = match metadata.level() {
                &tracing::Level::TRACE => "\x1b[35mTRACE\x1b[0m", // Purple
                &tracing::Level::DEBUG => "\x1b[34mDEBUG\x1b[0m", // Blue
                &tracing::Level::INFO => "\x1b[32m INFO\x1b[0m",  // Green
                &tracing::Level::WARN => "\x1b[33m WARN\x1b[0m",  // Yellow
                &tracing::Level::ERROR => "\x1b[31mERROR\x1b[0m", // Red
            };
            let _ = writeln!(std::io::stdout(), "{colored_level} {msg}");
        }
    }
}
```

This can be then used like so:

```rust
tracing_subscriber::registry()
    .with(EnvFilter::new("info"))
    .with(MessageOnlyLayer)
    .init();
```

However, if you do actually intend on storing the traces/spans: great! That means you get to interact with the wonderful world that is the Rust OpenTelemetry crates.

Fortunately, there is mostly one simple rule to follow. The `tracing-opentelemetry` crate needs to be 1 version ahead of all the other `opentelemetry` crates - so if you are using `tracing-openelemetry v0.31.0`, the other OpenTelemetry crates need to be at `v0.30.0`.

Once you're done, all you need to do is to set the tracing layer up like so:
```rust
let exporter = opentelemetry_otlp::SpanExporter::builder()
    .with_http()
    .with_protocol(opentelemetry_otlp::Protocol::HttpBinary)
    .build()?;

let provider = SdkTracerProvider::builder()
    .with_batch_exporter(exporter)
    .with_resource(Resource::builder().with_service_name("rig-demo").build())
    .build();

let tracer = provider.tracer("rig-demo");

let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

tracing_subscriber::registry()
    .with(EnvFilter::new("info"))
    .with(otel_layer)
    .with(MessageOnlyLayer)
    .init();
```

Pretty easy, right? Kind of, anyway.

Once that's done, you pretty much just need to set up your otel collector and then it's sorted. There is a bit to do on setting up your otel collector and connecting to Langfuse, but that's beyond the scope of this article.

## Learnings
So, in terms of learnings there's a couple things I learned from this.
### Tracing isn't as hard as you think (beyond macros)
It seems that `tracing` seems to have acquired a bit of a reputation as being this really advanced system that is extremely difficult to use. It kind of is, but it also solves a very important issue - distributed tracing. That kind of problem requires very complex type and asynchronous machinery to solve competently.

Overall, writing my own spans was not too complicated and I didn't really have to think that much about it. In fact, I would go so far as to say that once I had learned to just make the spans myself, it was even easy. A lot of the issue comes down to primarily how to get the data into the span itself: chiefly, that a given field needs to actually be defined in the span (macro) first before you can actually give it a value. Otherwise, it just doesn't appear.

The other thing was mostly just trying to figure out what the API design is. At the moment it is pretty much only internally-facing so there's no actual public API changes.

### Generative AI observability is still pretty emergent
While the OTel GenAI SemConv standard is mostly pretty filled out, there are a few queries around things like streaming responses (for example: should we decide Time to First Token? what about total streaming time? etc) and the like.

This means that ultimately, some things may end up being placeholders until a proper convention has been made for the sake of ensuring that the user can extract useful information.

## Conclusion
Thanks for reading! This was a pretty fun project to do. Hopefully you are able to take something away from this article.

If you are interested in trying out Rig, [please check out the repo.](https://github.com/0xPlaygrounds/rig)
