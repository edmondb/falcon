Falcon [![Build Status](https://travis-ci.org/racker/falcon.png)](https://travis-ci.org/racker/falcon) [![Coverage Status](https://coveralls.io/repos/racker/falcon/badge.png?branch=master)](https://coveralls.io/r/racker/falcon)
======

:runner: Come hang out with us in **#falconframework** on freenode.

Falcon is a [high-performance Python framework][home] for building cloud APIs. It encourages the REST architectural style, and tries to do as little as possible while remaining [highly effective][benefits].

> Perfection is finally attained not when there is no longer anything to add, but when there is no longer anything to take away.
>
> *- Antoine de Saint-Exupéry*

[home]: http://falconframework.org/index.html
[benefits]: http://falconframework.org/index.html#Benefits

<img align="right" src="http://falconframework.org/img/falcon.png" alt="Falcon picture" />

### Design Goals ###

**Fast.** Cloud APIs need to turn around requests quickly, and make efficient use of hardware. This is particularly important when serving many concurrent requests. Falcon processes requests [several times faster][bench] than other popular web frameworks.

**Light.** Only the essentials are included, with *six* and *mimeparse* being the only dependencies outside the standard library. We work to keep the code lean, making Falcon easier to test, optimize, and deploy.

**Flexible.** Falcon can be deployed in a variety of ways, depending on your needs. The framework speaks WSGI, and works great with [Python 2.6 and 2.7, PyPy, and Python 3.3][ci]. There's no tight coupling with any async framework, leaving you free to mix-and-match what you need.

[bench]: http://falconframework.org/#Metrics
[ci]: https://travis-ci.org/racker/falcon

### Features ###

* Intuitive routing via URI templates and resource classes
* Easy access to headers and bodies through request and response classes
* Idiomatic HTTP error responses via a handy exception base class
* DRY request processing using global, resource, and method hooks
* Snappy unit testing through WSGI helpers and mocks
* 20% speed boost when Cython is available
* Python 2.6, Python 2.7, PyPy and Python 3.3 support
* Speed, speed, and more speed!

### Install ###

```bash
$ pip install cython falcon
```

### Test ###

```bash
$ pip install nose && nosetests
```

To test across all supported Python versions:

```bash
$ pip install tox && tox
```

### Usage ###

Docstrings can be found throughout the Falcon code base for your learning pleasure. You can also ask questions in **#falconframework** on freenode. We are planning on having real docs eventually and would of course greatly appreciate pull requests to help accelerate that.

You can also check out [Marconi's WSGI driver](https://github.com/openstack/marconi/tree/master/marconi/queues/transport/wsgi) to get a feel for how you might
leverage Falcon in a real-world app.

Here is a simple, contrived example showing how to create a Falcon-based API.

```python
# things.py

# Let's get this party started
import falcon


# Falcon follows the REST architectural style, meaning (among
# other things) that you think in terms of resources and state
# transitions, which map to HTTP verbs.
class ThingsResource:
    def on_get(self, req, resp):
        """Handles GET requests"""
        resp.status = falcon.HTTP_200  # This is the default status
        resp.body = ('\nTwo things awe me most, the starry sky '
                     'above me and the moral law within me.\n'
                     '\n'
                     '    ~ Immanuel Kant\n\n')

# falcon.API instances are callable WSGI apps
app = falcon.API()

# Resources are represented by long-lived class instances
things = ThingsResource()

# things will handle all requests to the '/things' URL path
app.add_route('/things', things)

```

You can run the above example using any WSGI server, such as uWSGI
or Gunicorn. For example:

```bash
$ pip install gunicorn
$ gunicorn things:app
```

Then, in another terminal:

```bash
$ curl localhost:8000/things
```

### A More Complex Example ###

Here is a more involved example that demonstrates reading headers and query parameters, handling errors, and working with request and response bodies.

```python

import json
import logging
from wsgiref import simple_server

import falcon


class StorageEngine:
    pass


class StorageError(Exception):
    @staticmethod
    def handle(ex, req, resp, params):
        description = ('Sorry, couldn\'t write your thing to the '
                       'database. It worked on my box.')

        raise falcon.HTTPError(falcon.HTTP_725,
                               'Database Error',
                               description)


class Proxy(object):
    def forward(self, req):
        return falcon.HTTP_503


class SinkAdapter(object):

    def __init__(self):
        self._proxy = Proxy()

    def __call__(self, req, resp, **kwargs):
        resp.status = self._proxy.forward(req)
        self.kwargs = kwargs


def token_is_valid(token, user_id):
    return True  # Suuuuuure it's valid...


def auth(req, resp, params):
    # Alternatively, use Talons or do this in WSGI middleware...
    token = req.get_header('X-Auth-Token')

    if token is None:
        description = ('Please provide an auth token '
                       'as part of the request.')

        raise falcon.HTTPUnauthorized('Auth token required',
                                      description,
                                      href='http://docs.example.com/auth')

    if not token_is_valid(token, params['user_id']):
        description = ('The provided auth token is not valid. '
                       'Please request a new token and try again.')

        raise falcon.HTTPUnauthorized('Authentication required',
                                      description,
                                      href='http://docs.example.com/auth',
                                      scheme='Token; UUID')


def check_media_type(req, resp, params):
    if not req.client_accepts_json:
        raise falcon.HTTPUnsupportedMediaType(
            'This API only supports the JSON media type.',
            href='http://docs.examples.com/api/json')


class ThingsResource:

    def __init__(self, db):
        self.db = db
        self.logger = logging.getLogger('thingsapp.' + __name__)

    def on_get(self, req, resp, user_id):
        marker = req.get_param('marker') or ''
        limit = req.get_param_as_int('limit') or 50

        try:
            result = self.db.get_things(marker, limit)
        except Exception as ex:
            self.logger.error(ex)

            description = ('Aliens have attacked our base! We will '
                           'be back as soon as we fight them off. '
                           'We appreciate your patience.')

            raise falcon.HTTPServiceUnavailable(
                'Service Outage',
                description,
                30)

        resp.set_header('X-Powered-By', 'Donuts')
        resp.status = falcon.HTTP_200
        resp.body = json.dumps(result)

    def on_post(self, req, resp, user_id):
        try:
            # req.stream corresponds to the WSGI wsgi.input environ variable,
            # and allows you to read bytes from the request body.
            #
            # json.load assumes the input stream is encoded at utf-8 if the
            # encoding is not specified explicitly.
            #
            # See also: PEP 3333
            thing = json.load(req.stream)
        except ValueError:
            raise falcon.HTTPError(falcon.HTTP_753,
                                   'Malformed JSON',
                                   'Could not decode the request body. The '
                                   'JSON was incorrect.')

        proper_thing = self.db.add_thing(thing)

        resp.status = falcon.HTTP_201
        resp.location = '/%s/things/%s' % (user_id, proper_thing.id)

# Configure your WSGI server to load "things.app" (app is a WSGI callable)
app = falcon.API(before=[auth, check_media_type])

db = StorageEngine()
things = ThingsResource(db)
app.add_route('/{user_id}/things', things)

# If a responder ever raised an instance of StorageError, pass control to
# the given handler.
app.add_error_handler(StorageError, StorageError.handle)

# Proxy some things to another service. This example shows how you might
# send parts of an API off to a legacy system that hasn't been upgraded
# yet, or perhaps is a single cluster that all datacenters have to share.
sink = SinkAdapter()
app.add_sink(sink, r'/v1/[charts|inventory]')

# Useful for debugging problems in your API; works with pdb.set_trace()
if __name__ == '__main__':
    httpd = simple_server.make_server('127.0.0.1', 8000, app)
    httpd.serve_forever()

```

### Contributing ###

Kurt Griffiths (kgriffs) is the creator and current maintainer of the
Falcon framework, with the generous help of a number of contributors. Pull requests are always welcome.

Before submitting a pull request, please ensure you have added/updated the appropriate tests (and that all existing tests still pass with your changes), and that your coding style follows PEP 8 and doesn't cause pyflakes to complain.

Commit messages should be formatted using [AngularJS conventions][ajs] (one-liners are OK for now but body and footer may be required as the project matures).

Comments follow [Google's style guide][goog-style-comments].

[ajs]: http://goo.gl/QpbS7
[goog-style-comments]: http://google-styleguide.googlecode.com/svn/trunk/pyguide.html#Comments

### Legal ###

Copyright 2013 by Rackspace Hosting, Inc.

Falcon image courtesy of [John O'Neill](https://commons.wikimedia.org/wiki/File:Brown-Falcon,-Vic,-3.1.2008.jpg).

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
