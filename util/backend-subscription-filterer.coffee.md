Backend subscription functions
------------------------------

This is a helper for backend modules (especially ones that only implement transports, and need to handle remote backends' subscriptions).

    backend_subscriptions_filterer = (source) ->

      subscriptions_set source
      .map (set) ->
        (stream) ->
          stream
          .filter (msg) ->
            key = Key msg
            return false unless key?

By default the set is empty, and all keys are forwarded.

            return true if set.size is 0

The backend might subscribe to indidual keys.

            return true if set.has key

The backend might also subscribe to `<type>:*`, which forwards all keys starting with `<type>:` (types can only contain letters, digits, underscore, or dash).

            type = key.match(/^([\w-]+):/)?[1]
            type? and set.has "#{type}:*"

If at least one type or individual key is subscribed to, then only messages with keys matching the subscriptions are forwarded.

          .multicast()

    module.exports = backend_subscriptions_filterer

    {Key} = require 'abrasive-ducks-transducers'
    subscriptions_set = require './subscriptions-set.coffee.md'
