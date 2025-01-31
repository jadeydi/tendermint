# RFC 006: Event Subscription

## Changelog

- 30-Oct-2021: Initial draft (@creachadair)

## Abstract

The Tendermint consensus node allows clients to subscribe to its event stream
via methods on its RPC service.  The ability to view the event stream is
valuable for clients, but the current implementation has some deficiencies that
make it difficult for some clients to use effectively. This RFC documents these
issues and discusses possible approaches to solving them.


## Background

A running Tendermint consensus node exports a [JSON-RPC service][rpc-service]
that provides a [large set of methods][rpc-methods] for inspecting and
interacting with the node.  One important cluster of these methods are the
`subscribe`, `unsubscribe`, and `unsubscribe_all` methods, which permit clients
to subscribe to a filtered stream of the [events generated by the node][events]
as it runs.

Unlike the other methods of the service, the methods in the "event
subscription" cluster are not accessible via [ordinary HTTP GET or POST
requests][rpc-transport], but require upgrading the HTTP connection to a
[websocket][ws].  This is necessary because the `subscribe` request needs a
persistent channel to deliver results back to the client, and an ordinary HTTP
connection does not reliably persist across multiple requests.  Since these
methods do not work properly without a persistent channel, they are _only_
exported via a websocket connection, and are not routed for plain HTTP.


## Discussion

There are some operational problems with the current implementation of event
subscription in the RPC service:

- **Event delivery is not valid JSON-RPC.** When a client issues a `subscribe`
  request, the server replies (correctly) with an initial empty acknowledgement
  (`{}`). After that, each matching event is delivered "unsolicited" (without
  another request from the client), as a separate [response object][json-response]
  with the same ID as the initial request.

  This matters because it means a standard JSON-RPC client library can't
  interact correctly with the event subscription mechanism.

  Even for clients that can handle unsolicited values pushed by the server,
  these responses are invalid: They have an ID, so they cannot be treated as
  [notifications][json-notify]; but the ID corresponds to a request that was
  already completed.  In practice, this means that general-purpose JSON-RPC
  libraries cannot use this method correctly -- it requires a custom client.

  The Go RPC client from the Tendermint core can support this case, but clients
  in other languages have no easy solution.

  This is the cause of issue [#2949][issue2949].

- **Subscriptions are terminated by disconnection.** When the connection to the
  client is interrupted, the subscription is silently dropped.

  This is a reasonable behavior, but it matters because a client whose
  subscription is dropped gets no useful error feedback, just a closed
  connection.  Should they try again?  Is the node overloaded?  Was the client
  too slow?  Did the caller forget to respond to pings? Debugging these kinds
  of failures is unnecessarily painful.

  Websockets compound this, because websocket connections time out if no
  traffic is seen for a while, and keeping them alive requires active
  cooperation between the client and server.  With a plain TCP socket, liveness
  is handled transparently by the keepalive mechanism.  On a websocket,
  however, one side has to occasionally send a PING (if the connection is
  otherwise idle).  The other side must return a matching PONG in time, or the
  connection is dropped.  Apart from being tedious, this is highly susceptible
  to CPU load.

  The Tendermint Go implementation automatically sends and responds to pings.
  Clients in other languages (or not wanting to use the Tendermint libraries)
  need to handle it explicitly.  This burdens the client for no practical
  benefit: A subscriber has no information about when matching events may be
  available, so it shouldn't have to participate in keeping the connection
  alive.

- **Mismatched load profiles.** Most of the RPC service is mainly important for
  low-volume local use, either by the application the node serves (e.g., the
  ABCI methods) or by the node operator (e.g., the info methods).  Event
  subscription is important for remote clients, and may represent a much higher
  volume of traffic.

  This matters because both are using the same JSON-RPC mechanism. For
  low-volume local use, the ergonomics of JSON-RPC are a good fit: It's easy to
  issue queries from the command line (e.g., using `curl`) or to write scripts
  that call the RPC methods to monitor the running node.

  For high-volume remote use, JSON-RPC is not such a good fit: Even leaving
  aside the non-standard delivery protocol mentioned above, the time and memory
  cost of encoding event data matters for the stability of the node when there
  can be potentially hundreds of subscribers. Moreover, a subscription is
  long-lived compared to most RPC methods, in that it may persist as long the
  node is active.

- **Mismatched security profiles.** The RPC service exports several methods
  that should not be open to arbitrary remote callers, both for correctness
  reasons (e.g., `remove_tx` and `broadcast_tx_*`) and for operational
  stability reasons (e.g., `tx_search`). A node may still need to expose
  events, however, to support UI tools.

  This matters, because all the methods share the same network endpoint. While
  it is possible to block the top-level GET and POST handlers with a proxy,
  exposing the `/websocket` handler exposes not _only_ the event subscription
  methods, but the rest of the service as well.

### Possible Improvements

There are several things we could do to improve the experience of developers
who need to subscribe to events from the consensus node. These are not all
mutually exclusive.

1. **Split event subscription into a separate service**. Instead of exposing
   event subscription on the same endpoint as the rest of the RPC service,
   dedicate a separate endpoint on the node for _only_ event subscription.  The
   rest of the RPC services (_sans_ events) would remain as-is.

   This would make it easy to disable or firewall outside access to sensitive
   RPC methods, without blocking access to event subscription (and vice versa).
   This is probably worth doing, even if we don't take any of the other steps
   described here.

2. **Use a different protocol for event subscription.** There are various ways
   we could approach this, depending how much we're willing to shake up the
   current API. Here are sketches of a few options:

   - Keep the websocket, but rework the API to be more JSON-RPC compliant,
     perhaps by converting event delivery into notifications.  This is less
     up-front change for existing clients, but retains all of the existing
     implementation complexity, and doesn't contribute much toward more serious
     performance and UX improvements later.

   - Switch from websocket to plain HTTP, and rework the subscription API to
     use a more conventional request/response pattern instead of streaming.
     This is a little more up-front work for existing clients, but leverages
     better library support for clients not written in Go.

     The protocol would become more chatty, but we could mitigate that with
     batching, and in return we would get more control over what to do about
     slow clients: Instead of simply silently dropping them, as we do now, we
     could drop messages and signal the client that they missed some data ("M
     dropped messages since your last poll").

     This option is probably the best balance between work, API change, and
     benefit, and has a nice incidental effect that it would be easier to debug
     subscriptions from the command-line, like the other RPC methods.

   - Switch to gRPC: Preserves a persistent connection and gives us a more
     efficient binary wire format (protobuf), at the cost of much more work for
     clients and harder debugging. This may be the best option if performance
     and server load are our top concerns.

     Given that we are currently using JSON-RPC, however, I'm not convinced the
     costs of encoding and sending messages on the event subscription channel
     are the limiting factor on subscription efficiency, however.

3. **Delegate event subscriptions to a proxy.** Give responsibility for
   managing event subscription to a proxy that runs separately from the node,
   and switch the node to push events to the proxy (like a webhook) instead of
   serving subscribers directly.  This is more work for the operator (another
   process to configure and run) but may scale better for big networks.

   I mention this option for completeness, but making this change would be a
   fairly substantial project.  If we want to consider shifting responsibility
   for event subscription outside the node anyway, we should probably be more
   systematic about it. For a more principled approach, see point (4) below.

4. **Move event subscription downstream of indexing.** We are already planning
   to give applications more control over event indexing. By extension, we
   might allow the application to also control how events are filtered,
   queried, and subscribed. Having the application control these concerns,
   rather than the node, might make life easier for developers building UI and
   tools for that application.

   This is a much larger change, so I don't think it is likely to be practical
   in the near-term, but it's worth considering as a broader option. Some of
   the existing code for filtering and selection could be made more reusable,
   so applications would not need to reinvent everything.


## References

- [Tendermint RPC service][rpc-service]
- [Tendermint RPC routes][rpc-methods]
- [Discussion of the event system][events]
- [Discussion about RPC transport options][rpc-transport] (from RFC 002)
- [RFC 6455: The websocket protocol][ws]
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)

[rpc-service]: https://docs.tendermint.com/v0.34/rpc/
[rpc-methods]: https://github.com/tendermint/tendermint/blob/main/internal/rpc/core/routes.go#L12
[events]: ./rfc-005-event-system.rst
[rpc-transport]: ./rfc-002-ipc-ecosystem.md#rpc-transport
[ws]: https://datatracker.ietf.org/doc/html/rfc6455
[json-response]: https://www.jsonrpc.org/specification#response_object
[json-notify]: https://www.jsonrpc.org/specification#notification
[issue2949]: https://github.com/tendermint/tendermint/issues/2949
