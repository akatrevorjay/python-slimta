
.. include:: /global.rst

.. _gevent_subprocess: https://github.com/bombela/gevent_subprocess
.. _courier-maildrop: http://www.courier-mta.org/maildrop/
.. _dovecot-deliver: http://wiki.dovecot.org/LDA
.. _pyaio: https://github.com/felipecruz/pyaio
.. _ESPs: http://en.wikipedia.org/wiki/E-mail_service_provider
.. _spam-filled world: http://www.maawg.org/email_metrics_report
.. _Celery Distributed Task Queue: http://www.celeryproject.org/

Extensions
==========

In an effort to keep a small number of dependencies and avoid a reputation for
"bloating", several desirable features are included as extensions installed
separately. These extensions, and where and how to use them, are discussed
here.

.. _maildrop-relay:

Maildrop Delivery
"""""""""""""""""

This simple extension uses the `gevent_subprocess`_ library to initiate local
delivery using `courier-maildrop`_. It could easily be adapted to use another
command-based LDA such as `dovecot-deliver`_.

Installation of this extension is as simple as::

    $ pip install python-slimta-maildrop

With a normal system configuration, the following should be plenty to create a
:class:`~slimta.maildroprelay.MaildropRelay` instance::

    from slimta.maildroprelay import MaildropRelay

    relay = MaildropRelay()

For more information, the :doc:`mda` tutorial and :mod:`~slimta.maildroprelay`
module documentation may be useful.

.. _disk-storage:

Disk Storage
""""""""""""

In the fashion of traditional MTAs, the :mod:`~slimta.diskstorage` extension
writes |Envelope| data and queue metadata directly to disk to configurable
directories. This extension relies on `pyaio`_ to asynchronously write and read
files on disk.

To ensure file creation and modification is atomic, files are
first written to a scratch directory and then :func:`os.rename` moves them to
their final destination. For this reason, it is important that the scratch
directory (``tmp_dir`` argument in the constructor) reside on the same
filesystem as the envelope and meta directories (``env_dir`` and ``meta_dir``
arguments, respectively).

The files created in the envelope directory will be identified by a
:func:`~uuid.uuid4` :attr:`hexadecimal string <uuid.UUID.hex>` appended with
the suffix ``.env``. The files created in the meta directory will be identified
by the same uuid string as its corresponding envelope file, but with the suffix
``.meta``. The envelope and meta directories can be the same, but two
:class:`~slimta.diskstorage.DiskStorage` should not share directories.

To install this extension::

    $ pip install python-slimta-diskstorage

And to initialize a new :class:`~slimta.diskstorage.DiskStorage`::

    from slimta.diskstorage import DiskStorage

    queue_dir = '/var/spool/slimta/queue'
    queue = DiskStorage(queue_dir, queue_dir)

.. _celery-queue:

Celery Distributed Queuing
""""""""""""""""""""""""""

Why It's Better
'''''''''''''''

One of the original inspirations for |slimta| was splitting apart the "big 3"
components of an MTA in such a way that different server clusters could be
responsible for each component. These "big 3" components are called the *edge*,
the *queue*, and the *relay* in this library.

One of the largest and most complicated pieces of logic in modern-day `ESPs`_
is anti-abuse. It is reasonable to assume that simply handling inbound traffic
from a `spam-filled world`_ is more than enough for a server cluster to be
responsible for. The server running the *edge* service should be able to simply
hand off a received message for delivery to the *queue* running on another
machine.

Despite the name, a *queue* in the MTA sense is not a simple FIFO we learned
about in our Computer Science course. It is responsible at a minimum for:

* Receiving new messages from the *edge* and persistently storing them.
* Requesting delivery from the *relay* service.
* Delaying message delivery retry after transient failures.
* Reporting permanent delivery failure back to the sender with a bounce
  message.

If you're familiar with the `Celery Distributed Task Queue`_, it fits the bill
perfectly.

Setting It Up
'''''''''''''

Celery will actually take care of managing the *relay* and *queue* services,
when all is said and done. The message broker and results backend of Celery
act as the *queue*, and the task workers act as the *relay*.

In a new file (I called mine ``mytasks.py``), set up your ``celery`` object::

    from celery import Celery

    celery = Celery('mytasks', broker='redis://localhost/0',
                               backend='redis://localhost/0')

We'll also set up our |Relay| object now::

    from slimta.relay.smtp.mx import MxSmtpRelay

    relay = MxSmtpRelay()

Next, create a new :class:`~slimta.celeryqueue.CeleryQueue` using both of these
objects::

    from slimta.celeryqueue import CeleryQueue

    queue = CeleryQueue(celery, relay)

Simply creating a :class:`~slimta.celeryqueue.CeleryQueue` instance will
register a new celery task called ``attempt_delivery``. Each delivery attempt
and retry will call this task, including delivery of bounce messages.

Now, back inside your |slimta| application code, you can import ``queue``
from this file, add your policies, and create your |Edge|::

    from mytasks import queue
    from slimta.policy.headers import *
    from slimta.edge.smtp import SmtpEdge

    queue.add_policy(AddDateHeader())
    queue.add_policy(AddMessageIdHeader())
    queue.add_policy(AddReceivedHeader())

    edge = SmtpEdge(('', 25), queue)
    edge.start()

Finally, in a new terminal start your task worker, using :mod:`gevent` as the
worker thread pool::

    $ celery worker -A mytasks -l debug -P gevent

Now you are all set up with a distributed Celery queue! You're now free to
scale your *edge* by adding more machines running |SmtpEdge| to the a ``queue``
with the same backend and broker. You're now free to scale your *queue* by
scaling your backend and broker (might be easier with RabbitMQ than Redis).
And finally, you're free to scale your *relay* by adding machines designated as
Celery task workers. Go nuts!

