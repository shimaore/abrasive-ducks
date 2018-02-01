CouchDB view as a stream of `row`
---------------

    view_as_stream = (db_uri,app,view,key,endkey) ->
      view_uri_as_stream view_as_uri db_uri, app, view, key, endkey

    view_uri_as_stream = (url) ->
      r = oboe_stream 'rows.*', n = oboe {url}
      n.node 'rows.*', oboe.drop
      r

    view_as_uri = (db_uri,app,view,key,endkey) ->
      params =
        reduce: false

      if key?
        if endkey?
          params.startkey = JSON.stringify key
          params.endkey = JSON.stringify endkey
        else
          params.key = JSON.stringify key

      "#{db_uri}/_design/#{app}/_view/#{view}?#{qs.stringify params}"

    module.exports = {view_as_stream,view_as_uri,view_uri_as_stream}
