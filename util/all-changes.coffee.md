Return all changes from a database as a stream of results.
--------

The `db` argument is a PouchDB database.

The resulting stream consists of one `{doc}` record for each change.

    most = require 'most'

    all_changes = (db) ->

Note: we can't use a filter:`_view` nor filter:`_find` here, because these cannot take arguments.

      changes = db.changes
        live: true
        include_docs: true
        since: 'now'

      most
      .fromEvent 'change', changes
      .until most.fromEvent('complete',changes).chain ({results}) -> most.from results
      .until most.fromEvent('error',changes)

    module.exports = all_changes
