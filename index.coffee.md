Core server
-----------

    most = require 'most'
    {EventEmitter2} = require 'eventemitter2'

    deny_all = (s) -> s.filter -> false
    allow_all = (s) -> s

    apply_policy = (remote_policy,source) ->
      remote_policy
      .map (policy) -> policy source
      .switch()
      .multicast()

    core = (fromJS,limit = most.never()) ->

      sources_bus = new EventEmitter2

### Master bus (from the clients)

The (unfiltered) datastream from the client is the `source`.

```
source → (authorize) → authorized_source → (join) → master_source
```

`authorized_sources` is a higher-order stream of all sources as they appear

      authorized_sources = most.fromEvent 'join', sources_bus
        .until limit

`sources` is the master stream containing all requests from all sources

      master_source = authorized_sources
        .join().multicast()

### client join

The remote-policy is a stream of policy functions (maybe just one) that apply to that source.
The remote-policy is applied both to messages from the client and to messages towards the client.

      frontend_join = (remote_source,remote_policy = most.just deny_all) ->

        remote_source = remote_source.map fromJS

        authorized_source = apply_policy remote_policy, remote_source

        sources_bus.emit 'join', authorized_source

#### Sink

The datastream sent to the client is the `sink`.

```
master_source → (subscriptions) → subscribed → (authorize) → sink
```

To build the client sink we need access to:
- the client subscriptions
- the client policy

(Note that for now we work with a naive implementation, we do
the heavy-lifting of the subscriptions from a single master stream.)

The `subscriptions` stream is built from the (authorized) subscriptions received from the client.
It contains filtering functions that reflect the state of subscriptions at the time they are emitted.

        subscriptions = authorized_source
          .thru subscriptions_filterer

`remote_policy` is a stream of policy functions (for the given client)
`subscriptions` is a stream of subscription functions, which is mapped to a stream of streams.

        subscribed = subscriptions
          .map (subscription) -> subscription master_source
          .switch()

        remote_sink = apply_policy remote_policy, subscribed

        return remote_sink

### Backend join

`fn = (sink) -> source` (where sink is "from client" and source is "towards client")

      backend_join = (fn,remote_policy = most.just allow_all) ->

        remote_sink = apply_policy remote_policy, master_source

        remote_source = fn remote_sink

        remote_source = remote_source.map fromJS

        authorized_source = apply_policy remote_policy, remote_source

        sources_bus.emit 'join', authorized_source
        return

### Pump the master-source

The current code (with subscriptions as written) does not trigger a start of the master-source (which probably means that the code above or the code in subscriptions-filtere is improperly written).
We're initiating it manually here.

      master_source
      .drain()
      .then ->
        console.log 'Master Source terminated.'
      .catch (error) ->
        console.log 'Master Source failed', error

      return {frontend_join,backend_join}

    module.exports = core
    subscriptions_filterer = require './util/subscriptions-filterer'
