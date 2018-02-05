Client interaction
------------------

Socket.io server example

    socketio_frontend = (io,build_client_policy) ->

      io.on 'connection', (socket,params) ->

request contains headers such as Cookie, handshake contains headers, query, … see https://socket.io/docs/server-api/#socket-handshake

        client_policy = build_client_policy socket.request, socket.handshake

        client_source = most
          .fromEvent 'msg', socket
          .until most.fromEvent 'disconnect', socket

        client_sink = join client_source, client_policy

        client_sink.forEach (msg) -> socket.emit 'msg', msg

        return

      return
