Subscription functions
----------------------

Since we (currently) do not have a Set that allows for deepEqual on values, we map the keys to strings.

    marker = '\uffff'

    key_to_string = (key) ->
      return null unless key?
      return key if typeof key is 'string'
      marker + JSON.stringify key

    subscriptions_filterer = (source) ->

      subscriptions_set source.map ({op,key}) -> {op,key: key_to_string key}
      .map (set) ->
        (stream) ->
          stream
          .tap ({key}) -> console.log 'filtering', key, set
          .filter ({key}) -> key? and set.has key_to_string key
          .multicast()

    module.exports = subscriptions_filterer

    subscriptions_set = require './subscriptions-set.coffee.md'
