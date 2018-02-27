Changes Semantic(s)
===================

Analyzes what operation was realized on the record/document by injecting a message in at most one of `.create`, `.update` or `.delete`.

    changes_semantic = (stream) ->

      source = stream
        .map cleanup
        .filter valid_id
        .multicast()

      create: source.filter(_create).multicast()
      update: source.filter(_update).multicast()
      delete: source.filter(_delete).multicast()

Message examples
----------------

The fields we're concerned with are:
- `id` / `doc._id`: should never start with `_`
- `rev` / `doc._rev`
- `doc`
- `operations`

`id` / `doc._id` is always required if this message is going to be interpreted as a changeset.

    valid_id = (msg) ->
      msg.id? and typeof msg.id is 'string' and msg.id[0] isnt '_'

`rev` / `doc._rev` is required for `update` and `delete`.

    valid_rev = (msg) ->
      msg.rev? and typeof msg.rev is 'string'

The following descriptions indicate first how the message is normalized by the function before being sent.

### Create

Create requires a document field.

    _create = (msg) ->
      not msg.deleted and not msg.rev? and
        msg.had_doc and not msg.operations

Normalized:

```
{
 id: "something"
 doc: { field: value, … }
}
```

Acceptable inputs:

```
{
 doc: { _id: "something", field: value, … }
}
```

### Update

Updates can be expressed with a document (denoting changes) and/or operations.

    _update = (msg) ->
      not msg.deleted and valid_rev(msg) and
        (msg.had_doc or msg.operations)

Normalized:

```
{
 id: "something"
 rev: "my rev"
 doc: { field: value, … }
 operations: {
  "json-path": ["set",value]
  "json-path": ["set"]
  …
 }
}
```

Note: the `doc` field is a shortcut for the `set` operation; the difference with `operations` is that the key is a field name, not a JSON path.

Acceptable inputs:

```
{
 doc: { _id: "something", _rev: "my rev", field: value, … }
 operations: {
  "json-path": ["set",value]
  "json-path": ["set"]
  …
 }
}
```

### Delete

Deletion.

    _delete = (msg) ->
      msg.deleted and valid_rev(msg) and
        not (msg.had_doc or msg.operations)

Normalized:

```
{
 id: "something"
 rev: "my rev"
 deleted: true
}
```

Acceptable inputs:

```
{
 doc: {_id,_rev,_deleted:true}
}
```

Cleanup
-------

The message is cleaned-up before being routed.

    cleanup = (msg) ->

      msg.had_doc = msg.doc?

Re-import the meta-data from the document (this allows clients to only specify `.doc` in most cases).

      msg.doc ?= {}

`.id` and `.doc._id` are equivalent

      msg.id ?= msg.doc._id

`.rev` and `.doc._rev` are equivalent

      msg.rev ?= msg.doc._rev

Deletion might be indicated in the message metadata, in the document's `_deleted` field, or by a operation on the `_deleted` json-path.

      msg.deleted ?= false
      msg.deleted or= msg.doc._deleted

      delete msg.doc._id
      delete msg.doc._rev
      delete msg.doc._deleted

      msg

    module.exports = changes_semantic
