CouchDB back-end
----------------

    couchdb_backend = (db,db_uri,all_map,app,view,clients_sources) ->

      semantic =
        create: ({id,doc}) ->
          doc._id = id
          db.put doc

        update: (message) ->
          db
          .get message.id, rev: message.rev
          .then (doc) ->
            for name, value of message.doc
              doc[name] = value
            for path, cmd of message.operations
              [operation,args...] = cmd
              return unless json_path doc, path, operation, args

            db.put doc

        delete: (message) ->
          db.delete _id: message.id, _rev: message.rev

For `update` events it will update the document.

FIXME: figure out the pattern to push in bulk

The semantic really is 'UPDATE', not 'OVERWRITE'
  → we could either do bulk-get / bulk-put (aka `_all_docs` and `_bulk_docs`),
  → or use an [update function](http://docs.couchdb.org/en/2.1.1/api/ddoc/render.html#db-design-design-doc-update-update-name)
For now the code (above) does single GET/PUT on updates.

      route = clients_sources
        .filter operation 'update'
        .thru changes_semantic()

      route.create.forEach semantic.create
      route.update.forEach semantic.update
      route.delete.forEach semantic.delete

For `subscribe` events it will GET the document and send it into the `master_source` (property) and add the monitored id to the list of monitored IDs.

      provisioning_changes = all_changes db

      provisioning_subscriptions = subscription_filterer clients_sources

Fetch current value

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
          view_as_stream db_uri, app, view, key
        .map Value

      most.mergeArray [
        provisioning_subscriptions.ap provisioning_changes
        provisioning_values
      ]

    module.exports = couchdb_backend

    changes_view = require '../util/changes-view'
    {view_as_stream} = require '../util/view'
    changes_semantic = require '../util/changes-semantic'
