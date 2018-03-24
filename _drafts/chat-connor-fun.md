---
title: "Chat.connor.fun or: How I Built a REST Service in a Week (or so)"
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

I don't think this section will be all that long since I suspect everything that I could say here has been said before. I've never really tried to embrace REST quite as much as I did for the 
backend service on this project before, and it left me with mixed feelings. I also never got around to implementing HATEOAS because it seemed like it would be more trouble than it was worth
given the simplicity of our API. I think generally REST can lead to clean and concise code, but it has some limitations and idiosyncrasy that have been addressed by newer technologies like GraphQL. 
For this project I tried to get into the semantics of what a single resource really was, and I found that it is often difficult to decide. 

I also found that determining a resource's path could be a challenge. For example, should a `Message` resource be found at the root path (e.g `/message/xyz`), the path of the user who created the 
message (e.g `/user/123/message/xyz`), or the path of the room in which it was sent (e.g. `/room/123/message/xyz`)? There's no great answer to this question in my opinion. All three methods seem to
separate the message from its context (i.e. a message belongs to both a user and a room), while the two paths that add some context seem strange, since each message has a universe-wide UUID. One might 
suggest that the message be available at each endpoint, but I think that just leads to more confusion. In a typical REST pattern, each resource is wholly defined by its path. Having the message available
at multiple paths would give the message multiple identities (i.e. it would appear as three different resources). What I ended up choosing was the first path (`/message`). The endpoint allows the client 
to filter the response by user and room using query parameters. While this works, I still think it lacks elegance.

In practice we found that keeping a universe-wide UUID to identify objects was a whole lot easier than doing anything else. In modern programming languages with good library support, generating a UUID
v5 is fast and simple. Keeping the UUID for each resource was convenient because it more easily allows that resource to mutate. For example, imagine that `User` resources are identified by a username. 
Since there are many relations between `User` resources and other resources (such as messages or chat rooms), if the user ever wanted to change their username, the service would have to track all of these
relations and update them. This makes the code much more difficult to maintain in my opinion as it means changing part of the message handling logic could un-intuitively require changes to the user handling
logic (as changes to a `User` will require changes to all resources related to it). By using a UUID, relations involving `User` resources are not broken when the `User` is mutated.  

Using UUID for identification can lead to some confusion in a REST model. Functionally, the client could request specific resources from the root path without a problem (since all UUIDs are unique across 
all resource types). The fact that this is possible demonstrates the problem: REST itself enforces a method resource identification based on the path - adding another form of universal identity requires the
business logic to keep a "mapping" of some sort between the two identities. This "mapping" is a bit unintuitive, however. The actual "mapping" ends up being contained in the controller logic for the REST 
endpoints. When writing a new endpoint, the developer must have a priori knowledge about the mapping. The case of the `Message` resource is a great example of this - in order to properly fulfill identity 
in the "REST way", the `Message` must only be available at a single endpoint, but its universal identity makes it easy to implement a system where it's available in multiple ways. THe developer solves this problem
by choosing a single endpoint for `Message` resources. This choice creates an _implicit_ mapping between the two identities which must be enforced in any future changes. The problem is that there is no good way
for future developers to know how this mapping works since it is entangled with the logic of the controller.

I think GraphQL may provide a solution to this type of problem because it decouples the identity of a resource with the method by which it is accessed. In GraphQL, a resource is accessed based on its relation
to other resources. To the client, this is still very intuitive since resources are typically accessed based on a relation to the current client state. GraphQL leaves the universal identification of an object
to the business logic, which removes the necessity of a mapping between identities all together. This isn't to say that GraphQL is a perfect technology, but discussing its problems is outside of the scope
of this article.

### Authorization and Authentication

For the backend service I designed and built a custom Role Based Access Control (RBAC) system. I don't think it's anything revolutionary or new, but it was very interesting to implement and I took most of the 2ish week development period to get working (and some features still aren't in yet). The system is implemented as two stages of middleware that get run before every request. Access control as two separate domains: Authentication and Authorization. As such, the chat.connor.fun RBAC system can be broken down into these two parts.

#### Authentication

For authentication, we used signed JSON Web Tokens transmitted with every request. Our implementation was fairly standard and probably doesn't warrant a lot of discussion. When the user logs in, the service
responds by transmitting a JWT with claims about who the user is. The JWT also contains a creation date and expiration date to ensure that the JWT is still valid. If the client transmits a valid JWT with a request
the user's identification information is passed along to the authorization component. If no JWT is transmitted the request is still passed forward to the authorization component, but it is noted that the 
request is being made anonymously. In the final case, if a client transmits an invalid JWT, a `status 401` is returned, signaling to the client that it should request a new JWT.

#### Authorization

After the client making a request is authenticated, the middleware must determine if the client is authorized to do what they are requesting. Like most RBACs, allowed client actions are represented as `permission`s. In order to decide if a client to authorized to make a request, the authorization system must decide if the client has a `Role` that has a `Permission` that allows the request. Permissions can also be attached to a request in special ways (i.e. extra permissions can be noted in an authorization table in the database or can be transmitted to the server as claims in the JWT). 

As such, there are three clear objects necessary for the authorization system `User`, `Role`, and `Permission`. Both the `User` and `Role` objects provide information on _who the client is_. The `Permission` object tells us _what the client can do_. 

For chat.connor.fun, we felt that we would need both fine grained resource-level permissions (e.g. user 123 can update message xyz) and very general high-level permissions (e.g. all users can read all messages). Following my "all REST" experiment, I decided to base permissions on the actual path of the resource. In order to support widespread permissions, permissions can support wildcard characters. If a user wants access a resource located at `/foo/bar/1234`, they would need a permission that grants access to that path such as `/foo/bar/*` or `/for/bar/1234`. The first permission uses a wildcard to give the user access to all resources located under `/foo/bar`. The second gives the client specific access to the resource `/foo/bar/1234`. I liked this approach as there was a clear, enforceable, and intuitive mapping between permissions and resources. Permissions also need to contain information about what type of actions a user can perform with a resource. For this, I decided to use a simple set of `CRUD` operations. A permission contains a field that specifies what subset of `CRUD` operations it allows.

Another nifty feature of this path based permission system was the ability to automatically decide what paths should be protected. In chat.connor.fun, every endpoint requires authorization by default. Endpoints can be globally excluded (mostly for serving static files). Otherwise, the client must have a permission that grants access to the path. This essentially comes "for free" since it is simple to programmatically 
determine if a permission applies to a given path. The developer doesn't need to do any wiring or configuration to ensure their endpoint is protected. They must simply add a permission to the appropriate roles. Since permissions identify resources and actions the same way the client does using HTTP requests, the permission system is expressive enough to represent any possible set of access restrictions. 

This system has some problems, of course. The most prominent of these is that it only works if every resource is accessed using REST. This was mostly not a problem for me, since the backend service is almost entirely RESTful. However, live chat rooms are accessed using Websockets and therefore exist outside of the REST model. In order to solve this problem, a I devised a rather inelegant solution that authorizes clients before doing the protocol upgrade from HTTPS to WSS. This leads to difficulties when the client's permissions change while connected to a chat room. Currently, our solution to this is to reconnect to any open Websockets whenever the client does anything that could cause a change of permissions. I think it's relatively obvious why this is suboptimal.

The other major problem is that the system has no concept of resource ownership. When a client creates a resource, there's no way to denote that they actually own it. I decided to do this because I felt it would
solve some potential problems involving the removal of users. I was worried that an ownership system could result in ownerless resources which cannot be removed from the database without special action (since 
no client had permission to remove them except the owner). In retrospect, I think this fear was overblown and lead to some unintuitive solutions to problems. For example, since a user cannot own a message, they 
must instead gain a special permission that grants them access to special permissions regarding messages (such as updating the message). While this has the same functional outcome as a permission system that
allowed ownership, it ends up spreading out a client's permissions across many sources which will probably hurt maintainability. I'm afraid that if we ever need to change what permissions the creator of a message has, it will be somewhat difficult to apply the changes to messages that have already been sent (as we will have to modify permissions already in the database). An ownership system could allow a single permission to show that only the owner of a resource can update it which improves clarity and maintainability. In the aforementioned scenario, a change to the permissions of all previously sent message would be as simple
as changing a line in a configuration file.

The authorization system decides whether a user can preform a given request by assembling all of the client's permissions from each possible source. A client is authorized to perform a request if any permission from any of these sources allows it. In this way the authorization system is as permissive as possible. This design makes it much simpler to debug and maintain.

The possible sources for sources include the client's roles, the permissions table in the database, and any permission claims made by the client in their JWT. Clients can have multiple roles in order to 
support more complex identities. Permissions in the database are used to grant permissions to specific users. For example, the user that creates a message should have permission to edit it. Thus, every time a
user creates a message, a permission is added to the table which grants them access to update it. While there are currently no cases where we use them, permissions in the JWT can be useful when granted special temporary permissions. For example, a link could contain an encoded JWT which grants an otherwise anonymous client the ability to view a specific message that they normally could not access. It's worth noting
that the use of JWTs is decoupled from the authorization system. These types of special permissions are viewed by the authorization system as a special set of permissions that are passed in from the authentication system. 

Roles can be configured without making changes to the source code. The set of available roles and the permissions that belong to each role are defined in a JSON configuration file. Each role is given a unique name and a set of permissions. There are two special roles that must be defined: `admin` and `banned`. The admin role indicates that the client should be authorized to perform _any_ action. The `banned` role indicates that the client should _not_ be authorized to perform any action. I'm not super happy with the necessity for these special roles, but they provide a degree of simplicity to permission management in special cases. I would eventually like to do away with the `banned` role since it breaks the pattern of being maximally permissive (as it is the only role that can _remove_ permissions).

### General Thoughts on the RBAC

Overall, I think the RBAC system turned out pretty nicely. While the code is pretty messy, it's missing some features, and there are some design issues, I think it's about as good as I could make it given the
limited time constraints. I built the RBAC first, and it took almost an entire week to complete. It was an interesting challenge, however, and I feel that I learned a lot of the design of access control systems.
The ability to "auto-authorize" a path also saved a lot of development time, as changing authorization configuration was as simple as changing a few lines in a JSON file. 

### Websockets

The Websocket standard seems like a mess. Simple problems like transmitting user authentication data ended up becoming somewhat complex messes and more complex problems like proper rate limiting
seemed completely out of reach given our timescale. 

For example, there's no way for the client transmit an authorization header when making a request to upgrade from HTTPS to WSS. This forced us into a no-win scenario for authorizing requests to
connect to a chat room. Either we implemented a special authentication and authorization subsystem for handling access control over the websocket connection (which would likely require the duplication
of logic or result it a tight coupling between the RBAC system and the websocket controller), or we could do something strange to transmit the required access control information. We ended up going for
the later approach (which was less elegant but required the fewest code changes). We use the protocol field of the upgrade request to transmit the encoded JWT and added a special component to the 
authentication that allowed it to be retrieved. This works fine for some client implementations of websockets, but we found that others (like Chrome's) require that the websocket retransmit the protocol
string back to the client. This forced us into making some strange, inelegant changes to allow the websocket controller to gain access to the required data. 

Given more time I think we could've found a better solution to this problem, but I legitimately feel that this should not have been a problem in the first place. I cannot see any reason why authorization headers
should not be allowed when making upgrade requests.

### Persistance Layer

I decided not to use any form of ORM for this project so it's probably worth discussing the persistance layer solution I arrived at. One problem I ran into is that early on I did not think it would be necessary for us to make multiple queries to the database in a transaction. Later in development we decided to add support for email verification. My implementation required that a second table be written to in the database to store the one time use verification code. An error in creating the user could've left the verification table in an inconsistent state. Thus I had to refactor the entire persistance layer to add proper support for transactions. I also didn't want to break old code too much, so I had to build a system that could add drop in support for transactions very easily. I think the system I ended up developing is fairly
elegant, but not perfect.

## Methodology

For this project, I worked on a small team. Each of us had well defined areas that we'd be working on. This meant that we could all work pretty much independently, at
least when we first started. Overtime we naturally expanded to more involved project management. 