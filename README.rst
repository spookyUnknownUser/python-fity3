
Fity3
-----

Fity3 is a Twitter snowflake like scheme generator that fits in 53 bits.

.. image:: https://travis-ci.org/cablehead/python-fity3.svg?branch=master
       :target: https://travis-ci.org/cablehead/python-fity3

Its scheme is::

    timestamp | worker_id | sequence
     41 bits  |  8 bits   |  4 bits

`Twitter's snowflake scheme
<https://blog.twitter.com/2010/announcing-snowflake>`_ for id generation has a
bunch of nice properties. An enormous number of roughly-sorted ids can be
created per second in an uncoordinated manner.

However it's *painful* working with 64 bit integers in environments that use
IEEE 754 floating points for numerics. Particularly `JavaScript
<https://dev.twitter.com/overview/api/twitter-ids-json-and-snowflake>`_ and the
`scores for Redis' sorted sets
<http://stackoverflow.com/questions/20295544/redis-sorted-set-wrong-score>`_.

This scheme allows for:

    * 69 years of ids (the same as snowflakes)
    * at most 256 unique id generating workers
    * each worker can produce at most 16 ids per millisecond
    * so at most 4 million ids per second

If you're building a system in the sweet spot of not being too popular that
usage will grow to creating more than 4 million new things a second but you
could benefit from uncoordinated incrementing id generation and don't want to
punch yourself in the face every time you request data over a websocket from
JavaScript, this scheme might be a useful alternative.

.. code:: python

    >>> import fity3
    >>> f3 = fity3.generator(1)
    >>> next(f3)
    14127739136
    >>> next(f3)
    14132125952
    >>> next(f3)
    14135079168

    # convenience to convert to a unix timestamp
    >>> fity3.to_timestamp(14135079168)
    1413374250

Let's just hope we won't be the ones supporting any of these systems in 2079.

Django
=====
You can also use fity3 as a primary key/something else in Django, but you have
to make a helper function to do this otherwise Django will only generate the 
snowflake when you import the model. See https://stackoverflow.com/q/16925129/1649917

Anyway to get around this all you need to do is make your model something like this:

.. code:: python

       # app/models.py
       from fity3 import generator

       from django.db import models
       
       def get_id():
           return next(generator(0))

       class Post(models.Model):
           id = models.BigIntegerField(primary_key=True, default=get_id, editable=False)
           # ...

Or you can even do something like this, to get the date from fity3 without storing it

.. code:: python

       # app/models.py
       from django.db import models
       from django.utils import timezone
       from fity3 import generator, to_timestamp


       def get_id():
           return next(generator(0))


       class Post(models.Model):
           id = models.BigIntegerField(primary_key=True, default=get_id, editable=False)
           # ...

           def created_at(self):
               timestamp = to_timestamp(self.id)
               return timezone.datetime.utcfromtimestamp(timestamp)
               
Or, if you're feeling DRY, and maybe want to use a snowflake id for every field in your db, you
can make a common class that does all this for you can make a common class:

.. code:: python

       # app/models.py
       # ... imports

       def get_id():
           return next(generator(0))


       class CommonInfo(models.Model):
           id = models.BigIntegerField(primary_key=True, default=get_id, editable=False)

           def created_at(self):
               timestamp = to_timestamp(self.id)
               return timezone.datetime.utcfromtimestamp(timestamp)

           class Meta:
               abstract = True


       class Post(CommonInfo):
           post = models.CharField(max_length=50, unique=True)
           # ...


       class SomethingElse(CommonInfo):
           name = models.CharField(max_length=20, unique=True)
           # ...

Now all classes that implement ``CommonInfo`` will have a ``snowflake id`` and a ``created_at`` field! 

Just make sure that whatever you do, you do don't do this, you'll find out pretty soon it doesn't work. :

.. code:: python

        class Post(models.Model):
                  id = models.BigIntegerField(primary_key=True, default=next(generator(0)), editable=False)
