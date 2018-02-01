    has_key = (desired_key) ->
      ({key}) ->
        deepEqual key, desired_key

    module.exports = has_key
