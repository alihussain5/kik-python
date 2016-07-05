.. _user:

User Guide
==========

Ready to get started? This page gives an overview of how to install and use kik

Installation
------------

You can install the library though pip, either from the command line::

    $ pip install kik

or add the following line to your project's requirements.txt file:

.. parsed-literal::

    kik==\ |version|

Example Bot
-----------

Here is a bot built using Kik's Python API and Flask.

.. code-block:: python
    :linenos:

   from flask import Flask, request, Response

   from kik import KikApi, Configuration
   from kik.messages import messages_from_json, TextMessage, PictureMessage, SuggestedResponseKeyboard, TextResponse, StartChattingMessage

   app = Flask(__name__)
   kik = KikApi('BOT_USERNAME_HERE', 'BOT_API_KEY_HERE')

   kik.set_configuration(Configuration(webhook='WEBHOOK_HERE'))


   @app.route('/incoming', methods=['POST'])
   def incoming():
       if not kik.verify_signature(request.headers.get('X-Kik-Signature'), request.get_data()):
           return Response(status=403)

       messages = messages_from_json(request.json['messages'])

       for message in messages:
           # Check if it's the user's first message
           if isinstance(message, StartChattingMessage):
               user = kik.get_user(message.from_user)

               response_body = u'Hey {}, How are you?'.format(user.first_name)

               response_messages = [
                   TextMessage(
                       to=message.from_user,
                       chat_id=message.chat_id,
                       body=response_body,
                       keyboards=[
                           SuggestedResponseKeyboard(
                               responses=[TextResponse('Good'), TextResponse('Bad')]
                           )]
                   )]

           # Check if the user has sent a text message
           elif isinstance(message, TextMessage):
               user = kik.get_user(message.from_user)

               # Check if this message is part of the suggested responses
               if message.body == 'Good':
                   response_body = 'That\'s Great! :)'
               elif message.body == 'Bad':
                   response_body = 'Oh no! :('
               else:
                   response_body = 'Sorry, I didn\'t quite get that'

               response_messages = [
                   TextMessage(
                       to=message.from_user,
                       chat_id=message.chat_id,
                       body=response_body
                   )]

               # Send the user a response along with their profile picture (function definition is below)
               response_messages += profile_pic_check_messages(user, message)

           # If its not a text message, give them another chance to use the suggested responses
           else:
               response_body = 'Sorry, I didn\'t quite get that'

               response_messages = [
                   TextMessage(
                       to=message.from_user,
                       chat_id=message.chat_id,
                       body=response_body,
                       keyboards=[
                           SuggestedResponseKeyboard(
                               responses=[TextResponse('Good'), TextResponse('Bad')]
                           )]
                   )]

           kik.send_messages(response_messages)

       return Response(status=200)


   #  Function to check if user has a profile picture and returns appropriate messages.
   def profile_pic_check_messages(user, message):
       messages_to_send = []

       profile_picture = user.profile_pic_url

       if profile_picture is not None:
           messages_to_send.append(
               PictureMessage(
                   to=message.from_user,
                   chat_id=message.chat_id,
                   pic_url=profile_picture
               ))

           profile_picture_response = 'Here\'s your profile picture!'
       else:
           profile_picture_response = 'It doesn\'t look like you have a profile picture, you should set one'

       messages_to_send.append(
           TextMessage(
               to=message.from_user,
               chat_id=message.chat_id,
               body=profile_picture_response
           ))

       return messages_to_send


   if __name__ == "__main__":
       app.run(port=8080, host='0.0.0.0', debug=True)

The API Client
--------------

The core of the library is the :class:`KikApi<kik.KikApi>` class, which is used to send requests to the Kik API.
The client needs to be instantiated with your bot's username and API key:

    >>> from kik import KikApi
    >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)

Configuration
-------------

The `Configuration API <https://dev.kik.com/#/docs/messaging#configuration>`_ can be accessed through two
functions.

:func:`KikApi.get_configuration<kik.KikApi.get_configuration>` retrieves your bots current configuration as a
:class:`Configuration<kik.Configuration>` object.

  >>> from kik import KikApi
  >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
  >>> config = kik.get_configuration()
  >>> config.webhook
  'https://example.com/incoming'

:func:`KikApi.set_configuration<kik.KikApi.set_configuration>` sets your bot's configuration, taking a
:class:`Configuration<kik.Configuration>` object.

  >>> from kik import KikApi, Configuration
  >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
  >>> config = Configuration(webhook='https://example.com/incoming')
  >>> kik.set_configuration(config)
  <kik.Configuration>

Receiving Messages
------------------

The library contains two functions that are useful when receiving messages to your webhook

The first is :func:`KikApi.verify_signature<kik.KikApi.verify_signature>` which is takes care of `authenticating
incoming requests to your webhook <https://dev.kik.com/#/docs/messaging#api-authentication-with-webhook-endpoint>`_.

Just call the method with the provided signature header and the body of the incoming HTTP request:

    >>> from kik import KikApi
    >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
    >>> kik.verify_signature(SIGNATURE_HEADER, REQUEST_BODY)
    True

If this method returns `False`, you should ignore the incoming request, as it may be malicious.

.. note:: :func:`verify_signature<kik.KikApi.verify_signature>` must be called with the raw request body, not the parsed
   JSON

The second important function for receiving messages is
:func:`messages.messages_from_json<kik.messages.messages_from_json>`, which converts incoming messages into Python
objects.
After you parse the incoming request as JSON, simply pass the array of messages in the `messages` field to
the function to get an array of message objects.

   >>> from kik.messages import messages_from_json
   >>> messages_from_json(messages)
   [<kik.messages.TextMessage>, <kik.messages.LinkMessage>]

For a complete list of message types you might receive, see the
`Kik API Documentation <https://dev.kik.com/#/docs/messaging#message-formats>`_.

Sending Messages
----------------

Messages are sent using :func:`KikApi.send_messages<kik.KikApi.send_messages>` for the messaging API.

   >>> from kik import KikApi
   >>> from kik.messages import TextMessage
   >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
   >>> kik.send_messages([
   ...     TextMessage(
   ...         to='aleem',
   ...         chat_id='8c595a879e4140dbecb60f6c6933348bfd940cd9cbd6014e8fa51f24b5c8f74a',
   ...         body='Test'
   ...     )
   ... ])
   {}

Similarly, messages can be sent through the `broadcasting API <https://dev.kik.com/#/docs/messaging#broadcasting>`_,
using :func:`KikApi.send_broadcast<kik.KikApi.send_broadcast>`.

   >>> from kik import KikApi
   >>> from kik.messages import TextMessage
   >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
   >>> kik.send_broadcast([
   ...     TextMessage(
   ...         to='aleem',
   ...         chat_id='8c595a879e4140dbecb60f6c6933348bfd940cd9cbd6014e8fa51f24b5c8f74a',
   ...         body='Test'
   ...     )
   ... ])
   {}

Messages are constructed using the :class:`Message<kik.messages.Message>` subclasses. in :mod:`kik.messages`.
These classes directly mirror the `API message formats <https://dev.kik.com/#/docs/messaging#message-formats>`_, with
the exceptions of snake_case naming, `from` being renamed to `from_user` (as `from` is a reserved keyword in Python),
and the handling of attribution and keyboards (explained below).

Attribution
-----------

All message types that support attribution are subclasses of
:class:`AttributableMessage<kik.messages.attributable_message.AttributableMessage>`. To give custom attribution to these
messages, simply assign their ``attribution`` property to a :class:`CustomAttribution<kik.messages.CustomAttribution>`
instance.

    >>> from kik.messages import CustomAttribution, LinkMessage
    >>> message = LinkMessage()
    >>> message.attribution = CustomAttribution(
    ...     name='A Name',
    ...     icon_url='http://foo.bar/anicon'
    ... )

Additionally, there are special attribution values to make a :class:`PictureMessage<kik.messages.PictureMessage>` or
:class:`VideoMessage<kik.messages.VideoMessage>` appear to be from the camera or gallery.
To achieve these effects, assign the ``attribution`` property of the message
:const:`PresetAttributions.CAMERA<kik.messages.PresetAttributions.CAMERA>` or
:const:`PresetAttributions.GALLERY<kik.messages.PresetAttributions.GALLERY>`

    >>> from kik.messages import PresetAttributions
    >>> message = PictureMessage()
    >>> message.attribution = PresetAttributions.CAMERA

Keyboards
---------

All message types that support `keyboards <https://dev.kik.com/#/docs/messaging#keyboards>`_ are subclasses of
:class:`KeyboardMessage<kik.messages.keyboard_message.KeyboardMessage>`.
These messages contain a ``keyboards`` array holding any number of :class:`Keyboard<kik.messages.keyboards.Keyboard>`
instances.

Currently, the only supported keyboard types is
:class:`SuggestedResponseKeyboard<kik.messages.SuggestedResponseKeyboard>`, which must be assigned a ``responses`` array
of :class:`TextResponse<kik.messages.TextResponse>` instances.

   >>> from kik.messages import TextMessage, SuggestedResponseKeyboard, \
   ...     TextResponse
   >>> message = TextMessage()
   >>> message.keyboards.append(
   ...     SuggestedResponseKeyboard(
   ...         to='aleem',
   ...         hidden=True,
   ...         responses=[TextResponse('OK')]
   ...     )
   ... )

Users
-----

The User Profile API is accessed through :func:`KikApi.get_user<kik.KikApi.get_user`, which retrieves a user's profile
from their username.

The function returns a :class:`User<kik.User>`, containing the user's profile

  >>> from kik import KikApi
  >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
  >>> user = kik.get_user('aleem')
  >>> user.first_name
  'Johnny'

Kik Codes
---------

The Kik Code creation API is accessed through :func:`KikApi.create_code<kik.KikApi.create_code>`.
This function takes an optional data parameter which will be embedded in the Kik Code, and returned in the
:class:`ScanDataMessage<kik.messages.ScanDataMessage>` you receive when the user scans the code.

:func:`create_code<kik.KikApi.create_code>` returns a :class:`Code<kik.Code>`, which allows you to get a URL
for the code.

  >>> from kik import KikApi
  >>> kik = KikApi(BOT_USERNAME, BOT_API_KEY)
  >>> code = kik.create_code({'some': 'data'})
  >>> code.url()
  'https://api.kik.com/v1/code/161d764eeebf050fba373ae8cef9f5052524019a'
