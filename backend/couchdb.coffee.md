CouchDB back-end
----------------

TBD / FIXME : attachments
TBD / FIXME : only handle known record types ?

- `db` and `db_uri` refer to the same database.
- `all_map`, `app`, `view` refer to the same view.

    couchdb_backend = (db,db_uri,all_map,app,view) ->

      semantic =

        create: ({id,doc}) ->
          doc._id = id
          db
          .put doc
          .catch -> yes

        update: (message) ->
          db
          .get message.id, rev: message.rev
          .then (doc) ->
            for name, value of message.doc
              doc[name] = value
            for path, cmd of message.operations
              [op,args...] = cmd
              return unless json_path doc, path, op, args

            db.put doc
          .catch -> yes

        delete: (message) ->
          db
          .delete _id: message.id, _rev: message.rev
          .catch -> yes

Document changes

      changes = all_changes db
        .multicast()

Changeset for wandering-country-view/all

      view_changes = changes_view all_map, changes
        .multicast()

The backend return a function that can be provided to `backend_join`.

      (clients_sources,clients_sources_subscriptions) ->

For `update` events it will update the document.

FIXME: figure out the pattern to push in bulk

The semantic really is 'UPDATE', not 'OVERWRITE'
  → we could either do bulk-get / bulk-put (aka `_all_docs` and `_bulk_docs`),
  → or use an [update function](http://docs.couchdb.org/en/2.1.1/api/ddoc/render.html#db-design-design-doc-update-update-name)
For now the code (above) does single GET/PUT on updates.

        route = clients_sources
          .filter operation UPDATE
          .thru changes_semantic

        route.create.forEach semantic.create
        route.update.forEach semantic.update
        route.delete.forEach semantic.delete

For `subscribe` events it will GET the document and send it into the `master_source` (property) and add the monitored id to the list of monitored IDs.

        subscriptions_keys =
          clients_sources
          .filter operation SUBSCRIBE
          .map Key

No action is needed for `unsubscribe` events.

Fetch current value, return a message similar to a row from `_all_docs`.

        values =
          subscriptions_keys
          .filter is_string # Can only retrieve/store string keys
          .map (key) -> db.get(key).catch -> null
          .chain most.fromPromise
          .filter not_null
          .map (doc) ->
            op: NOTIFY
            id: doc._id
            key: doc._id
            value: rev: doc._rev
            doc: doc

Compute values for wandering-country-view/all by querying the server-side view.

        view_values =
          subscriptions_keys
          .chain (key) ->
            view_as_stream db_uri, app, view, key

The output is the combination of:

        most.mergeArray [

- subscriptions to document changes in the database

          clients_sources_subscriptions
            .map (subscription) -> subscription changes
            .switch()

- requested documents in the database

          values

- subscriptions to changes in the view

          clients_sources_subscriptions
            .map (subscription) -> subscription view_changes
            .switch()

- requested entries in the view

          view_values

        ]

    module.exports = couchdb_backend

    changes_view = require '../util/changes-view'
    view_as_stream = require '../util/view'
    changes_semantic = require '../util/changes-semantic'
    all_changes = require '../util/all-changes'
    {operation,Key,is_string,is_object,not_null,has_key} = require '../util/transducers'
    subscriptions_filterer = require '../util/subscriptions-filterer'
    most = require 'most'
    {UPDATE,SUBSCRIBE,UNSUBSCRIBE} = require 'red-rings/operations'
