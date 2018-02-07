Policy
------

As a reminder, the system is based on the concept of messages sent from a client to some backends, and conversely sent from a backend to some clients.

The policy is expressed in terms of actions taken on behalf of the client; it is implemented as a stream of functions (since the policy will evolve over time) that process the messages; that processing happens as-close as possible to the client (i.e. as the first step upon reception of a message from a client, or as the last step before sending a message to a client).

The goal of the policy is to:
- ensure well-formedness of the messages coming from the client
- authorize the operations requested by the clients (typically by dropping requests messages or fields that are not authorized)
- authorize operations towards the clients (typically by dropping messages or fields that are not authorized)

However notice how we're not using the policy as a yes/no (permit/deny) methodology on a message (allow/drop), but as a larger "filtering and mapping" functionality. Some of the resulting outcomes of the decisions implemented by a policy might be: drop the message, modify some fields, remove some fields, add some fields, replace some values by an aggregate, … (As far as I know this is not specified anywhere and is a novel authorization scheme.)

Policy frameworks
-----------------

The concept of policy as defined above is relatively straightforward, but does not define _how_ such a policy might be brought together.

There may actually be many underlying policy frameworks; we offer one framework consistent with the existing CCNQ4 procedures.

### Users

- use of a `…domain….user:` document-type to identify users (authentication is provided by external means and must map back to one such user);
- one user belong to one or more groups (a user might have rights in multiple entities with varying policies, etc.);

### Groups

- groups are sets of users;
- groups are assigned sets of permissions over sets of documents;

### Set of documents

- a "set of documents" may be defined as individual documents (enumerated using their IDs), a set of documents (e.g. all documents within an account) etc.

### Policy management

- granting rights is an operation on documents (just like the rest of the whole thing) and is managed by users inside groups, those groups being given sets of permissions over sets of policy-management documents;
- in other words, there is a set of policies that define how rights/permissions are managed;

### Groups hierarchy

- groups are managed inside a tree;
- the set of permissions for a child group should be a subset of the permissions of the parent group;
- this is enforced by the policy-management framework;

### Permissions sets

In order to facilitate the management of permissions, individual permissions functions are grouped into permissions sets inside the policy-management framework (for example the "Create a line" permission set will probably regroup dozens of individual permissions).

### Permissions

- permissions are filtering and/or mapping functions (over a stream)
- permissions are defined for each given document (message) type; the type is deduced from the form of the message id (`<type>:<key:`) (for example by having each permissions being gated, based on a regex on the `msg.id`)
- in order for the "smaller subset" thing to work, the permissions must be defined in a way such that if a permissions is removed, the amount of information that goes through is smaller;
- in other words the "default" permission would be to remove all fields in a message and drop all messages;
- in other words each permissions should _add_ access to more information;

(This might be too difficult to do properly, especially because I haven't defined yet how permissions are applied to a stream; if the permissions are applied in sequence (`.thru permission1 .thru permission2` etc.) then the semantic above is probably _not_ going to work.)

(Also permissions should be applied to _operations_ on _documents_, not on operations on messages (which are not defined anywhere yet)?)
(Therefor messages coming from backends should be treated as a set of operations, similarly to the semantic defined for documents coming from clients.)

In that later sense the permissions are modifiers over operations sets as defined in messages (but should also be allowed to filter messages altogether, since information can be gathered of the fact that a message was sent, therefor that some state changed that triggered the sending of the message).

For example, if a user (actually a group) is granted permission to know whether/when an endpoint is registered, the permission should allow them to receive the messages, but with all data provided by ccnq4-opensips replaced with the single field `registered` (a boolean). Another permission (more inclusive, more permissive, providing more information) would allow to also know the user-agents; another one would also allow to know the IP addresses the registrations were received from; etc.

So the problematic is:
- given a set of "permissions functions" such as this one, how do we send the messages through them in order to obtain a consistent whole? (probably the best way is to have the functions "build" the resulting document by adding "operations" in its set of operations, the operation set being empty at the beginning)
- 


Algo then:
```
resulting message = empty (= drop), meaning "set of operations = []"
for each permission in the set defined for all groups the user belong to
  if the permission gate (=id regexp) matches the `msg.id`
    add the set of operations built by the permission function to the set of operations for this message
if the set of operations is empty
  drop the message
else
  build the message from the set of operations (i.e. copy the msg.id and .rev, and msg.operations = set of operations)
```

Ooooh. In that sense the set of permissions builds a transform of a set-of-operations to another set-of-operations.

  new-set-of-operations = Union-of (operations returned by permission-functions for the current group and message)

or (maybe better), _each_ permission function could be a transform over a set-of-operations

  new-set-of-operations = set-of-operations.thru perm1 .thru perm2 .thru perm3 …

i.e. set-of-operations is itself treated as a stream and the permission functions are stream transformers for that stream.

However we still have the issue with having proper semantics of combining permission functions; this still needs work.


### Comparison with ABAC

To map to the [ABAC: Attribute-based access control](https://en.wikipedia.org/wiki/Attribute-based_access_control) terminology:
- subject attributes: stored in the document of the user and/or the group
- action attributes: provided in the message from/to the client
- resources attributes: stored in database(s)
- contextual attributes

However remember we are _not_ implementing ABAC, but a larger policy structure that allows for modification of messages as they go through.



User → Group
Group x Documents → Operations
