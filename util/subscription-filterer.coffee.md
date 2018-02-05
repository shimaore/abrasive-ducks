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
        (stream) ->
          stream
            .map Key
            .filter not_null
            .filter (key) -> set.has key

    module.exports = subcriptions_filterer

    most = require 'most'
    {operation,not_null,Key} = require './transducers'
