CouchDB view as a stream of `row`
---------------

    view_as_stream = (db_uri,app,view,key,endkey) ->

      uri = "#{db_uri}/_design/#{app}/_view/#{view}"

      params =
        reduce: false

      if key?
        if endkey?
          params.startkey = JSON.stringify key
          params.endkey = JSON.stringify endkey
        else
          params.key = JSON.stringify key

      n = oboe_stream_request url:uri,qs:params

      r = oboe_stream 'rows.*', n
      n.node 'rows.*', -> oboe.drop
      r

    oboe_stream = require 'oboe-as-stream'
    oboe = require 'oboe'
    request = require 'request'
    oboe_stream_request = (require 'oboe-stream-request') oboe, request
    module.exports = view_as_stream
