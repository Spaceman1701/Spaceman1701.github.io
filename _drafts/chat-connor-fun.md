---
title: "chat.connor.fun or: How I Built a REST Service in a Week (or so)"
layout: single
---

In this post I'm going to discuss [chat.connor.fun]() (link pending), a RESTful webchat service I built with [Connor Hudson]()
and [Aaron Aaeng]() over winter break. The actual app is really just an inside joke that went to far, but we had a good time building
it. It actually turned out to be quite a challenge to build since we wanted to make something that was reasonably high quality in a very short
amount of time. On top of that, I wanted to get some more experience working with [Go](). 

Unlike a lot of my other posts about projects, this isn't really going to be a post-mortem, but more of a discussion of how the development process
went. Since are development schedule was incredibly accelerated, we sort of stumbled upon development practices that worked and I feel like some
of what we came up with is rather interesting. I also think the technical aspects of the project are reasonably interesting since (the backend at least)
was pretty much built from scratch. 

## What is chat.connor.fun?

Before I get into anything else, it's probably worth talking about what chat.connor.fun actually is/is supposed to be. Fundamentally, chat.connor.fun
is a instant messaging web application.

chat.connor.fun is intended to be a location based messaging service. The user's physical location is used to assign them to a chat room where they can talk 
to other nearby users. In order to support this, chat.connor.fun supports an arbitrary number of active chat rooms each with an arbitrary number of users. Each
chat room is associated with a real-world geographic area. When a user opens the app they are then recommended nearby chat rooms (at least, in theory - this feature
is, at the time of writing, not yet implemented in the front end). 

chat.connor.fun requires that a user create an account before they can send messages. It also supports multiple user roles. This is used primarily for administration
right now (e.g admins are allowed to create new rooms), but we have plans to add support for moderation in the future (e.g allowing users to report offensive messages and 
then allowing moderators to review the reported message and take appropriate action).

chat.connor.fun also supports profanity filtering for messages. This allows for some level of automatic moderation. This feature was very important to us as we believed
any online forum like chat.connor.fun can quickly degrade without any sort of moderation. This is also the reason we require users to create accounts before sending messages:
it takes away a user's anonymity and allows the user to be banned. 

## How does it work?

On the frontend, chat.connor.fun is a single page react.js app that follows the [Progressive Web App]() development practices. On the
backend it's a monolithic REST service. Instant messaging is handled through [Websockets](). Since I only worked on the backend, that's 
all I'll be discussing here.

As I previously mentioned, chat.connor.fun's backend is written in [Go](). For this project, I wanted to experiment with really committing to
the REST philosophy. That means embracing the core principles of REST across the entire application. Every component of the backend service, from the
auth system to the domain model was designed to be part of a REST service. While this might not be a great idea in a real application (as it is probably
a good idea to keep business logic decoupled with the method for accessing it), it was a fun little experiment for this project. 

I also decided to use Websockets for the chat feature as an experiment. I'd never used them before and instant messaging seemed like a textbook case for
them. It also allowed me to justify using Go for the project since Websockets require asynchronous control that can be rather difficult in other languages.

For this project I also implemented my own authorization and authentication stack. I used industry standard technology like JWTs for many of the low level
components, but the high level code was completely custom. I'll have an entire section of this further down.

## Details

The rest of this post will be broken down into two sections: Development methodology and Technology. I'm choosing to focus on methodology more than I normally
do because I feel that it is the more interesting aspect of this project. While chat.connor.fun is not a very revolutionary piece of technology, it was built
on a very accelerated timescale. 

## Technology

### Go

The first bit of technology I want to discuss is Go. Before this project most of the web development I've done was with Java or Python. As such, I've become 
very used to their models. In this project, Go felt like a breath of fresh air. Gone where Python's tenuous build requirements (I mean, Flask's application model
requires the developer import modules in a certain order) and Java's Giant Evil Monolithic Frameworks of Doom (a topic I've [discussed]() before). 

For the backend service I decided to use the [Echo Framework]() for Go. This choice was actually a rather difficult one to make as a lot of people on the Go subreddit seemed very against the idea
of using a framework. I guess the word "framework" means something different in the Go world than in the Java world, because compared to the monstrosity that is the [Spring Framework](),
Echo is tiny. Instead of being a giant inversion of control framework, Echo came with a relatively small set of useful utilities. What I found most useful was the simplification of the 
Go standard library's HTTP request handlers. I also liked the addition of simple middleware registration and easy-to-use HATEOAS support. 

There were some things I found a bit akward about Echo, however. In the Java and Python world's I've become rather used to dependency injection as a means to improve testability. While I have 
some problems with the way many Java frameworks handle it (I don't like functionality being hidden from me), it's massively useful when unit tests require mocks. Since echo has no good way to support
this, I ended up having to develop a special pattern for HTTP handlers. In Echo (and in the Go standard library), HTTP handlers are registered by passing the handler function to the a registration 
function. This means that any external state must be global as the handler function cannot take any special parameters. This poses a major problem for testing (and introduces ugly patterns into the design).
Luckily, Go has full closure support which ended up providing (what I think) is a pretty elegant solution: HTTP handler functions can close over variables passed to a function that wraps over them. This way 
each handler function can be passed a reference to the external state it needs at registration time. This ends up looking something like this:
```go
func SomeEndpoint(repository db.TransactionalRepository, someState pkg.SomeStateStruct) echo.HandlerFunc {
    return func(c echo.Context) error {
        //do some stuff
    }
}
```
which gets register like this:
```go
func RegisterEndpoints(repository db.TransactionalRepository, someState pkg.SomeStateStruct, someOtherSate ...) {
    e.GET("/someendpoint", SomeEndpoint(repository, someState));
    //register more here
}
```

I also felt that Go had some unfortunate limitations. First, I think the packaging system is a little too flexible. I'm still a fan of the way Java's packaging system forces you to think about
structure of your project and, to some extent, forces the developer to structure the project properly. Go's attempts to do this, but I think it falls a little bit short. The requirement that the 
package name of each symbol get placed in front of every use is a bit bothersome and encourages a project structure that's a bit more flat than I think is ideal. I'm also not sure how I
feel about the use of `error` in place of exceptions. It results in code that's filled with `if error != nil` checks (it actually reminds me a lot of C). I sort of understand the argument that
using the `error` checks as opposed to `try... catch` blocks results in all of the non-error handling code being in the "main" execution branch, but I think in practice it ends up obfuscating 
the function logic more than I'd like. With exceptions, all the error handling remains separate from the normal logic which improves readability considerably. In my opinion, 
a lot of the [error handling patterns]() just end up recreating a lot of the functionality that you would get with a normal exception based approach to error handling. 

### The REST Philosophy

TODO

### Authorization and Authentication

TODO

### Websockets

TODO

## Methodology

For this project, I worked on a small team. Each of us had well defined areas that we'd be working on. This meant that we could all work pretty much independently, at
least when we first started.