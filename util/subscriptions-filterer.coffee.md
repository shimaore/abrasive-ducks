Subscription functions
----------------------

    subscriptions_filterer = (source) ->

      subscriptions_set source
      .map (set) ->
        (stream) ->
          stream
          .filter ({key}) -> key? and set.has key
          .multicast()

    module.exports = subscriptions_filterer

    subscriptions_set = require './subscriptions-set.coffee.md'
