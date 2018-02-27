Subscription functions
----------------------

    subscriptions_filterer = (source) ->

      subscriptions_set source
      .map (set) ->
        (stream) ->
          stream
          .filter (msg) ->
            key = Key msg
            key? and set.has key
          .multicast()

    module.exports = subscriptions_filterer

    {Key} = require 'abrasive-ducks-transducers'
    subscriptions_set = require './subscriptions-set.coffee.md'
