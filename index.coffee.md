Core server
-----------

    most = require 'most'
    {EventEmitter2} = require 'eventemitter2'

    core = ->

      sources_bus = new EventEmitter2

### Master bus (from the clients)

The (unfiltered) datastream from the client is `client_source`.

```
client_source → (authorize) → client_authorized_source → (join) → client_sources
```

`clients_authorized_sources` is a higher-order stream of all client sources

      clients_authorized_sources = most.fromEvent 'client join', sources_bus

`clients_sources` is the master stream containing all requests from all client sources

      clients_sources = clients_authorized_sources
        .join().multicast()

      clients_sources_subscriptions = clients_sources
        .thru subscriptions_filterer
        .multicast()

Each back-end proxy will 'observe' the `clients_sources` stream and apply its changes.

### Master bus (from the backends)

      backend_sources = most.fromEvent 'backend join', sources_bus

      master_source = backend_sources.join().multicast()

### Client join

The client-policy is a stream of policy functions (maybe just one) that apply to that client-source.
The client-policy is applied both to messages from the client and to messages towards the client.

      client_join = (client_source,client_policy) ->

        apply_client_policy_to = (source,why) ->
          client_policy
          .map (policy) -> policy source
          .switch()
          .multicast()

        client_authorized_source = apply_client_policy_to client_source, 'client_authorized_source'

        sources_bus.emit 'client join', client_authorized_source

#### Client Sink

The datastream sent to the client is `client_sink`.

```
master_source → (subscriptions) → subscribed → (authorize) → client_sink
```

To build the client sink we need access to:
- the client subscriptions
- the client policy

(Note that for now we work with a naive implementation, we do
the heavy-lifting of the subscriptions from a single master stream.)

The `client_subscriptions` stream is built from the (authorized) subscriptions received from the client.
It contains filtering functions that reflect the state of subscriptions at the time they are emitted.

        client_subscriptions = client_authorized_source
          .thru subscriptions_filterer

`client_policy` is a stream of policy functions (for the given client)
`client_subscriptions` is a stream of subscription functions, which is mapped to a stream of streams.

        subscribed = client_subscriptions
          .map (subscription) -> subscription master_source
          .switch()

        client_sink = apply_client_policy_to subscribed, 'client_sink'

        return client_sink

### Backend join

      backend_join = (fn) ->

        backend_source = fn clients_sources, clients_sources_subscriptions

        sources_bus.emit 'backend join', backend_source

        return

### Pump the master-source

The current code (with subscriptions as written) does not trigger a start of the master-source (which probably means that the code above or the code in subscriptions-filtere is improperly written).
We're initiating it manually here.

      master_source
      .tap (v) -> console.log 'master_source', v
      .drain()
      .then ->
        console.log 'Master Source terminated.'
      .catch (error) ->
        console.log 'Master Source failed', error

      return {client_join,backend_join}

    module.exports = core
    subscriptions_filterer = require './util/subscriptions-filterer'
