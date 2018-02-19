Datastream routing engine for CCNQ4
-----------------------------------

This module implements a core message/datastream routing engine for CCNQ4 (based on `most.js` streams). It provides opinionated options for message-passing authorization.

Motivation
----------

This server intends to be a replacement for the `spicy-action` server and provide access to the system datastreams in the following way:

- A datastream consists of messages of the following four types:
  - `UPDATE document(id,rev)`
  - `SUBSCRIBE TO document(key)`
  - `UNSUBSCRIBE FROM document(key)`
  - `NOTIFY (key)`
- The concept of `document` is very wide and encloses:
  - documents in the CouchDB sense (which might be stored in CouchDB or not)
  - notification documents (such as those sent by huge-play, ccnq4-opensips, …)
  - operational documents (such as those received and sent by huge-play to trigger calls, …)
  - list of document, cross-references, views output, …
  - aggregate documents (currently provided by CouchDB views)
  - other documents outside of this specification.
- All documents are expressed as JSON structures. For example, even though some operations refer to a `Set` datatype, it is transported as a JSON array. (This also allows easy mapping to other transport structures such as msgpack etc.)
- The security model is responsible for providing a transform (`.map`,`.filter`) for streams coming from a frontend or backend. (Different transforms are applied based on the authenticated user, …)

Although this module applies a stream of per-client policies, it does not define how these policies are created or managed.

Message format
--------------

The message format is built so that data coming from CouchDB views can be easily exported.

- `.op` — `update`, `subscribe`, `unsubscribe`, `notify` — these are defined in the `red-rings` project.
- `.key` — (required for `subscribe`, `unsubscribe`, `notify`) the filtering key

The above two fieds really are the only ones required for this module to works. The following conventions are adopted in conjunction with the red-rings modules, and follow closely from the CCNQ4 and CouchDB specifications:

- `.id` (required for `update`, may be present in `notify`) — document unique ID, in the form `<type>:<key>`, where `type` must be one of the predefined types
- `.rev` (required for `update`) — document sequence; treated essentially as opaque, allows recipients to keep track of the last (or last N) revisions for a document and know this one is different.
- `.value` (possible for `notify`) — the value emitted by a view, all-docs, etc.
- `.doc` (possible for `update` and `notify`) — this might be a document's content (for `notify` messages), or a shortcut for `.operations`, where each field indicates a basic `set` operation (for `update` messages);
- `.operations` (only in `update`) — operations on the existing revision of a document.

The difference between `.id` and `.key`, and between `.doc` and `.value`, are references to the CouchDB view system and hold the same semantics.

The `.operations` field is an addition of this specification (see below), while the interpretation of `.doc` as a set of updates (instead of a value) is also an addition of this specification.

Other fields might be defined, although best practices would suggest any `.id`-related date should be provided in the `.doc` object. (In other words, fields at the root of the message object should be metadata about the message itself, while payload content is carried in `.doc` in most cases, except for complex transformations carried in `.operations`.)

(In NodeRED for example the `.doc` / `.value` convention is equivalent to the `.payload` convention. `.doc` is kept because of the CouchDB background but could easily be translated into `.payload` at the boundaries of a NodeRED system.)

Update operations
-----------------

The "override old values with new values" semantic doesn't work because the client won't have access to the old record in its entirety, the policy won't have access to the old record either, etc.

Therefor we need to specify operational semantics rather than data. This means for example:

- create a document (and set some values)
- delete a document
- modify values:
  - replace a field's value with a new one
  - delete a field
  - append (to array)
  - add (to set, on deepEqual)
  - remove (from set, on deepEqual)
  - splice from array (using index)
  - remove from object (using key)

Notice: all values are JSON values (string, number, object, array, `true`, `false`, `null`).

So basically we're looking for some kind of language to use to specifiy those operations and combine them (imperatively and in a way that can be parsed by the `client_policy` code easily).

Some references: [JSONpath](https://www.npmjs.com/package/jsonpath), [JSONata](http://docs.jsonata.org/programming.html)
But both offer something much closer to Turing-complete, which means the descriptions can't be analyzed by a security policy.

Maybe something along the lines of (again) `flat-ornaments`, with operations:

```
["set",json-path,value] — set a value (notice that `null` is a valid value)
["set",json-path] — remove at json-path
["union",json-path,value...] — treat array at json-path as a Set (Redis' `sadd`) using deepEqual
["minus",json-path,value...] — treat array at json-path as a Set (Redis' `srem`) using deepEqual
["append",json-path,value] — append to array
["splice",json-path,start,count,value...] — same as existing 'splice'
```

Note: creation is done by not specifying a revision.
Note: deletion is done by setting `_deleted` to `true` using any of the `set` operations, or setting the `deleted` meta-tag.
Note: json-path may be specified as a string, or an array of strings.

Using the above operations, the semantics of the `.doc` field in an `UPDATE` operation is no longer "this field contains the new value of the document" but "this field contains a list of `set` operations". With proper policies this is actually transparent to a consumer of the data for most operations (the policy will provide only the proper fields in `NOTIFY` messages, and allow modifications of the proper fields in `UPDATE` messages).

The final implementation is located in `util/json-path`, which is used (e.g.) by `backend/couchdb` to apply a changeset to a CouchDB record.

Attachments
-----------

For binary, backends should automatically map attachments with two URIs:
- one to upload new content
- one to download the content

For example In CouchDB this is done using
```
_attachments:
  "name.txt":
    content_ype: "text/plain"
    (data: base64-encoded-data)
    (length: …)
    (digest: "…")
    (revpos: …)
    stub: true
    download_uri: "…" # e.g. Ceph/AWS `getSignedUrl` for `getObject`
    upload_uri: "…" # e.g. Ceph/AWS `getSignedUrl` for `putObject`, or `createPresignedPost`
```

The policy can then filter out whatever should be made available to the client (e.g. only `download_uri`, no URI, etc.).

Typically a backend could (should?) also include some kind of proxy for its database (e.g. CouchDB with Proxy Authentication) if the database doesn't support cross-upload/download (similarly to what Ceph/AWS do with the signed URLs).

See also [how CouchDB does multiple attachment upload](https://wiki.apache.org/couchdb/HTTP_Document_API#Multiple_Attachments)?
