![](https://raw.github.com/mattt/rocket/master/rocket.svg)

# Rocket

**a hybrid approach to real-time cloud applications**

_a.k.a REST over WebSockets_

---

### A Tale of Two Paradigms

Just as light can act as both a particle and a wave...

![](https://raw.github.com/mattt/rocket/master/figure-light-particle-wave.svg)

...so information can be thought as both a document and a stream.

![](https://raw.github.com/mattt/rocket/master/figure-information-document-stream.svg)

Cloud application developers are comfortable interacting with data like documents, according to REST conventions:

- `GET /messages` **reads** a list of messages
- `POST /messages` **creates** a new message
- `GET /messages/123` **reads** a message
- `PUT /messages/123` **updates** a message
- `DELETE /messages/123` **destroys** a message

But messages could also be subscribed to, as a stream, such that new messages pushed to the client in real-time.

- `READ` **streams** messages
- `WRITE` **creates** a new message

Each approach has its particular strengths and weaknesses:

- **Documents** are _stateless_, _long-lived_ and _cacheable_, but must be polled for changes
- **Streams** are _stateful_, _ad-hoc_, and _real-time_, but has limited semantics for managing information

It's clear that realtime will be increasingly important as cloud applications become more connected and ubiquitous, but there will still be a need for the current request-response architecture.

## Rocket is a proposal for how to bridge this gap.

It's pretty simple, actually: applications keep their existing REST architectures, and build realtime on top.

- When a client makes an HTTP `GET` request to a resource endpoint, a list of records is returned, as always.

#### Request

~~~
GET /resources HTTP/1.1
Host: example.com
Accept: application/json
~~~

#### Response

~~~
HTTP/1.1 200 OK
Content-Type: application/json

{
  "resources": [...]
}
~~~

- However, if a client opens a Websocket to that same endpoint, it is now subscribed to events for that resource, like a channel.

#### Request

~~~
GET /resources HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
~~~

#### Response

~~~
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
~~~

- Anytime a record is created, updated, or destroyed, a message is sent in real-time over the websocket. But here's the trick: the message sent over the websocket is, itself, _an HTTP message_.

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
<pre>HTTP/1.1 201 Created
Content-Type: application/json
Content-Location: /resources/123

{
  "resource": {...}
}</pre>
        </tt>
      </td>
      <td>
        <tt>
<pre>HTTP/1.1 202 Accepted
Content-Type: application/json
Content-Location: /resources/123

{
  "resource": {...}
}</pre>
        </tt>
      </td>
      <td>
        <tt>
<pre>HTTP/1.1 204 No Content
Content-Type: application/json
Content-Location: /resources/123</pre>
        </tt>
      </td>
    </tr>
  </tbody>
</table>

- Think of the websocket as providing responses for requests the client _should_ have made, but didn't have the knowledge to.
- This means that the same client-side logic can be used to serialize server responses, whether it came from an HTTP request or websocket message.

## In Practice

### Server

[Rack::Scaffold](https://github.com/mattt/rack-scaffold) has an [experimental branch](https://github.com/mattt/rack-scaffold/tree/experimental-rocket) that demonstrates how websockets can be incorporated systematically into a data provider.

> Its current implementation does messaging directly in the app itself, which prevents the architecture from scaling beyond a single server process. This will be addressed before being merged into master.

### Client

[AFRocketClient](https://github.com/AFNetworking/AFRocketClient) provides a client interface to Rocket:

~~~{objective-c}
[self.client SUBSCRIBE:@"/artists" parameters:nil success:^(NSURLRequest *request, NSHTTPURLResponse *response, id responseObject) {
    if (responseObject[@"artists"]) {
        [self.messages addObjectsFromArray:responseObject[@"artists"]];
    } else {
        NSUInteger idx = [self.messages indexOfObjectPassingTest:^BOOL(id obj, NSUInteger idx, BOOL *stop) {
            return [[obj valueForKey:@"url"] isEqual:[responseObject valueForKey:@"url"]];
        }];

        if (response.statusCode == 201) {
            [self.messages addObject:responseObject];
        } else if (response.statusCode == 202) {
            if (idx != NSNotFound) {
                [self.messages replaceObjectAtIndex:idx withObject:responseObject];
            } else {
                [self.messages addObject:responseObject];
            }
        } else if (response.statusCode == 204) {
            [self.messages removeObjectAtIndex:idx];
        }
    }
    [self.tableView reloadData];
} failure:^(NSError *error) {
    NSLog(@"Error: %@", error);
}];
~~~

A unified interface to handle data events from the server makes it easy to build real-time functionality on top of an existing REST app.

## Concluding Thoughts

It's not entirely clear what Rocket is, whether it's a recommendation, a convention, or something that could become a standard. At the very least, I think Rocket's ideas are compelling, and could make it much easier to build realtime software, without throwing out existing RESTful infrastructure.

As far as I can tell, there is no existing standard for using websockets as a transport layer for web application state. I've scoured the internet in an attempt to find anything to base this work off of, but was not able to find anything too similar.

Much of Rocket's success is contingent on Heroku adding websocket support to its platform, allowing developers to experiment with this new architecture without any addons or additional configuration. I remain hopeful that we'll be able to ship something soon.

I'd love nothing to talk through this whole space with more people, so if you have any questions or comments, don't hesitate to get in touch: <mattt@heroku.com>

