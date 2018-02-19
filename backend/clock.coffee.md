This really is a toy for demonstration purposes.

    clock_backend = (period) ->
      (sources) ->
        most
          .constant {op:NOTIFY,key:'clock'}, most.periodic period

    most = require 'most'
    {NOTIFY} = require 'red-rings/operations'
    module.exports = clock_backend
