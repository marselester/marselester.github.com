==================
API based on Flask
==================

:date: 2013-12-09
:tags: python, flask, api
:category: Python
:slug: api-based-on-flask

Here I want to consider implementation of API best practices which
usually don't follow Fielding's REST strictly. `Example Flask project`_
is on GitHub.

.. note::
    Post will be updated.

API Versioning
--------------

Interfaces are changed hence versioning is mandatory in order to not annoy
your users. You might need to add new resource or field to particular resource.
You write code, tests and update documentation. Users are happy.
It's possible that you have to rename or delete field of some resource.
This case is harder and you might make the easiest decision — spawn
a lot of ``if`` statements and write more tests consequently.
Code base maintaining is getting worse.

I think it's better to make API versions isolated.
It will keep things simple and tests as well. You may even want to change
framework if legacy API implementation is not good enough.
For example, you have two WSGI applications. Each application implements
certain API version, e.g, **messaging_api_v1**, **messaging_api_v2**.
In order to hide versioning information from applications, e.g., URL prefix,
you can dispatch requests by Werkzeug's ``DispatcherMiddleware``.

.. code-block:: python

    from werkzeug.wsgi import DispatcherMiddleware
    from werkzeug.exceptions import NotFound

    import messaging_api_v1
    import messaging_api_v2


    app = DispatcherMiddleware(NotFound(), {
        '/v1': messaging_api_v1.app,
        '/v2': messaging_api_v2.app,
    })

Dispatcher in work:

.. code-block:: console

    $ gunicorn messaging_api:app

Let's request the same API resource from different WSGI applications:

.. code-block:: console

    $ curl http://127.0.0.1:8000/v1/messages/1
    {
      "content": "hi world from V1"
    }
    $ curl http://127.0.0.1:8000/v2/messages/1
    {
      "content": "hi world from V2"
    }

URLs
----

API resources are nouns, so ``/messages/`` URL is for a collection of messages.
You should think about it as UNIX folder. Hence ``/messages`` is correct
folder path and Flask will redirect to the canonical URL ``/messages/``.
Certain message's ``/messages/1`` URL must not contain trailing slash
in order to look like UNIX file path.

Resources usually have relationships and they might be expressed in URLs,
e.g., get messages from account which id is ``1``.

.. code-block:: console

    $ curl https://api.example.com/v1/accounts/1/messages/

.. _Example Flask project: https://github.com/marselester/api-example-based-on-flask
