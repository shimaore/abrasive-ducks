    subscriptions_set = (source) ->

      subscriptions = Immutable.Set()

      sub = source.filter operation SUBSCRIBE
      unsub = source.filter operation UNSUBSCRIBE

      adder = (stream) ->
        stream
        .filter not_null
        .tap (key) -> subscriptions = subscriptions.add key
        .map -> subscriptions

      deleter = (stream) ->
        stream
        .filter not_null
        .tap (key) -> subscriptions = subscriptions.delete key
        .map -> subscriptions

      streams = [
        adder sub.map Key
        deleter unsub.map Key
      ]

      most.mergeArray streams

    module.exports = subscriptions_set

    most = require 'most'
    Immutable = require 'immutable'
    {operation,not_null,Key} = require 'abrasive-ducks-transducers'
    {SUBSCRIBE,UNSUBSCRIBE} = require 'red-rings/operations'
