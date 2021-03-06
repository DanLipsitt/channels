Reference
=========

.. _ref-consumers:

Consumers
---------

When you configure channel routing, Channels expects the object assigned to
a channel to be a callable that takes exactly one positional argument, here
called ``message``. This is a :ref:`message object <ref-message>`.

Consumers are not expected to return anything, and if they do, it will be
ignored. They may raise ``channels.exceptions.ConsumeLater`` to re-insert
their current message at the back of the channel it was on, but be aware you
can only do this so many time (10 by default) until the message is dropped
to avoid deadlocking.


.. _ref-message:

Message
-------

Message objects are what consumers get passed as their only argument. They
encapsulate the basic :doc:`ASGI <asgi>` message, which is a ``dict``, with
extra information. They have the following attributes:

* ``content``: The actual message content, as a dict. See the
  :doc:`ASGI spec <asgi>` or protocol message definition document for how this
  is structured.

* ``channel``: A :ref:`Channel <ref-channel>` object, representing the channel
  this message was received on. Useful if one consumer handles multiple channels.

* ``reply_channel``: A :ref:`Channel <ref-channel>` object, representing the
  unique reply channel for this message, or ``None`` if there isn't one.

* ``channel_layer``: A :ref:`ChannelLayer <ref-channellayer>` object,
  representing the underlying channel layer this was received on. This can be
  useful in projects that have more than one layer to identify where to send
  messages the consumer generates (you can pass it to the constructor of
  :ref:`Channel <ref-channel>` or :ref:`Group <ref-group>`)


.. _ref-channel:

Channel
-------

Channel objects are a simple abstraction around ASGI channels, which by default
are unicode strings. The constructor looks like this::

    channels.Channel(name, alias=DEFAULT_CHANNEL_LAYER, channel_layer=None)

Normally, you'll just call ``Channel("my.channel.name")`` and it'll make the
right thing, but if you're in a project with multiple channel layers set up,
you can pass in either the layer alias or the layer object and it'll send
onto that one instead. They have the following attributes:

* ``name``: The unicode string representing the channel name.

* ``channel_layer``: A :ref:`ChannelLayer <ref-channellayer>` object,
  representing the underlying channel layer to send messages on.

* ``send(content)``: Sends the ``dict`` provided as *content* over the channel.
  The content should conform to the relevant ASGI spec or protocol definition.


.. _ref-group:

Group
-----

Groups represent the underlying :doc:`ASGI <asgi>` group concept in an
object-oriented way. The constructor looks like this::

    channels.Group(name, alias=DEFAULT_CHANNEL_LAYER, channel_layer=None)

Like :ref:`Channel <ref-channel>`, you would usually just pass a ``name``, but
can pass a layer alias or object if you want to send on a non-default one.
They have the following attributes:

* ``name``: The unicode string representing the group name.

* ``channel_layer``: A :ref:`ChannelLayer <ref-channellayer>` object,
  representing the underlying channel layer to send messages on.

* ``send(content)``: Sends the ``dict`` provided as *content* to all
  members of the group.

* ``add(channel)``: Adds the given channel (as either a :ref:`Channel <ref-channel>`
  object or a unicode string name) to the group. If the channel is already in
  the group, does nothing.

* ``discard(channel)``: Removes the given channel (as either a
  :ref:`Channel <ref-channel>` object or a unicode string name) from the group,
  if it's in the group. Does nothing otherwise.


.. _ref-channellayer:

Channel Layer
-------------

These are a wrapper around the underlying :doc:`ASGI <asgi>` channel layers
that supplies a routing system that maps channels to consumers, as well as
aliases to help distinguish different layers in a project with multiple layers.

You shouldn't make these directly; instead, get them by alias (``default`` is
the default alias)::

    from channels import channel_layers
    layer = channel_layers["default"]

They have the following attributes:

* ``alias``: The alias of this layer.

* ``registry``: An object which represents the layer's mapping of channels
  to consumers. Has the following attributes:

  * ``add_consumer(consumer, channels)``: Registers a :ref:`consumer <ref-consumers>`
    to handle all channels passed in. ``channels`` should be an iterable of
    unicode string names.

  * ``consumer_for_channel(channel)``: Takes a unicode channel name and returns
    either a :ref:`consumer <ref-consumers>`, or None, if no consumer is registered.

  * ``all_channel_names()``: Returns a list of all channel names this layer has
    routed to a consumer. Used by the worker threads to work out what channels
    to listen on.


.. _ref-asgirequest:

AsgiRequest
-----------

This is a subclass of ``django.http.HttpRequest`` that provides decoding from
ASGI requests, and a few extra methods for ASGI-specific info. The constructor is::

    channels.handler.AsgiRequest(message)

``message`` must be an :doc:`ASGI <asgi>` ``http.request`` format message.

Additional attributes are:

* ``reply_channel``, a :ref:`Channel <ref-channel>` object that represents the
  ``http.response.?`` reply channel for this request.

* ``message``, the raw ASGI message passed in the constructor.


.. _ref-asgihandler:

AsgiHandler
-----------

This is a class in ``channels.handler`` that's designed to handle the workflow
of HTTP requests via ASGI messages. You likely don't need to interact with it
directly, but there are two useful ways you can call it:

* ``AsgiHandler(message)`` will process the message through the Django view
  layer and yield one or more response messages to send back to the client,
  encoded from the Django ``HttpResponse``.

* ``encode_response(response)`` is a classmethod that can be called with a
  Django ``HttpResponse`` and will yield one or more ASGI messages that are
  the encoded response.
