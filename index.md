---
title: "Rocket: a hybrid approach to real-time cloud applications"
layout: default
---

> _Rocket is a technique for building real-time functionality on top of REST web services that leverages web standards like [Server-Sent Events][SSE] and [JSON Patch][RFC6902]._ Most importantly, it fits comfortably with how you're already building applications.

# A Tale of Two Paradigms

>> Just as light can act as both a _particle_ and a _wave_ so information can be thought as both a _document_ and a _stream_.

Cloud application developers are comfortable interacting with data like documents, according to REST conventions, but messages could also be subscribed to, as a stream, such that new messages pushed to the client in real-time.

Each approach has its particular strengths and weaknesses:

- **Documents** are _stateless_, _long-lived_ and _cacheable_, but must be polled for changes
- **Streams** are _stateful_, _ad-hoc_, and _real-time_, but has limited semantics for managing information

It's clear that real-time will be increasingly important as cloud applications become more connected and ubiquitous, but there will still be a need for the current request-response architecture.

## Rocket is a proposal for how to bridge this gap.

According to REST conventions, when a client makes an HTTP `GET` request to a resource endpoint, a list of records is returned. With Rocket, a client can additionally subscribe to changes for that resource by requesting an event stream at that endpoint.

<table id="document-versus-stream">
  <thead>
    <tr>
      <th></th>
      <th>Document</th>
      <th>Stream</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Request</td>
      <td>
<pre><code>GET /resources
Accept: application/json
</code></pre></td>
      <td>
<pre><code>GET /resources
Accept: text/event-stream
</code></pre></td>
    </tr>
    <tr>
      <td>Response</td>
      <td>
<pre><code>HTTP/1.1 200 OK
Content-Type: application/json

{"resources": [...]}
</code></pre></td>
      <td>
<pre><code>HTTP/1.1 200 OK
Content-Type: text/event-stream

event: patch
data: [{"op": "add", "path": "/resources/123", "value": {...}}]
</code></pre></td>
    </tr>
  </tbody>
</table>

Anytime a record is _created_, _updated_, or _destroyed_, an event is sent in real-time over the event stream, encoding the changes as a JSON Patch document.

<table>
  <thead>
    <tr>
      <th>Create</th>
      <th>Update</th>
      <th>Destroy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <tt>
<pre>{
  "op": "add",
  "path": "/resources/123",
  "value": {...}
}</pre>
        </tt>
      </td>
      <td>
        <tt>
<pre>{
  "op": "update",
  "path": "/resources/123",
  "value": {...}
}</pre>
        </tt>
      </td>
      <td>
        <tt>
<pre>{
  "op": "remove",
  "path": "/resources/123"
}</pre></tt>
      </td>
    </tr>
  </tbody>
</table>

# Building on a Solid Foundation

>> Building robust, scalable software is difficult, but not nearly as tricky as getting developers to agree on things. _Standards, Conventions, and Specifications allow us to stop bike-shedding, and get back to solving real problems._

## REST

[**Re**presentational **S**tate **T**ransfer][REST], or <acronym>REST</acronym>, is a convention for structuring web application interfaces.

Resources are accessed and manipulated using the existing vocabulary and semantics of HTTP:

- `GET /messages` **reads** a list of messages
- `POST /messages` **creates** a new message
- `GET /messages/123` **reads** a message
- `PUT /messages/123` **updates** a message
- `DELETE /messages/123` **destroys** a message

## Server-Sent Events

[Server-Sent Events][SSE] are used to push events to clients over a persistent HTTP connection. An event source stream is designated by the `text/event-stream` MIME type.

Event messages are composed of newline-delimited fields, each of which contains the field name, followed by a colon, followed by the associated value. The following fields are defined in the specification:

- `event`: The event's type.
- `id`: The event ID.
- `data`: The data field for the message. Multiple lines that begin with "data:" will have their lines concatenated with a newline.
- `retry`: The reconnection time to use when attempting to send the event. This must be an integer, specifying the reconnection time in milliseconds. If a non-integer value is specified, the field is ignored.

Although originally designed to send messages to browsers using the `EventSource` DOM element, Server-Sent Events can be generalized for use with non-browser clients as well.

## JSON Patch

[JSON Patch][RFC6902] describes a common format to represent changes in structured data. A JSON Patch response is designated by the `application/json-patch+json` MIME type.

A patch is comprised by an array of operations. Six operation types are defined in the specification: `add`, `remove`, `replace`, `move`, `copy`, and `test`. Each operation specifies its type (`op`), a `path`, and an optional `value` or `from` field:

    [
      { "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
      { "op": "remove", "path": "/a/b/c" },
      { "op": "replace", "path": "/a/b/c", "value": 42 },
      { "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
      { "op": "copy", "from": "/a/b/d", "path": "/a/b/e" },
      { "op": "test", "path": "/a/b/c", "value": "foo" }
    ]

Although the specification is described primarily in terms of applying a set of changes with an `HTTP` `PATCH` request made by the client to the server, the same structure could represent a change on the server that is communicated to clients as a means of resource synchronization.

# Implementations

### Server

- [Rack::Scaffold](https://github.com/mattt/rack-scaffold) has an [experimental branch](https://github.com/mattt/rack-scaffold/tree/experimental-rocket) that demonstrates how server-sent events can be incorporated systematically into a data provider.

### Client

- [AFRocketClient](https://github.com/AFNetworking/AFRocketClient) provides a unified client interface to Rocket, making it easy to build real-time functionality on top of an existing REST app.

# Contributing

## Discussion

If you have any strong ideas about how real-time cloud applications should be structured, it'd be great to get your perspective. Likewise, if you have any concerns or questions about Rocket in its concept or implementation, please share your thoughts.

In any case, feel free to start or join a conversation on [GitHub Issues](https://github.com/Rocket/rocket.github.io). Or if you're looking for more individual attention, send an e-mail to <mattt@heroku.com>.

## Implementations

Want to get in on the ground floor of Rocket's ecosystem? Try your hand at implementing Rocket on your favorite server or client language / framework. And if you do get something up and running, let us know by [submitting a pull request to mention the project on this page](https://github.com/Rocket/rocket.github.io).

# Thinking Out Loud

>> _We've only scratched the surface of what's possible with real-time cloud applications._ What follows are some random thoughts surrounding this space.

- Rocket could also be implemented using web sockets as its transport layer without any other changes.
- It's yet unclear how a traditional MVC architecture should delegate responsibilities for  resource change notifications. Because controllers mediate persistent connections to clients, a strong argument can be made for delegating responsibilities for notification entirely to the controller.
- That said, if JSON Patch were to be re-appropriated as a wire protocol for replication, changes could be streamed directly to clients from the database... which could be awesome.
- A pedantic concern: should the event stream response `Content-Type` somehow encode that the `data` fields are encoded as JSON Patch documents?
- Of the commonly-documented software design patterns, is there one that encapsulates an object providing a concrete implementation for executing patch commands on a collection?

# References

- [RFC 2616 - Hypertext Transfer Protocol — HTTP/1.1][RFC2616]
- [RFC 6902 - JavaScript Object Notation (JSON) Patch][RFC6902]
- [W3C Draft Specification - Server-Sent Events][SSE]
- [Wikipedia Article for REST](http://en.wikipedia.org/wiki/Representational_state_transfer)
- [Wikipedia Article for AJAX](http://en.wikipedia.org/wiki/Ajax_%28programming%29)
- [Wikipedia Article for Comet](http://en.wikipedia.org/wiki/Comet_%28programming%29)
- ["Using Server-Sent Events" on the Mozilla Developer Center](https://developer.mozilla.org/en-US/docs/Server-sent_events/Using_server-sent_events)

[RFC2616]: http://tools.ietf.org/html/rfc2616
[RFC6902]: http://tools.ietf.org/html/rfc6902
[SSE]: http://dev.w3.org/html5/eventsource/
[REST]: http://en.wikipedia.org/wiki/Representational_state_transfer
