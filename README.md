Event Source Library for Python
===============================

This library implements W3C Draft's on event-source:
 * http://dev.w3.org/html5/eventsource/

It enables a halfduplex communication from server to client, but initiated
by the client, through standard HTTP(S) communication.

Dependances
===========

 - Fairly recent python (tested with 2.7)
 - Fairly recent tornado (tested with 2.2.1)

Usage
=====

 1. Launch the server:
    
    python event_source/event_source_listener.py -P 8888 -i -k 50000

 2. Launch the client:

    python event_source/event_source_client.py 69:69:69:69:69:69 -r 5000

 3. Send requests:

    python event_source/send_request.py 69:69:69:69:69:69 ping "42"
    python event_source/send_request.py 69:69:69:69:69:69 close

Command Line arguments
======================

 - event_source_listener:

    usage: event_source/event_source_listener.py [-h] [-H HOST] [-P PORT] [-d]
                                                [-j] [-k KEEPALIVE] [-i]

    Event Source Listener

    optional arguments:
    -h, --help            show this help message and exit
    -H HOST, --host HOST  Host to bind on
    -P PORT, --port PORT  Port to bind on
    -d, --debug           enables debug output
    -j, --json            to enable JSON Event
    -k KEEPALIVE, --keepalive KEEPALIVE
                            Keepalive timeout
    -i, --id              to generate identifiers

 - event_source_client:

    usage: event_source/event_source_client.py [-h] [-H HOST] [-P PORT] [-d]
                                            [-r RETRY]
                                            token

    Event Source Client

    positional arguments:
    token                 Token to be used for connection

    optional arguments:
    -h, --help            show this help message and exit
    -H HOST, --host HOST  Host to connect to
    -P PORT, --port PORT  Port to be used connection
    -d, --debug           enables debug output
    -r RETRY, --retry RETRY
                            Reconnection timeout

 - send_request:

    usage: event_source/send_request.py [-h] [-H HOST] [-P PORT] [-j]
                                        token action [data]

    Generates event for Event Source Library

    positional arguments:
    token                 Token to be used for connection
    action                Action to send
    data                  Data to be sent

    optional arguments:
    -h, --help            show this help message and exit
    -H HOST, --host HOST  Host to connect to
    -P PORT, --port PORT  Port to be used connection
    -j, --json            Treat data as JSON


Integrate
=========

On the server side, basically all you have to do is to add the following to your code:

    from event_source import event_source_listener

    application = tornado.web.Application([
        (r"/(.*)/(.*)", event_source_listener.EventSourceHandler, 
                                          dict(event_class=EVENT,
                                               keepalive=KEEPALIVE)),
    ])

    application.listen(PORT)
    tornado.ioloop.IOLoop.instance().start()

where:
 - PORT is an integer for the port to bind to
 - KEEPALIVE is an integer for the timeout between two keepalive messages (to protect from disconnections)
 - EVENT is a event_source_listener.Event based class, either one you made or 
    - event_source_listener.StringEvent : Each event gets and resends multiline strings
    - event_source_listener.StringIdEvent : Each event gets and resends multiline strings, with an unique id for each event
    - event_source_listener.JSONEvent : Each event gets and resends JSON valid strings
    - event_source_listener.JSONIdEvent : Each event gets and resends JSON valid string, with an unique id for each event

Extend
======

To extend the behaviour of the event source library, without breaking event_source
definition, the Event based classes implements all processing elements that shall
be done on events. 

There is two abstract classes that defines Event:
 - event_source_listener.Event : defines the constructor of an Event
 - event_source_listener.EventId : defines an always incrementing id handler

here is an example to create a new Event that takes multiline data and join it in a one
line string seperated with semi-colons.

    class OneLineEvent(Event):
        ACTIONS = ["ping",Event.FINISH]

        """Property to enable multiline output of the value"""
        def get_value(self):
            # replace carriage returns by semi-colons
            # this method shall always return a list (even if one value)
            return [";".join([line for line in self._value.split('\n')])]

        value = property(get_value,set_value)

And now, I want to add basic id support to OneLineEvent, in OneLineIdEvent, 
nothing is easier :

    class OneLineIdEvent(OneLineEvent,IdEvent):
        id = property(IdEvent.get_value)

Or if I want the id to be a timestamp:

    import time
    class OneLineTimeStampEvent(OneLineEvent):
        id = property(lambda s: "%f" % (time.time(),))

You can change the behaviour of a few things in a Event-based class:
    - Event.LISTEN contains the GET action to open a connection (per default "poll")
    - Event.FINISH contains the POST action to close a connection (per default "close")
    - Event.RETRY contains the POST action to define the timeout after reconnecting on network disconnection (per default "0", which means disabled)
    - in the Event.ACTIONS list, you define what POST actions are allowed, per default,  only Event.FINISH is allowed. 
    - Event.content_type contains the "content_type" that will be asked for every form (it is not enforced).

To change the way events are generated, you can directly call EventSourceHandler.buffer_event()
to create a new event to be sent. But the post action is best, at least while WSGI can't handle
correctly long polling connections.

EOF
