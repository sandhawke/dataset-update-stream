# dataset-update-stream
[![NPM version][npm-image]][npm-url]

**A tiny protocol for maintaining a local mirror of a remote dataset**

Status: Just a proposal at the moment. 

## Motivation

Sometimes you want a client to keep an up-to-date copy of some data on a server. This client might be part of another server, a mobile app, or some app running in a web browser.

An efficient protocol for mirroring data will depend on the kind of data and how it changes.  This protocol applies to to data where:
1. The data consists of a **set** of items, an unordered collection without duplicates. For example, this might be rows in a SQL database table or a spreadsheet, as long as duplicates are not allowed. (Key fields are not necessary, though.)
2. Each item can be serialized (stringified) to a string without newlines and then accurately reconstituted (parsed). For example, any objects which survive JSON.parse(JSON.stringify(item))) are fine, since JSON escapes newlines.  The encoding does not have to be JSON, as long as client and server agree on the encoding.

## Design Discussion

### Notifying Client

Some general design options:

* Clients could poll. Using [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) this can be fairly efficient.  If a propagation delay of hours is okay, this is probably fine.  Maybe even minutes.  Probably not viable if you want to be synchronized within seconds, though.  Requires transmitting the full content with each change, which could also be wasteful if there are many small changes.

* Using [WebSub](https://www.w3.org/TR/websub/) (standard pub-sub via webhooks) gets rid of the propagation delay.  Unfortunately, it requires the client to be available on the web as a server.  Also, while fairly simple, it's not trivial to implement.  Requires transmitting full content with each change, until/unless WebSub is extended to support patches (which would fit the architecture, but is not currently standardized).

* For getting rapid change propagation while working in the browser, we need a "server push" technology.  The main options are [Long Poll](https://en.wikipedia.org/wiki/Push_technology#Long_polling), [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events), and [WebSockets](https://en.wikipedia.org/wiki/WebSocket).  At this point, Long Poll appears to be obsolete due to SSE and WebSockets. Since this application only requires one-pay communication, SSE seems like the best option.

SSE is a very simple streaming HTTP(S) protocol, implementable with a few lines of server code, and [supported](https://caniuse.com/#search=eventsource) in all modern browsers (except Edge) as [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource).  ([Polyfills](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events#Tools) exist for Edge and older browsers.)  It was [standardized by W3C](https://www.w3.org/TR/eventsource/) and is now maintained as [part of the HTML Living Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events).

An SSE stream consists of newline-delimited lines, grouped into events, separated by a blank line.  Each event consists of an event type and zero or more data lines.  (There are additional features we don't use.)

### Addition of Items

To signal the addition of items, then, we can simply use an event-stream like this:

    event: add
    data: encoding of item to be added
    data: encoding of another item to be added


### Removal of Items

There are some design options for how to remove items.
1. Send the full text of the encoding of the item. This can only remove one item at a time.
2. Send some of the text, and use a wildcard or regex mechanism, allowing shorter deletion messages and removing multiple items at once, under certain conditions. This includes the useful feature of being able to delete all items to start again, if the synchronization state is unknown. It is not clear how, however, how to implement this efficiently at either end in the general case.
3. Send item numbers, with optional ranges. This is fairly easy to implement and terse, and allows removing all items, but requires the client maintain some kind of numbering.  There are a few variations on this:
    * sequence numbers (this was the 47th item added)
    * index numbers (this is the 47th item currently held, in the order they were added)
    * reverse sequence numbers, where item "1" would be the most recently added item
    * reverse index numbers, where item "1" would be the most recently added item still held. 

For simplicity at this point, we use the first option, but recognize a future version of this protocol might support one of the later ones.  We also include an event for resetting to the empty set.

    event: remove
    data: encoding of the item to be removed
    data: encoding of another item to be removed

    event: remove-all


## Finding the update stream

Given the URL of some dataset resource, how do you find the associated update stream?  Also, if there are multiple update-stream protocols, how do you find the right one?

The basic options are:
1. The client can tell the server it wants an update stream and which protocols it implements.
    * HTTP content negotiation?  Doesn't work because SSE mandates "text/event-stream"
    * Use an HTTP [Prefer header](https://tools.ietf.org/html/rfc7240)? Doesn't work because the EventSource API in browsers does not include a way to set additional headers.
    * Modify the URL in some way, such as adding a query parameter?  There's no way to do this in a standard way, and the text of URLs is generally best treated as opaque at this layer.
2. The server can tell the client that it offers one or more different update streams at other URLs, and let the client pick among them.
    * Inside the data? Would only work for some kinds of datasets
    * Using Link: headers, with rel=alternate.  Unclear how to indicate to the client which alternate to use.
    * Using Link: headers, with rel=<some assigned identifier>. This should work, but technicall requires IANA processing.
    * Using Link: headers, with rel=<url based on this spec>. This should work, although it embeds a domain name into the protocol, which might not be good, long-term.
    * Using Link: headers, with rel=<uuid url>.   This should also work, but is a little hard to read.

For now, let's go with:

    Link: <...some-event-stream-url...>; rel="https://sandhawke.github.io/dataset-update-steam/v1"

Do we want to include in that identier the serialization syntax used? I don't think we need to; we can rely on the dataset's URL syntax, which can be content-negotiated, if necessary.

### Continuation

TBD.

Continuation?  Our protocol say the event-stream URL MUST gracefully accept an optional query parameter "since-version-etag", although it MAY ignore it.  And/or since-version-id or since-version-integrity.  Maybe should use Link-Template.

## Example

Consider a table listing the current temperature in selected cities:

|  City     | TempF |
|:---------:|:-----:|
| Boston    |   26  |
| Barcelona |   54  |

This could be published on the Web in a CSV file:

    $ curl https://example.org/citytemp.csv
    City,TempF
    Boston,26
    Barcelona,54
    $

We can look to see if there's an update stream:

    $ curl -v https://example.org/citytemp.csv
    ...
    < link: <https://example.org/citytemp.csvstream>; rel="https://sandhawke.github.io/dataset-update-steam/v1"
    ...

    $ curl https://example.org/citytemp.csvstream
    event: remove-all

    event: add
    data: City,TempF
    data: Boston,26
    data: Barcelona,54

    event: remove
    data: Boston,26

    event: add
    data: Boston, 28

    ^C

This event stream will continue until interrupted (shown here as ^C).

There's a bit of hack with CSV: we just treat the first CSV data item as the header line.  This could cause problems if the header line could be mistaken for a data item, but is okay in this case because the TempF column is numeric, so it could never have the value "TempF", even if we had a city name "City". To avoid this sort of issue, more robust line-oriented formats like [JSON lines](http://jsonlines.org/) and [N-Quads](https://www.w3.org/TR/n-quads/) are often preferrable.

## Event Types

### add

Zero or more items were added to the origin data set and should be added to the mirror.  Each "data" line contains the serialization of one such item.

(Issue: do we need that restriction?  We could allow all the datalines to be taken at once.  Newlines are even okay.)

For example, in N-Quads:

    event: add
    data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>.
    data: <http://example.org/vocab#Boston> <http://example.org/vocab#label> "Boston, Massachusetts".


### remove

Zero or more items were removed from the origin data set and should be removed from the mirror. The "data" lines should be parsed as with the "add" event, but now the serialized items are to be removed.

### remove-all

Reset the mirror to being empty.  

### update-response-headers (optional)

This allows the server to inform the client about various metadata it would have gotten during a GET of the dataset resource. It can be used at the start, to provide metadata that doesn't apply to the event stream, and also as data changes.  Servers MAY send these events, and clients MAY ignore them.

For example:

    event: update-response-headers
    data: last-modified: Tue, 11 Dec 2018 10:40:12 GMT
    data: etag: "8809-57cbcb4c41b00;89-3f26bd17a2f00"

These header fields MUST be the same as a client would have received doing a GET on the origin resource at that point in time.

Some additional header fields are suggested for use in this context which are not standard.  They will be proposed to IETF if they prove useful:

#### Item-Count

For simple error detection.

    event: remove-all

    event: update-response-headers
    data: Item-Count: 0

    event: add
    data: <https://example.org/#a> <https://example.org/#b> <https://example.org/#c>.

    event: update-response-headers
    data: Item-Count: 1

#### Link rel=self

(Issue: Or maybe rel=canonical?)

"Link" with "rel=self" (originally from [atom](https://tools.ietf.org/html/rfc4287)) is used to indicate the URL by which the Dataset refers to itself, for its metadata.  For example:

    event: update-response-headers
    data: Link: <http://example.org/ds1>; rel=self

    event: line-add
    data: <http://example.org/ds1> <http://purl.org/dc/terms/creator> "Alice Example".

Here, the dataset is telling us the name of its creator.

Editor's Note: this use of rel=self needs to be tested and more widely considered.  See [How can you embed metadata in an N-Quads file and have it survive the file being copied/moved/proxied to a different URL?](https://www.quora.com/unanswered/How-can-you-embed-metadata-in-an-N-Quads-file-and-have-it-survive-the-file-being-copied-moved-proxied-to-a-different-URL)

#### Version-Integrity

"Version-Integrity" allows specifying a secure hash of what would be the original resource representation of the current state, computed as per (Subresource Integrity)[https://www.w3.org/TR/SRI/] and [Version Integrity](https://github.com/sandhawke/version-integrity).  Comparison should be done such that either base64 or base64url encoding can be used.

For example:

    event: remove-all
    
    event: update-response-headers
    data: Version-Integrity: sha256-47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU=
    
    event: add
    data: <https://example.org/#a> <https://example.org/#b> <https://example.org/#c>.
    
    event: update-response-headers
    data: Version-Integrity: sha256-ejVcS-5rqOG7TXp8VZ7wRKLuLEmbOvp4HyT1YULD1fg=

These strings can be used to make sure changes are being processed properly, and potentially to securely resume an event string from a given state.

Design alternatives:
* We could assume if the etag looks like a resource-integrity string (starting with "shaNNN"), it is one.  It seems unlikely IETF would like this header, after removing Content-MD5.
* We could use rel=canonical with a version-integrity URL.


[npm-image]: https://img.shields.io/npm/v/dataset-update-stream.svg?style=flat-square
[npm-url]: https://npmjs.org/package/dataset-update-stream
