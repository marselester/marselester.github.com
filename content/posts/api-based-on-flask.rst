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

Representation
--------------

There are two popular approaches to specify response format. First one
is based on **Accept** header, second — URL based. Here I use header based
approach.

I wrote ResponsiveFlask_ class which extends ``Flask`` by supporting
dictionary response. Views can return dict and it will be represented
based on **Accept** header. When ``ResponsiveFlask.make_response()`` receives
dictionary it creates real response object using appropriate formatter.
Formatter is picked by mimetype.

.. code-block:: console

    $ curl http://127.0.0.1:8000/v2/messages/1 -i -H 'Accept: application/json'
    HTTP/1.1 200 OK
    Server: gunicorn/18.0
    Date: Tue, 10 Dec 2013 07:52:31 GMT
    Connection: close
    Content-Type: application/json
    Content-Length: 35

    {
      "content": "hi world from V2"
    }

Error Handling
--------------

Flask shows error pages by default with basic description in
**text/html** format. It would be better if error representation depends
on **Accept** header. ``ResponsiveFlask`` class concerns about it.

.. code-block:: console

    $ curl http://127.0.0.1:8000/v2/messages/666 -i
    HTTP/1.1 404 NOT FOUND
    Server: gunicorn/18.0
    Date: Fri, 03 Jan 2014 05:16:03 GMT
    Connection: close
    Content-Type: application/json
    Content-Length: 49

    {
      "code": 404,
      "message": "404: Not Found"
    }

You can set your own HTTP error handler by using
app.default_errorhandler_ decorator. Note that it might override
already defined error handlers, so you should declare it before them.

It's convenient to add URL of detailed error description into response.

.. code-block:: console

    {
      "code": 404,
      "info_url": "http://developer.example.com/errors.html#error-code-404",
      "message": "404: Not Found"
    }

Misc
----

It's good idea to keep in mind following:

- HTTPS;
- response should contain resource url, e.g.,
  ``{'url': 'https://api.example.com/v2/messages/1'}``;
- pagination by ``offset`` and ``limit`` QS arguments with default values;
- filtration and search by QS arguments;
- partial response by ``fields=id,lastname`` QS argument.

.. _Example Flask project: https://github.com/marselester/api-example-based-on-flask
.. _ResponsiveFlask: https://github.com/marselester/flask-api-utils#accept-header-based-response
.. _app.default_errorhandler: https://github.com/marselester/flask-api-utils#error-handling
