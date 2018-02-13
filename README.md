Datastream server for CCNQ4
---------------------------


Motivation
----------

This server intends to be a replacement for the `spicy-action` server and provide access to the system datastreams in the following way:

- A datastream from a client might suggest operations:
  - `UPDATE document(id,rev)`
  - `SUBSCRIBE TO document(key)`
  - `UNSUBSCRIBE FROM document(key)`
- A datastream from the server provides `document(id,rev,fields)`.
- The concept of `document` is very wide and encloses:
  - documents in the CouchDB sense (which might be stored in CouchDB or not)
  - notification documents (such as those sent by huge-play, ccnq4-opensips, …)
  - operational documents (such as those received and sent by huge-play to trigger calls, …)
  - list of document, cross-references, views output, …
  - aggregate documents (currently provided by CouchDB views)
  - other documents outside of this specification.
- All documents are expressed as JSON structures.
- The security model is responsible for:
  - providing a transform (`.map`,`.filter`) for streams coming from a client, before they are submitted to back-ends;
  - providing a transform (`.map`,`.filter`) for streams coming from the back-ends, before they are sent to the client.

Message format
--------------

The message format is built so that data coming from CouchDB views can be easily exported.

- `.operation` — `update`, `subscribe`, `unsubscribe`
- `.key` — (required for `subscribe`, `unsubscribe`) the filtering key
- `.id` (required) — document unique ID, in the form `<type>:<key>`, where `type` must be one of the predefined types
- `.rev` (required) — document sequence; treated essentially as opaque, allows recipients to keep track of the last (or last N) revisions for a document and know this one is different
- `.doc` — content / set of basic `set` operations
- `.operations` — operations on the existing revision of a document (only in `UPDATE` messages coming from a client)

Other fields might be defined, although best practices would suggest any `.id`-related field be provided in the `.doc` object. (In other words, fields at the root of the message object should be metadata about the message itself, while payload content is carried in `.doc` in most cases, except for complex transformations carried in `.operations`.)

(In NodeRED for example the `.doc` convention is equivalent to the `.payload` convention. `.doc` is kept because of the CouchDB background but could easily be translated into `.payload` at the boundaries of a NodeRED system.)

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

Notice: values are JSON values (string, number, object, array, `true`, `false`, `null`).

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

Typically a back-end could also include some kind of proxy for its database (e.g. CouchDB with Proxy Authentication) if the database doesn't support cross-upload/download (similarly to what Ceph/AWS do with the signed URLs).

See also [how CouchDB does multiple attachment upload](https://wiki.apache.org/couchdb/HTTP_Document_API#Multiple_Attachments)?
