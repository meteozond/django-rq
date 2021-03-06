=========
Django-RQ
=========

.. image:: https://secure.travis-ci.org/ui/django-rq.png

Django integration with `RQ <https://github.com/nvie/rq>`_, a `Redis <http://redis.io/>`_
based Python queuing library. `Django-RQ <https://github.com/ui/django-rq>`_ is a
simple app that allows you to configure your queues in django's ``settings.py``
and easily use them in your project.

============
Requirements
============

* `Django <https://www.djangoproject.com/>`_
* `RQ`_

============
Installation
============

* Install ``django-rq`` (or `download from PyPI <http://pypi.python.org/pypi/django-rq>`_):

.. code-block:: python

    pip install django-rq

* Add ``django_rq`` to ``INSTALLED_APPS`` in ``settings.py``:

.. code-block:: python

    INSTALLED_APPS = (
        # other apps
        "django_rq",
    )

* Configure your queues in django's ``settings.py`` (syntax based on Django's database config):

.. code-block:: python

    RQ_CONNECTIONS = {
        'default': {
            'HOST': 'localhost',
            'PORT': 6379,
            'DB': 0,
            'PASSWORD': 'some-password',
        },
        'high': {
            'URL': os.getenv('REDISTOGO_URL', 'redis://localhost:6379'), # If you're on Heroku
            'DB': 0,
        },
        'low': {
            'HOST': 'localhost',
            'PORT': 6379,
            'DB': 0,
        }
    }

* Include ``django_rq.urls`` in your ``urls.py``:

.. code-block:: python

    urlpatterns += patterns('',
        (r'^django-rq/', include('django_rq.urls')),
    )


=====
Usage
=====

Putting jobs in the queue
-------------------------

`Django-RQ` allows you to easily put jobs into any of the queues defined in
``settings.py``. It comes with a few utility functions:

* ``enqueue`` - push a job to the ``default`` queue of ``default`` connection:

.. code-block:: python

    import django_rq
    django_rq.enqueue(func, foo, bar=baz)

* ``get_queue`` - accepts a single queue and connection name arguments
  (defaults to "default") and returns an `RQ` ``Queue`` instance for you
  to queue jobs into:

.. code-block:: python

    import django_rq
    queue = django_rq.get_queue('high', connection_name='redis')
    queue.enqueue(func, foo, bar=baz)

* ``get_connection`` - accepts a single connection name argument (defaults
  to "default") and returns a connection to the queue's `Redis`_ server:

.. code-block:: python

    import django_rq
    redis_conn = django_rq.get_connection('high')

* ``get_worker`` - accepts optional combination of connection and queue names
  and returns a new `RQ` ``Worker`` instance for specified queues (or ``default`` queue):

.. code-block:: python

    import django_rq
    worker = django_rq.get_worker() # Returns a worker for "default" queue
    worker.work()
    worker = django_rq.get_worker('default.low', 'redis.high') # Returns a worker for "low" and "high"


@job decorator
--------------

To easily turn a callable into an RQ task, you can also use the ``@job``
decorator that comes with ``django_rq``:

.. code-block:: python

    from django_rq import job

    @job
    def long_running_func():
        pass
    long_running_func.delay() # Enqueue function in "default" queue

    @job('high', connection_name='default')
    def long_running_func():
        pass
    long_running_func.delay() # Enqueue function in "high" queue


Running workers
---------------
django_rq provides a management command that starts a worker for every queue
specified as arguments::

    python manage.py rqworker default.high default.default default.low

If you want to run ``rqworker`` in burst mode, you can pass in the ``--burst`` flag::

    python manage.py rqworker default.high default.default low --burst

Support for RQ Scheduler
------------------------

If you have `RQ Scheduler <https://github.com/ui/rq-scheduler>`_ installed,
you can also use the ``get_scheduler`` function to return a ``Scheduler``
instance for queues defined in settings.py's ``RQ_CONNECTIONS``. For example:

.. code-block:: python

    import django_rq
    scheduler = django_rq.get_scheduler('default', 'default')
    job = scheduler.enqueue_at(datetime(2020, 10, 10), func)

Support for django-redis and django-redis-cache
-----------------------------------------------

If you have `django-redis <https://django-redis.readthedocs.org/>`_ or
`django-redis-cache <https://github.com/sebleier/django-redis-cache/>`_
installed, you can instruct django_rq to use the same connection information
from your Redis cache. This has two advantages: it's DRY and it takes advantage
of any optimization that may be going on in your cache setup (like using
connection pooling or `Hiredis <https://github.com/redis/hiredis>`_.)

To use configure it, use a dict with the key ``USE_REDIS_CACHE`` pointing to the
name of the desired cache in your ``RQ_CONNECTIONS`` dict. It goes without saying
that the chosen cache must exist and use the Redis backend. See your respective
Redis cache package docs for configuration instructions. It's also important to
point out that since the django-redis-cache ``ShardedClient`` splits the cache
over multiple Redis connections, it does not work. Here is an example settings
fragment for django-redis:

.. code-block:: python

    CACHES = {
        'redis-cache': {
            'BACKEND': 'redis_cache.cache.RedisCache',
            'LOCATION': 'localhost:6379:1',
            'OPTIONS': {
                'CLIENT_CLASS': 'redis_cache.client.DefaultClient',
                'MAX_ENTRIES': 5000,
            },
        },
    }

    RQ_CONNECTIONS = {
        'high': {
            'USE_REDIS_CACHE': 'redis-cache',
        },
        'low': {
            'USE_REDIS_CACHE': 'redis-cache',
        },
    }

Queue statistics
----------------

``django_rq`` also provides a dashboard to monitor the status of your queues at
``/django-rq/`` (or whatever URL you set in your ``urls.py`` during installation.

You can also add a link to this dashboard link in ``/admin`` by adding
``RQ_SHOW_ADMIN_LINK = True`` in ``settings.py``. Be careful though, this will
override the default admin template so it may interfere with other apps that
modifies the default admin template.


Configuring Logging
-------------------

Starting from version 0.3.3, RQ uses Python's ``logging``, this means
you can easily configure ``rqworker``'s logging mechanism in django's
``settings.py``. For example:

.. code-block:: python

    LOGGING = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "rq_console": {
                "format": "%(asctime)s %(message)s",
                "datefmt": "%H:%M:%S",
            },
        },
        "handlers": {
            "rq_console": {
                "level": "DEBUG",
                "class": "rq.utils.ColorizingStreamHandler",
                "formatter": "rq_console",
                "exclude": ["%(asctime)s"],
            },
            # If you use sentry for logging
            'sentry': {
                'level': 'ERROR',
                'class': 'raven.contrib.django.handlers.SentryHandler',
            },
        },
        'loggers': {
            "rq.worker": {
                "handlers": ["rq_console", "sentry"],
                "level": "DEBUG"
            },
        }
    }


Testing tip
-----------

For an easier testing process, you can run a worker synchronously this way:

.. code-block:: python

    from django.test impor TestCase
    from django_rq import get_worker

    class MyTest(TestCase):
        def test_something_that_creates_jobs(self):
            ...                      # Stuff that init jobs.
            get_worker().work(burst=True)  # Processes all jobs then stop.
            ...                      # Asserts that the job stuff is done.

Synchronous mode
----------------

You can set the option ``ASYNC`` to ``False`` to make synchronous operation the
default for a given queue. This will cause jobs to execute immediately and on
the same thread as they are dispatched, which is useful for testing and
debugging. For example, you might add the following after you queue
configuration in your settings file:

.. code-block:: python

    # ... Logic to set DEBUG and TESTING settings to True or False ...

    # ... Regular RQ_CONNECTIONS setup code ...

    if DEBUG or TESTING:
        for queueConfig in RQ_CONNECTIONS.itervalues():
            queueConfig['ASYNC'] = False

Note that setting the ``async`` parameter explicitly when calling ``get_queue``
will override this setting.

=============
Running Tests
=============

To run ``django_rq``'s test suite::

    django-admin.py test django_rq --settings=django_rq.test_settings --pythonpath=.

=========
Changelog
=========

0.?.0
-----
* Dynamic QUEUE creation
* Getting queues list from redis
* ``RQ_QUEUES`` changed to `RQ_CONNECTIONS``
* Added ``connection_name`` argument to specify queue location


0.5.0
-----
* Added ``ASYNC`` option to ``RQ_QUEUES``
* Added ``get_failed_queue`` shortcut
* Django-RQ can now reuse existing ``django-redis`` cache connections
* Added an experimental (and undocumented) ``AUTOCOMMIT`` option, use at your own risk


0.4.7
-----
* Make admin template override optional.

0.4.6
-----
* ``get_queue`` now accepts ``async`` and ``default_timeout`` arguments
* Minor updates to admin interface

0.4.5
-----
* Added the ability to requeue failed jobs in the admin interface
* In addition to deleting the actual job from Redis, job id is now also
  correctly removed from the queue
* Bumped up ``RQ`` requirement to 0.3.4 as earlier versions cause logging to fail
  (thanks @hugorodgerbrown)

Version 0.4.4
-------------
* ``rqworker`` management command now uses django.utils.log.dictConfig so it's
  usable on Python 2.6

Version 0.4.3
-------------

* Added ``--burst`` option to ``rqworker`` management command
* Added support for Python's ``logging``, introduced in ``RQ`` 0.3.3
* Fixed a bug that causes jobs using RQ's new ``get_current_job`` to fail when
  executed through the ``rqworker`` management command

Version 0.4.2
-------------
Fixed a minor bug in accessing `rq_job_detail` view.

Version 0.4.1
-------------
More improvements to `/admin/django_rq/`:

* Views now require staff permission
* Now you can delete jobs from queue
* Failed jobs' tracebacks are better formatted

Version 0.4.0
-------------
Greatly improved `/admin/django_rq/`, now you can:

* See jobs in each queue, including failed queue
* See each job's detailed information

Version 0.3.2
-------------
* Simplified ``@job`` decorator syntax for enqueuing to "default" queue.

Version 0.3.1
-------------
* Queues can now be configured using the URL parameter in ``settings.py``.

Version 0.3.0
-------------
* Added support for RQ's ``@job`` decorator
* Added ``get_worker`` command

Version 0.2.2
-------------
* "PASSWORD" key in RQ_QUEUES will now be used when connecting to Redis.
