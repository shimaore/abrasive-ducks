Subscription functions
----------------------

    subscriptions_set = (source) ->

      subscriptions = new Set()

      sub = source.filter operation 'subscribe'
      unsub = source.filter operation 'unsubscribe'

      adder = (stream) ->
        stream
        .filter not_null
        .tap (key) -> subscriptions.add key
        .map -> subscriptions

      deleter = (stream) ->
        stream
        .filter not_null
        .tap (key) -> subscriptions.delete key
        .map -> subscriptions

      streams = [
        adder sub.map Key
        deleter unsub.map Key
      ]

      most.mergeArray streams

    subscriptions_filterer = (source) ->

      subscriptions_set source
      .map (set) ->
        console.log 'subscription: create filtered stream', set
        (stream) ->
          console.log 'subscription: build stream', set, stream.__name ? stream
          stream
          .filter ({key}) -> key? and set.has key
          .multicast()

    module.exports = subscriptions_filterer

    most = require 'most'
    {operation,not_null,Key} = require './transducers'
