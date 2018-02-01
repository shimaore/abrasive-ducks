Datastream server for CCNQ4
---------------------------

This server intends to be a replacement for the `spicy-action` server and provide access to the system datastreams in the following way:
- A datastream from a client might suggest operations:
  - `UPDATE document(id,rev) WITH value`
  - `UPDATE field IN document(id,rev) WITH value`
  - `SUBSCRIBE TO document(key)`
  - `SUBSCRIBE TO field IN document(key)`
- A datastream from the server provides `document(id,rev,fields)`.
- The concept of `document` is very large and encloses:
  - documents in the CouchDB sense (which might be stored in CouchDB or not)
  - notification documents (such as those sent by huge-play, ccnq4-opensips, …)
  - operational documents (such as those received and sent by huge-play to trigger calls, …)
  - aggregate documents (currently provided by CouchDB views)
  - other documents outside of this specification.
- The security model is responsible for:
  - providing a transform (`.map`,`.filter`) for streams coming from a client, before they are submitted to back-ends;
  - providing a transform (`.map`,`.filter`) for streams coming from the back-ends, before they are sent to the client.
