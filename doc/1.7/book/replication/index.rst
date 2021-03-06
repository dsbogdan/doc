.. _index-box_replication:

--------------------------------------------------------------------------------
Replication
--------------------------------------------------------------------------------

Replication allows multiple Tarantool instances to work on copies of the same
databases. The databases are kept in synch because each instance can communicate
its changes to all the other instances. A pack of instances which share the same databases
are a "replica set". Each instance in a replica set also has a numeric identifier which
is unique within the replica set, known as the "instance id" or "instance id".

To set up replication, it's necessary to set up the master instances which
make the original data-change requests, set up the replica instances which
copy data-change requests from masters, and establish procedures for
recovery from a degraded state.

=====================================================================
Replication architecture
=====================================================================

A replica gets all updates from the master by continuously fetching and
applying its write-ahead log (WAL). Each record in the WAL represents a
single Tarantool data-change request such as INSERT or UPDATE or DELETE,
and is assigned a monotonically growing log sequence number (LSN). In
essence, Tarantool replication is row-based: each data change command is
fully deterministic and operates on a single tuple.

A stored program invocation is not written to the write-ahead log. Instead,
log events for actual data-change requests, performed by the Lua code, are
written to the log. This ensures that possible non-determinism of Lua does
not cause replication to go out of sync.

A change to a temporary space is not written to the write-ahead log,
and therefore is not replicated.

=====================================================================
Setting up a master
=====================================================================

To prepare the master for connections from the replica, it's only necessary
to include ":ref:`listen <cfg_basic-listen>`" in the initial ``box.cfg`` request,
for example ``box.cfg{listen=3301}``. A master with enabled "listen" URI can accept
connections from as many replicas as necessary on that URI. Each replica
has its own :ref:`replication state <index-monitoring_replica_actions>`.

=====================================================================
Setting up a replica
=====================================================================

An instance requires a valid snapshot (.snap) file. A snapshot file is created
for a instance the first time that ``box.cfg`` occurs for it. If this first
``box.cfg`` request occurs without a "replication source" clause, then the
instance is a master and starts its own new replica set with a new unique UUID.
If this first ``box.cfg`` request occurs with a "replication source" clause,
then the instance is a replica and its snapshot file, along with the replica-set
information, is constructed from the write-ahead logs of the master.
Therefore, to start replication, specify the replication source in the
:ref:`replication <cfg_replication-replication>` parameter of a ``box.cfg``
request. When a replica contacts a master for the first time, it becomes part of
a replica set. On subsequent occasions, it should always contact a master in the
same replica set.

Once connected to the master, the replica requests all changes that happened
after the latest local LSN. It is therefore necessary to keep WAL files on
the master host as long as there are replicas that haven't applied them yet.
A replica can be "re-seeded" by deleting all its files (the snapshot .snap
file and the WAL .xlog files), then starting replication again - the replica
will then catch up with the master by retrieving all the master's tuples.
Again, this procedure works only if the master's WAL files are present.

.. NOTE::

    Replication parameters are "dynamic", which allows the replica to become
    a master and vice versa with the help of the
    :ref:`box.cfg <box_introspection-box_cfg>` statement.

.. NOTE::

    The replica does not inherit the master's configuration parameters, such
    as the ones that cause the :ref:`checkpoint daemon <book_cfg_checkpoint_daemon>`
    to run on the master. To get the same behavior, one would have to set the
    relevant parameters explicitly so that they are the same on both master and
    replica.

.. NOTE::

    Replication requires privileges. Privileges for accessing spaces could be
    granted directly to the user who will start the replica. However, it is more
    usual to grant privileges for accessing spaces to a
    :ref:`role <authentication-roles>`, and then grant the role to the user
    who will start the replica.

=====================================================================
Recovering from a degraded state
=====================================================================

"Degraded state" is a situation when the master becomes unavailable - due to
hardware or network failure, or due to a programming bug. There is no automatic
way for a replica to detect that the master is gone forever, since sources of
failure and replication environments vary significantly. So the detection of
degraded state requires a human inspection.

However, once a master failure is detected, the recovery is simple: declare
that the replica is now the new master, by saying
:extsamp:`box.cfg{... listen={*{URI}*} ...}`
Then, if there are updates on the old master that were not propagated before
the old master went down, they would have to be re-applied manually.

=====================================================================
Quick startup of a new simple two-instance replica set
=====================================================================

Step 1. Start the first instance thus:

.. cssclass:: highlight
.. parsed-literal::

    box.cfg{listen = *uri#1*}
    -- replace with more restrictive request
    box.schema.user.grant('guest', 'read,write,execute', 'universe')
    box.snapshot()

... Now a new replica set exists.

Step 2. Check where the second instance's files will go by looking at its
directories (:ref:`memtx_dir <cfg_basic-memtx_dir>` for snapshot files,
:ref:`wal_dir <cfg_basic-wal_dir>` for .xlog files).
They must be empty - when the second instance joins for the first time, it
has to be working with a clean state so that the initial copy of the first
instance's databases can happen without conflicts.

Step 3. Start the second instance thus:

.. cssclass:: highlight
.. parsed-literal::

    box.cfg{
      listen = *uri#2*,
      replication = *uri#1*
    }

... where ``uri#1`` = the :ref:`URI <index-uri>` that the first instance is listening on.

That's all.

In this configuration, the first instance is the "master" and the second instance
is the "replica". Henceforth every change that happens on the master will be
visible on the replica. A simple two-instance replica set with the master on one
computer and the replica on a different computer is very common and provides
two benefits: FAILOVER (because if the master goes down then the replica can
take over), or LOAD BALANCING (because clients can connect to either the master
or the replica for select requests). Sometimes the replica may be configured with
the additional parameter :ref:`read_only = true <cfg_basic-read_only>`.

.. _index-monitoring_replica_actions:

=====================================================================
Monitoring a replica's actions
=====================================================================

In :ref:`box.info <box_introspection-box_info>` there is a ``box.info.replication.status`` field:
"off", "stopped", "connecting", "auth", "follow", or "disconnected". |br|
If a replica's status is "follow", then there will be more fields --
the list is in the section :ref:`Submodule box.info <box_introspection-box_info>`.

In the :ref:`log <log-module>` there is a record of replication activity.
If a primary instance is started with:

.. cssclass:: highlight
.. parsed-literal::

    box.cfg{
      <...>,
      logger = *log file name*,
      <...>
    }

then there will be lines in the log file, containing the word "relay",
when a replica connects or disconnects.

.. _index-preventing_duplicate_actions:

=====================================================================
Preventing duplicate actions
=====================================================================

Suppose that the replica tries to do something that the master has already done.
For example: |br|
``box.schema.space.create('X')`` |br|
This would cause an error, "Space X exists".
For this particular situation, the code could be changed to: |br|
``box.schema.space.create('X', {if_not_exists=true})`` |br|
But there is a more general solution: the
:samp:`box.once({key}, {function})` method.
If ``box.once()`` has been called before with the
same :samp:`{key}` value, then :samp:`{function}`
is ignored; otherwise :samp:`{function}` is executed.
Therefore, actions which should only occur once during the
life of a replicated session should be placed in a function
which is executed via ``box.once()``. For example:

.. code-block:: lua

    function f()
      box.schema.space.create('X')
    end
    box.once('space_creator', f)

=====================================================================
Master-master replication
=====================================================================

In the simple master-replica configuration, the master's changes are seen by
the replica, but not vice versa, because the master was specified as the sole
replication source. In the master-master configuration,
also sometimes called multi-master configuration,
it's possible to go both ways.
Starting with the simple configuration, the first instance has to say:

.. cssclass:: highlight
.. parsed-literal::

    box.cfg{ replication = *uri#2* }

This request can be performed at any time --
:ref:`replication <cfg_replication-replication>` is a dynamic parameter.

In this configuration, both instances are "masters" and both instances are
"replicas". Henceforth every change that happens on either instance will
be visible on the other. The failover benefit is still present, and the
load-balancing benefit is enhanced (because clients can connect to either
instance for data-change requests as well as select requests).

If two operations for the same tuple take place "concurrently" (which can
involve a long interval because replication is asynchronous), and one of
the operations is ``delete`` or ``replace``, there is a possibility that
instances will end up with different contents.

=====================================================================
All the "What If?" questions
=====================================================================

.. container:: faq

    :Q: What if there are more than two instances with master-master?
    :A: On each instance, specify the :ref:`replication source
        <cfg_replication-replication>` for all the others. For example,
        instance #3 would have a request:

        .. cssclass:: highlight
        .. parsed-literal::

            box.cfg{ replication = {*uri1*}, {*uri2*} }


    :Q: What if an instance should be taken out of the replica set?
    :A: For a replica, run ``box.cfg{}`` again specifying a blank replication
        source: ``box.cfg{replication=''}``

    :Q: What if an instance leaves the replica set?
    :A: The other instances carry on. If the wayward instance rejoins, it will
        receive all the updates that the other instances made while it was away.

    :Q: What if two instances both change the same tuple?
    :A: The last changer wins. For example, suppose that instance#1 changes the
        tuple, then instance#2 changes the tuple. In that case instance#2's change
        overrides whatever instance#1 did. In order to keep track of who came
        last, Tarantool implements a `vector clock
        <https://en.wikipedia.org/wiki/Vector_clock>`_.

    :Q: What if two instances both insert the same tuple?
    :A: If a master tries to insert a tuple which a replica has inserted
        already, this is an example of a severe error. Replication stops.
        It will have to be restarted manually.

    :Q: What if a master disappears and the replica must take over?
    :A: A message will appear on the replica stating that the connection is
        lost. The replica must now become independent, which can be done by
        saying ``box.cfg{replication=''}``.

    :Q: What if it's necessary to know what replica set an instance is in?
    :A: The identification of the replica set is a UUID which is generated when the
        first master starts for the first time. This UUID is stored in a tuple
        of the :ref:`box.space._schema <box_space-schema>` system space. So to
        see it, say: ``box.space._schema:select{'cluster'}``

    :Q: What if it's necessary to know what other instances belong in the replica set?
    :A: The universal identification of an instance is a UUID in
        ``box.info.server.uuid``. The ordinal identification of an instance within
        a replica set is a number in ``box.info.server.id``. To see all the instances
        in the replica set, say: ``box.space._cluster:select{}``. This will return a
        table with all {server.id, server.uuid} tuples for every instance that has
        ever joined the replica set.

    :Q: What if one of the instance's files is corrupted or deleted?
    :A: Stop the instance, destroy all the database files (the ones with extension
        "snap" or "xlog" or ".inprogress"), restart the instance, and catch up
        with the master by contacting it again (just say
        ``box.cfg{...replication=...}``).

    :Q: What if replication causes security concerns?
    :A: Prevent unauthorized replication sources by associating a password with
        every user that has access privileges for the relevant spaces, and every
        user that has a replication :ref:`role <authentication-roles>`. That
        way, the :ref:`URI <index-uri>` for the :ref:`replication
        <cfg_replication-replication>` parameter will always have to have
        the long form ``replication='username:password@host:port'``

    :Q: What if advanced users want to understand better how it all works?
    :A: See the description of instance startup with replication in the
        :ref:`Internals <internals-replication>` section.

=====================================================================
Hands-on replication tutorial
=====================================================================

After following the steps here, an administrator will have experience creating
a replica set and adding a replica.

Start two shells. Put them side by side on the screen. (This manual has a tabbed
display showing "Terminal #1". Click the "Terminal #2" tab to switch to the
display of the other shell.)

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-1

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-1-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-1-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-1-content

        .. container:: b-documentation_tab
            :name: terminal-1-1

            .. code-block:: console

                $

        .. container:: b-documentation_tab
            :name: terminal-1-2

            .. code-block:: console

                $

On the first shell, which we'll call Terminal #1, execute these commands:

.. code-block:: tarantoolsession

    $ # Terminal 1
    $ mkdir -p ~/tarantool_test_node_1
    $ cd ~/tarantool_test_node_1
    $ rm -R ~/tarantool_test_node_1/*
    $ ~/tarantool/src/tarantool
    tarantool> box.cfg{listen = 3301}
    tarantool> box.schema.user.create('replicator', {password = 'password'})
    tarantool> box.schema.user.grant('replicator','execute','role','replication')
    tarantool> box.space._cluster:select({0}, {iterator = 'GE'})

The result is that a new replica set is configured, and the instance's UUID is displayed.
Now the screen looks like this: (except that UUID values are always different):

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-2

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-2-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-2-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-2-content

        .. container:: b-documentation_tab
            :name: terminal-2-1

            .. include:: 1_1.rst

        .. container:: b-documentation_tab
            :name: terminal-2-2

            .. include:: 1_2.rst

On the second shell, which we'll call Terminal #2, execute these commands:

.. code-block:: tarantoolsession

    $ # Terminal 2
    $ mkdir -p ~/tarantool_test_node_2
    $ cd ~/tarantool_test_node_2
    $ rm -R ~/tarantool_test_node_2/*
    $ ~/tarantool/src/tarantool
    tarantool> box.cfg{
             >   listen = 3302,
             >   replication = 'replicator:password@localhost:3301'
             > }
    tarantool> box.space._cluster:select({0}, {iterator = 'GE'})

The result is that a replica is set up. Messages appear on Terminal #1
confirming that the replica has connected and that the WAL contents have
been shipped to the replica. Messages appear on Terminal #2 showing that
replication is starting. Also on Terminal#2 the _cluster UUID values are
displayed, and one of them is the same as the _cluster UUID value that was displayed
on Terminal #1, because both instances are in the same replica set.

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-3

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-3-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-3-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-3-content

        .. container:: b-documentation_tab
            :name: terminal-3-1

            .. include:: 2_1.rst

        .. container:: b-documentation_tab
            :name: terminal-3-2

            .. include:: 2_2.rst

On Terminal #1, execute these requests:

.. code-block:: tarantoolsession

    tarantool> s = box.schema.space.create('tester')
    tarantool> i = s:create_index('primary', {})
    tarantool> s:insert{1, 'Tuple inserted on Terminal #1'}

Now the screen looks like this:

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-4

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-4-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-4-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-4-content

        .. container:: b-documentation_tab
            :name: terminal-4-1

            .. include:: 3_1.rst

        .. container:: b-documentation_tab
            :name: terminal-4-2

            .. include:: 3_2.rst

The creation and insertion were successful on Terminal #1. Nothing has happened
on Terminal #2.

On Terminal #2, execute these requests:

.. code-block:: tarantoolsession

    tarantool> s = box.space.tester
    tarantool> s:select({1}, {iterator = 'GE'})
    tarantool> s:insert{2, 'Tuple inserted on Terminal #2'}

Now the screen looks like this (remember to click on the "Terminal #2" tab when
looking at Terminal #2 results):

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-5

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-5-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-5-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-5-content

        .. container:: b-documentation_tab
            :name: terminal-5-1

            .. include:: 4_1.rst

        .. container:: b-documentation_tab
            :name: terminal-5-2

            .. include:: 4_2.rst

The selection and insertion were successful on Terminal #2. Nothing has
happened on Terminal #1.

On Terminal #1, execute these Tarantool requests and shell commands:

.. code-block:: console

    $ os.exit()
    $ ls -l ~/tarantool_test_node_1
    $ ls -l ~/tarantool_test_node_2

Now Tarantool #1 is stopped. Messages appear on Terminal #2 announcing that fact.
The ``ls -l`` commands show that both instances have made snapshots, which have
similar sizes because they both contain the same tuples.

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-6

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-6-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-6-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-6-content

        .. container:: b-documentation_tab
            :name: terminal-6-1

            .. include:: 5_1.rst

        .. container:: b-documentation_tab
            :name: terminal-6-2

            .. include:: 5_2.rst

On Terminal #2, ignore the error messages,
and execute these requests:

.. code-block:: tarantoolsession

    tarantool> box.space.tester:select({0}, {iterator = 'GE'})
    tarantool> box.space.tester:insert{3, 'Another'}

Now the screen looks like this (ignoring the error
messages):

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-7

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-7-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-7-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-7-content

        .. container:: b-documentation_tab
            :name: terminal-7-1

            .. include:: 6_1.rst

        .. container:: b-documentation_tab
            :name: terminal-7-2

            .. include:: 6_2.rst

Terminal #2 has done a select and an insert, even though Terminal #1 is down.

On Terminal #1 execute these commands:

.. code-block:: tarantoolsession

    $ ~/tarantool/src/tarantool
    tarantool> box.cfg{listen = 3301}
    tarantool> box.space.tester:select({0}, {iterator = 'GE'})

Now the screen looks like this:

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-8

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-8-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-8-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-8-content

        .. container:: b-documentation_tab
            :name: terminal-8-1

            .. include:: 7_1.rst

        .. container:: b-documentation_tab
            :name: terminal-8-2

            .. include:: 7_2.rst

The master has reconnected to the replica set, and has NOT found what the replica
wrote while the master was away. That is not a surprise -- the replica has not
been asked to act as a replication source.

On Terminal #1, say:

.. code-block:: tarantoolsession

    tarantool> box.cfg{
             >   replication = 'replicator:password@localhost:3302'
             > }
    tarantool> box.space.tester:select({0}, {iterator = 'GE'})

The screen now looks like this:

.. container:: b-block-wrapper_doc

    .. container:: b-doc_catalog
        :name: catalog-9

        .. raw:: html

            <ul class="b-tab_switcher">
                <li class="b-tab_switcher-item">
                    <a href="#terminal-9-1" class="b-tab_switcher-item-url p-active">Terminal #1</a>
                </li>
                <li class="b-tab_switcher-item">
                    <a href="#terminal-9-2" class="b-tab_switcher-item-url">Terminal #2</a>
                </li>
            </ul>

    .. container:: b-documentation_tab_content
        :name: catalog-9-content

        .. container:: b-documentation_tab
            :name: terminal-9-1

            .. include:: 8_1.rst

        .. container:: b-documentation_tab
            :name: terminal-9-2

            .. include:: 8_2.rst

    .. raw:: html

        <script>
            register_replication_tab(1);
            register_replication_tab(2);
            register_replication_tab(3);
            register_replication_tab(4);
            register_replication_tab(5);
            register_replication_tab(6);
            register_replication_tab(7);
            register_replication_tab(8);
            register_replication_tab(9);
        </script>

This shows that the two instances are once again in synch, and that each instance
sees what the other instance wrote.

To clean up, say "``os.exit()``" on both Terminal #1 and Terminal #2, and then
on either terminal say:

.. code-block:: console

    $ cd ~
    $ rm -R ~/tarantool_test_node_1
    $ rm -R ~/tarantool_test_node_2
