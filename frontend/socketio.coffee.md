Client interaction
------------------

Socket.io server example

    socketio_frontend = (io,build_client_policy,client_join) ->

      io.on 'connection', (socket,params) ->

request contains headers such as Cookie, handshake contains headers, query, … see https://socket.io/docs/server-api/#socket-handshake

        client_policy = build_client_policy socket.request, socket.handshake

        client_disconnect = most.fromEvent 'disconnect', socket

        client_source = most
          .fromEvent 'msg', socket
          .until client_disconnect
        client_source.__name = "socket.io-client_source-#{socket.id}"

        client_sink = client_join client_source, client_policy
        client_sink.__name = "socket.io-client_sink-#{socket.id}"
        console.log 'client_sink', client_sink

        ###
        client_sink
        .forEach (msg) ->
          console.log 'msg', msg
        ###

        client_sink
        .until client_disconnect
        .forEach (msg) ->
          socket.emit 'msg', msg

        return

      return

    module.exports = socketio_frontend
    most = require 'most'
