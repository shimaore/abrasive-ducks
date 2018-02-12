This really is a toy for demonstration purposes.

    clock_backend = (period) ->
      (clients_sources) ->
        # subscriptions = subscriptions_filterer clients_sources.filter (msg) -> msg.key is 'clock'
        most
          .constant {key:'clock'}, most.periodic period

    most = require 'most'
    module.exports = clock_backend
