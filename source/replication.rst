Replication
===========

Introduction
------------

The following sections describe replication procedure.

Replication in 1.10 series
--------------------------

Code flow
~~~~~~~~~

Entry point is in ``box_cfg_xc`` routine which is called once box
configuration initiated (via command line or startup script).

.. code-block:: c

    void
    replication_init(void)
    {
        memset(&replicaset, 0, sizeof(replicaset));
        replica_hash_new(&replicaset.hash);
        rlist_create(&replicaset.anon);
        vclock_create(&replicaset.vclock);
        fiber_cond_create(&replicaset.applier.cond);
        replicaset.replica_by_id = calloc(VCLOCK_MAX, ...);
        latch_create(&replicaset.applier.order_latch);
    }

The ``replicaset`` may carry up to ``VCLOCK_MAX=32`` replicas at once.
This limitation is due to saving network bandwidth when nodes communicate
between each other (same time 32 bits are good length for fast lookup, we
treat is as a bitmap).

Once ``replicaset`` is set we consider bootstrap of a new node, ie assume
there is no recovery from checkpoint procedure needed.

.. code-block:: c

    bootstrap(&instance_uuid, &replicaset_uuid,
              &is_bootstrap_leader);
        box_sync_replication(true);

In ``box_sync_replication`` we try to connect to **master** nodes
to receive updates from.

.. code-block:: c

    box_sync_replication
        appliers = cfg_get_replication(&count);
            // max num of appliers is VCLOCK_MAX;
            for (int i = 0; i < count; i++) {
                // get addresses
                *source = cfg_getarr_elem("replication", i);
                // allocate applier entries dynamically
                applier = applier_new(source, ...);
                    fiber_cond_create(&applier->resume_cond);
                    fiber_cond_create(&applier->writer_cond);
        replicaset_connect(appliers, count, connect_quorum);

Here we fetch replicas addresses from config and allocate ``applier`` array.
The ``applier`` will be notified by ``resume_cond`` and ``writer_cond``
conditions.

Then we initiate the connection procedure.

.. code-block:: c

    replicaset_connect(appliers, count, connect_quorum)
        struct applier_on_connect triggers[VCLOCK_MAX];
        struct replicaset_connect_state state;
        state.connected = state.failed = 0;
        fiber_cond_create(&state.wakeup);
        for (int i = 0; i < count; i++) {
            applier = appliers[i];
            trigger = &triggers[i];
            trigger_create(&trigger->base, applier_on_connect_f, ...);
            trigger->state = &state;
            trigger_add(&applier->on_state, &trigger->base);
            applier_start(applier);
                f = fiber_new_xc(name, applier_f);
                fiber_set_joinable(f, true);
                applier->reader = f;
                fiber_start(f, applier);

On every applier fiber allocated we assign a trigger to track the change of
the applier state which is handled by the ``applier_on_connect_f`` helper.

.. code-block:: c

    applier_on_connect_f(struct trigger *trigger, void *event)
        switch (applier->state) {
        case APPLIER_OFF:
        case APPLIER_STOPPED:
            state->failed++;
            break;
        case APPLIER_CONNECTED:
            state->connected++;
            break;
        default:
            // Not interested in any
            // other events just continue
            // executing an applier fiber.
            return;
        }
        // Notify replicaset_connect_state
        // and pause applier, will be kicked
        // to run explicitly.
        fiber_cond_signal(&state->wakeup);
        applier_pause(applier);

This trigger process ``failed`` and ``connected`` statistics which
is needed to fit the replicas quorum.

Note that ``APPLIER_OFF``, ``APPLIER_STOPPED`` and ``APPLIER_CONNECTED``
states notify the ``replicaset_connect`` waiters. Due to cooperative
multitasking model the ``replicaset_connect`` routine need to waits
for notifications from the applier fibers (described below).

Now back to applier fibers. The ``applier_start`` function creates an
``applier`` fiber and immediately runs ``applier_f`` routine.

.. code-block:: c

    applier_f(va_list ap)
        while (!fiber_is_cancelled()) {
            try {
                applier_connect(applier);
                ...
            } catch() {
                ...
            }
        }

In ``applier_connect`` we send a greeting to the remote node
and fetch UUID of the remote machine (of the **master** node).

.. code-block:: c

    applier_connect
        char greetingbuf[IPROTO_GREETING_SIZE];
        coio_connect(coio, uri, &applier->addr, &applier->addr_len);
        coio_readn(coio, greetingbuf, IPROTO_GREETING_SIZE);
        greeting_decode(greetingbuf, &greeting);
        applier->uuid = greeting.uuid;
        applier->version_id = greeting.version_id;
        if (applier->version_id >= version_id(1, 9, 0)) {
            ...
            xrow_decode_ballot_xc(&row, &applier->ballot);
            ...
        }
        // Notify replicaset waiter that we're done.
        // Then pause and wait for a kick.
        applier_set_state(applier, APPLIER_CONNECTED);
        ...

Once we reach ``APPLIER_CONNECTED`` state the fiber get paused
from ``applier_on_connect_f`` trigger. And left in this state
until being explicitly kicked to run.

The ``replicaset_connect`` caller at this moment is spinning in
a cycle waiting for notification from appliers to fit the quorum
of appliers in connected state.

.. code-block:: c

    replicaset_connect
        ...
        while (state.connected < count) {
            ...
            fiber_cond_wait_timeout(&state.wakeup, timeout);
            ...
        }

Once appliers are in ``APPLIER_CONNECTED`` state we clear ``applier_on_connect_f``
trigger and call ``replicaset_update``. Note that not all appliers might
be connected. Those ones which did not manage to are explicitly stopped,
ie fibers are ripped off.

.. code-block:: c

    replicaset_connect
        ...
        for (int i = 0; i < count; i++) {
            trigger_clear(&triggers[i].base);
            if (applier->state != APPLIER_CONNECTED)
                applier_stop(applier);
                    applier_set_state(applier, APPLIER_OFF);
                    applier->reader = NULL;
        }
        try {
            replicaset_update(appliers, count);
        } catch {}

In ``replicaset_update`` we map appliers to replicas and they
are linked into red-black tree ``uniq`` for fast lookup via their UUID.

.. code-block:: c

    replicaset_update(struct applier **appliers, int count)
        ...
        RLIST_HEAD(anon_replicas);
        replica_hash_new(&uniq);
        ...
        for (int i = 0; i < count; i++) {
            applier = appliers[i];
            replica = replica_new();
                replica = malloc(sizeof(struct replica));
                replica->relay = relay_new(replica);
                    relay = calloc(1, sizeof(struct relay));
                    relay->replica = replica;
                    fiber_cond_create(&relay->reader_cond);
                    relay->state = RELAY_OFF;
                replica->id = 0;
                replica->uuid = uuid_nil;
                replica->applier = NULL;
                trigger_create(&replica->on_applier_state,
                                replica_on_applier_state_f, NULL, NULL);
                replica->applier_sync_state = APPLIER_DISCONNECTED;
                latch_create(&replica->order_latch);
            replica_set_applier(replica, applier);
        ...

We allocate new replica and assign an applier to it.

Note that replica state is driven by ``replica_on_applier_state_f``
trigger. We won't be juming into it right now but the important thing is
that this trigger sends ``fiber_cond_signal(&replicaset.applier.cond)``
to the main replicaset instance.

Now back to the caller, ie ``applier_sync_state``. The replica
instances are created we continue walking over appliers

.. code-block:: c

    replicaset_update(struct applier **appliers, int count)
    ...
        for (int i = 0; i < count; i++) {
            ...
            // continue listing
            replica_set_applier(replica, applier);
            if (applier->state != APPLIER_CONNECTED) {
                // Any not yet connected appliers are
                // chained into anon_replicas list,
                // we will retry to reconnect later.
                rlist_add_entry(&anon_replicas, replica, in_anon);
                continue;
            }
            replica->uuid = applier->uuid;
            replica_hash_insert(&uniq, replica);
        }
        ...

In result we will have two sets, one in ``uniq`` tree which
is intended to keep alive connected replicas and ``anon_replicas``
list which carries not yet connected ones.

Then all alive replicas are marked and connected, statistics updated
and they are moved to global ``replicaset.hash`` tree.

.. code-block:: c

    replicaset_update(struct applier **appliers, int count)
        ...
        replicaset.applier.total = count;
        replicaset.applier.connected = 0;
        replicaset.applier.loading = 0;
        replicaset.applier.synced = 0;
        replica_hash_foreach_safe(&uniq, replica, next) {
            replica_hash_remove(&uniq, replica);
            ...
            replica_hash_insert(&replicaset.hash, replica);
            replica->applier_sync_state = APPLIER_CONNECTED;
            replicaset.applier.connected++;
        }
        ...
        rlist_swap(&replicaset.anon, &anon_replicas);

At the end the anonimous replicas (which is not connected)
are moved to global ``replicaset.anon``. So we have
global ``replicaset`` fully consistent and ready for use.

Now we need to jump up to the initial caller ``bootstrap``.

.. code-block:: c

    bootstrap
        ...
        master = replicaset_leader();
            replicaset_round
                replicaset_foreach(replica) {
                    // Walk over unique replicas
                    // from replicaset hash and
                    // choose one with more advinsed
                    // vclock or one with lowest UUID
            return leader;

We need to find out how exactly we should start, either
we are the master node or we should start from another
node which is choosen as a cluster leader (ie it has
most advansed vclock and low UUID).
