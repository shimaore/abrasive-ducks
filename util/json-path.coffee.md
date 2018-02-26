JSON-path manipulators
----------------------

This library is meant to be used inside Node.js or CouchDB `_update` functions etc. to update a document using a series of commands.


    FIXME  Use Immutable instead
    deepEqual = require 'windy-moon/deepEqual'

    operations =

Assign / delete

      set: (doc,name,args) ->
        switch args.length
          when 1
            value = args[0]
            doc[name] = value
            return true
          when 0
            delete doc[name]
            return true
          else
            return false

Set union

      union: (doc,name,args) ->
        return false unless name of doc
        a = doc[name]
        return false unless Array.isArray a
        args.forEach (x) ->
          a.push(x) unless a.find (y) -> deepEqual x,y
        return true

Set difference

      minus: (doc,name,args) ->
        return false unless name of doc
        a = doc[name]
        return false unless Array.isArray a
        doc[name] = a.filter (x) -> not args.find (y) -> deepEqual x,y
        return true

Append to an array

      append: (doc,name,args) ->
        return false unless name of doc
        a = doc[name]
        return false unless Array.isArray a
        doc[name] = a.concat args
        return true

Splice an array

      splice: (doc,name,args) ->
        return false unless name of doc
        a = doc[name]
        return false unless Array.isArray a
        return false unless 0 < args.length <= 3
        [start,count,rep...] = args
        return false unless typeof start is 'number'
        return false unless typeof count is 'number'
        return false unless Array.isArray rep
        a.splice start,count,rep...
        return true

    FIXME move json-path to red-rings-path or something like that
    FIXME export this function to module
    json_path_to_strings = (path) ->
      reg = /([^\.\"\[\]]+)\.?|\"([^"]+)\"\.?|\[([^\]]+)\]\.?/y
      reg.lastIndex = 0
      while $ = path.match reg
        $[1] ? $[2] ? $[3]

    FIXME export this function to module
    default_forbidden_names = new Set [
      '_id'
      '_rev'
      '_deleted'
    ]

    json_path = (doc,path,operation,args,forbidden_names = default_forbidden_names) ->
      op = operations[operation]
      unless op?
        console.error "Unknwon operation #{operation}"
        return false

      unless Array.isArray args
        console.error "Missing arguments"
        return false

      FIXME Use Immutable and path is always a List (which means the original `.operations` should be able to use List as keys and therefor be reworked) (and therefor that the policies will handle operations one-by-one to establish their semantics, and that a single msg may specify ambiguous changes… but I guess that was already the case)
      if Array.isArray path
        strings = path
      else
        strings = json_path_to_strings path

      l = strings.length

      for name, i in strings
        return false unless typeof name is 'string'
        return false if forbidden_names.has name

        if i is l-1
          if op doc, name, args
            return strings
          else
            return false

        if name of doc
          doc = doc[name]
        else
          doc = doc[name] = {}

      return false

    assert = require 'assert'
    assert.deepEqual ['hello','world'], json_path_to_strings 'hello.world'
    assert.deepEqual ['hello','world'], json_path_to_strings 'hello[world]'
    assert.deepEqual ['hello','world'], json_path_to_strings 'hello."world"'
    assert.deepEqual ['hello','world','3'], json_path_to_strings 'hello."world".3'
    assert.deepEqual ['hello','[world]'], json_path_to_strings 'hello."[world]"'
    assert.deepEqual ['hello','"world"'], json_path_to_strings 'hello.["world"]'
    assert.deepEqual ['àéï','world','ùù'], json_path_to_strings 'àéï"world"[ùù]'

    test = (doc,path,op,args...) ->
      json_path doc,path,op,args
      doc

    assert.deepEqual {a:3}, test {}, 'a', 'set', 3
    assert.deepEqual {a:[]}, test {}, 'a', 'set', []
    assert.deepEqual {a:[3]}, test {a:[]}, 'a', 'append', 3
    assert.deepEqual {a:[3,5]}, test {a:[3]}, 'a', 'union', 5
    assert.deepEqual {a:[3,5]}, test {a:[3,5]}, 'a', 'minus', 7
    assert.deepEqual {a:[3,5,{b:4}]}, test {a:[3,5]}, 'a', 'union', b:4
    assert.deepEqual {a:[3,5]}, test {a:[3,5,{b:4}]}, 'a', 'minus', b:4
    assert.deepEqual {a:[3,true]}, test {a:[3,5,{b:4}]}, 'a', 'splice', 1, 2, true
    assert.deepEqual {a:4}, test {}, 'a', 'set', 4
    assert.deepEqual {a:b:4}, test {}, 'a.b', 'set', 4
    assert.deepEqual {a:b:c:4}, test {}, 'a.b.c', 'set', 4
    assert.deepEqual {a:{}}, test {a:b:c:4}, 'a.b', 'set'
    assert.deepEqual {}, test {a:b:c:4}, 'a', 'set'
    assert.deepEqual {_id:12}, test {_id:12}, '[_id]', 'set', 42
    assert.deepEqual {_id:12}, test {_id:12}, '"_deleted"', 'set', true
