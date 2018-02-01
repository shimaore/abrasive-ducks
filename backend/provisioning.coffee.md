CouchDB back-end
----------------

    view = require 'wandering-country-view/all'
    view_name = 'all'
    {app} = require 'wandering-country-view'

    normalize_account = …

    all_map = (emit) ->
      view {normalize_account,emit}

For example the CouchDB provisioning back-end will listen for `update` and `subscribe` events.

    couchdb_backend = (db_uri,clients_sources) ->

For `update` events it will update the document.

FIXME: figure out the pattern to push in bulk

      clients_sources
      .filter operation 'update'
      .forEach (message) -> db.put message.doc

For `subscribe` events it will GET the document and send it into the `master_source` (property) and add the monitored id to the list of monitored IDs.

      provisioning_changes = all_changes db

      provisioning_subscriptions = subscription_filterer clients_sources

      provisioning_values =
        clients_sources
        .filter operation 'subscribe'
        .map Key
        .filter is_string
        .map (key) -> db.get(key).catch null
        .chain most.fromPromises
        .filter not_null
        .map (doc) ->
          message =
            type: 'document'
            payload: doc

The following compute values and changeset for wandering-country-view/all

      rows = changes_view all_map, provisioning_changes

      provisioning_views_changes =
        rows
        .filter has_key key
        .map Value

      provisioning_views_values =
        clients_sources
        .filter operation 'subscribe'
        .map Key
        .filter is_object
        .chain (key) ->
          view_as_stream key
        .map Value

      most.mergeArray [
        provisioning_subscriptions.ap provisioning_changes
        provisioning_values
      ]

    curry1 = (fn,arg1) ->
      (args...) -> fn arg1, args...

    curry = (fn,args1...) ->
      (args2...) -> fn args1..., args2...

    backend_join curry1 couchdb_backend, db
