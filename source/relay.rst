Relay
=====

Introduction
------------

Relay serves as a counterpart to replication procedure.

When replication is set in the box configuration we connect
to the remote peer from which we will fetch database updates,
to keep all nodes with data up to day.

Same time the local changes made on the node should be
sent out to the remote peers. For this sake we run
the relay service.

Relay start up
--------------

When remote node connects to us it passes ``IPROTO_JOIN`` code
via network message (the full process about message exchange
is described in replication section)

.. code-block:: c

    static void
    tx_process_replication(struct cmsg *m)
    {
        ...
        coio_create(&io, con->input.fd);
        switch (msg->header.type) {
        case IPROTO_JOIN:
            box_process_join(&io, &msg->header);
            break;

First we start the joining procedure

.. code-block:: c

    void
    box_process_join(struct ev_io *io, struct xrow_header *header)
    {
        ...
        struct tt_uuid instance_uuid = uuid_nil;
        xrow_decode_join_xc(header, &instance_uuid);
        ...
        struct replica *replica = replica_by_uuid(&instance_uuid);
        if (replica == NULL || replica->id == REPLICA_ID_NIL) {
            box_check_writable_xc();
            struct space *space = space_cache_find_xc(BOX_CLUSTER_ID);
            access_check_space_xc(space, PRIV_W);
        }

        if (wal_mode() == WAL_NONE) {
            tnt_raise(ClientError, ER_UNSUPPORTED, ...);
        }

        struct gc_consumer *gc = gc_consumer_register(&replicaset.vclock,
            "replica %s", tt_uuid_str(&instance_uuid));
        auto gc_guard = make_scoped_guard([&] { gc_consumer_unregister(gc); });
        ...

Here we test if replica has rights to join the cluster, make sure that
the WAL is enabled and register the consumer. The consumers makes sure
that WAL file is not rotated while there are some records which are not
yet propagated to the whole cluster.

Then  we create the relay instance itself

.. code-block:: c

    void
    box_process_join(struct ev_io *io, struct xrow_header *header)
    {
        ...
        struct vclock start_vclock;
        relay_initial_join(io->fd, header->sync, &start_vclock);
            // relay_initial_join code
            struct relay *relay = relay_new(NULL);
                struct relay *relay = (struct relay *)calloc(1, ...);
                relay->replica = replica;
                relay->last_row_time = ev_monotonic_now(loop());
                fiber_cond_create(&relay->reader_cond);
                diag_create(&relay->diag);
                stailq_create(&relay->pending_gc);
                relay->state = RELAY_OFF;
            relay_start(relay, fd, sync, relay_send_initial_join_row);
                ...
                relay->state = RELAY_FOLLOW;
            ...
            engine_prepare_join_xc(&ctx);
            ...
            wal_sync(vclock);
            ...
            xrow_encode_vclock_xc(&row, vclock);
            ...
            coio_write_xrow(&relay->io, &row);
            engine_join_xc(&ctx, &relay->stream);

The ``relay_send_initial_join_row`` creates new relay structure then
prepares data to be sent to the remote replica. First we get a read view
from engine, then check that there is rollback in progress in WAL engine
and fetch the latest vclock from it. Then we encode the tuples which
are matched the WAL record and send them out to remote replica.

Note the ``relay_initial_join`` frees ``relay`` upon completion.

Then we continue joining procedure

.. code-block:: c

    void
    box_process_join(struct ev_io *io, struct xrow_header *header)
    {
        ...
        // Check for replicaid or register new one
        box_on_join(&instance_uuid);
        ...
        // Master's vclock
        struct vclock stop_vclock;
        vclock_copy(&stop_vclock, &replicaset.vclock);

        // Send it to the peer
        struct xrow_header row;
        xrow_encode_vclock_xc(&row, &stop_vclock);
        row.sync = header->sync;
        coio_write_xrow(io, &row);

        // The WAL range (start_vclock; stop_vclock) with rows
        relay_final_join(io->fd, header->sync, &start_vclock, &stop_vclock);

        // End of WAL marker
        xrow_encode_vclock_xc(&row, &replicaset.vclock);
        row.sync = header->sync;
        coio_write_xrow(io, &row);

        // Advance the consumer position
        gc_consumer_advance(gc, &stop_vclock);

We fetch master's node vclock (the ``replicaset.vclock`` is updated
by WAL engine upon on commit when data is already written to the storage)
and send it out. Then we send the vlock range from ``start_vclock``
to ``stop_vclock`` together with rows bound to the range and end it
sending end of WAL marker.

The ``relay_final_join`` is a bit tricky

.. code-block:: c

    void
    relay_final_join(int fd, uint64_t sync, struct vclock *start_vclock,
                     struct vclock *stop_vclock)
    {
        struct relay *relay = relay_new(NULL);

        relay_start(relay, fd, sync, relay_send_row);
        auto relay_guard = make_scoped_guard([=] {
            relay_stop(relay);
            relay_delete(relay);
        });

        relay->r = recovery_new(cfg_gets("wal_dir"), false,
                                start_vclock);
        vclock_copy(&relay->stop_vclock, stop_vclock);

        int rc = cord_costart(&relay->cord, "final_join",
                              relay_final_join_f, relay);
        if (rc == 0)
            rc = cord_cojoin(&relay->cord);
        if (rc != 0)
            diag_raise();
    }

It runs ``relay_final_join_f`` in a separate thread waiting for
its completion. This function runs ``recover_remaining_wals``
which scans the WAL files (they can rotate) for rows associated
with ``{start_vclock; stop_vclock}`` range and send them all to
the remote peer.

After this stage our node is joined and we need to wait for
subscribe request from remote peer. Once received we prepare
our node to send local updates to the peer.

.. code-block:: c

    static void
    tx_process_replication(struct cmsg *m)
    {
        ...
        switch (msg->header.type) {
        ...
        case IPROTO_SUBSCRIBE:
            box_process_subscribe(&io, &msg->header);
            break;
        ...

The ``box_process_subscribe`` never returns but rather watches
for local changes and sends them up.

.. code-block:: c

    void
    box_process_subscribe(struct ev_io *io, struct xrow_header *header)
    {
        ...
        // Fetch vclock of the remote peer
        vclock_create(&replica_clock);
        xrow_decode_subscribe_xc(header, NULL, &replica_uuid, &replica_clock,
                                 &replica_version_id, &anon, &id_filter);
        ...
        //
        // Remember current WAL clock
        vclock_create(&vclock);
        vclock_copy(&vclock, &replicaset.vclock);

        // Send it to the peer
        struct xrow_header row;
        xrow_encode_subscribe_response_xc(&row, &REPLICASET_UUID, &vclock);

        // Set 0 component to ours 0 component value
        vclock_reset(&replica_clock, 0, vclock_get(&replicaset.vclock, 0));

        // Initiate subscription procedure
        relay_subscribe(replica, io->fd, header->sync, &replica_clock,
                        replica_version_id, id_filter);
    }

Everything should be clear from code comments except dropping
0th component. FIXME: describe why 0th component is so important.

The subscription routine runs until explicitly canceled

.. code-block:: c

    void
    relay_subscribe(struct replica *replica, int fd, uint64_t sync,
                    struct vclock *replica_clock, uint32_t replica_version_id,
                    uint32_t replica_id_filter)
    {
        struct relay *relay = replica->relay;
        relay_start(relay, fd, sync, relay_send_row);
        ...

        vclock_copy(&relay->local_vclock_at_subscribe, &replicaset.vclock);
        relay->r = recovery_new(cfg_gets("wal_dir"), false, replica_clock);
        vclock_copy(&relay->tx.vclock, replica_clock);
        ...
        int rc = cord_costart(&relay->cord, "subscribe",
                              relay_subscribe_f, relay);
        if (rc == 0)
            rc = cord_cojoin(&relay->cord);
        ...
    }

The ``relay->r = recovery_new`` provides us access to the WAL files
while ``relay_subscribe_f`` runs inside a separate thread.

.. code-block:: c

    static int
    relay_subscribe_f(va_list ap)
    {
        struct relay *relay = va_arg(ap, struct relay *);
        struct recovery *r = relay->r;

        coio_enable();
        relay_set_cord_name(relay->io.fd);

        /* Create cpipe to tx for propagating vclock. */
        cbus_endpoint_create(&relay->endpoint,
                tt_sprintf("relay_%p", relay),
                fiber_schedule_cb, fiber());
        cbus_pair("tx", relay->endpoint.name, &relay->tx_pipe,
                  &relay->relay_pipe, NULL, NULL, cbus_process);
        ...

        /* Setup WAL watcher for sending new rows to the replica. */
        wal_set_watcher(&relay->wal_watcher, relay->endpoint.name,
                        relay_process_wal_event, cbus_process);

        /* Start fiber for receiving replica acks. */
        char name[FIBER_NAME_MAX];
        snprintf(name, sizeof(name), "%s:%s", fiber()->name, "reader");
        struct fiber *reader = fiber_new_xc(name, relay_reader_f);
        fiber_set_joinable(reader, true);
        fiber_start(reader, relay, fiber());

        /*
         * If the replica happens to be up to date on subscribe,
         * don't wait for timeout to happen - send a heartbeat
         * message right away to update the replication lag as
         * soon as possible.
         */
        relay_send_heartbeat(relay);
        ...
    }

First we create ``relay->endpoint`` endpoint and pair it with
``tx`` endpoint (the ``tx`` endpoint comes from net thread
spinning inside ``net_cord_f``). Once paired we will have
``relay->tx_pipe`` which responsible to notify ``tx`` thread
to send out the data we provide, and ``relay->relay_pipe``
which notifies relay thread from ``tx`` thread side.

Then we setup a watcher on WAL changes. On every new commit
the ``relay_process_wal_event`` will be called which calls
the ``recover_remaining_wals`` helper to advance xlog cursor
in the WAL file and send new rows to the remote replica.

The reader of new Acks coming from remote node is implemented
via ``relay_reader_f`` fiber. The one of the key moment is
that all replicas are sending heartbeat messages each other
pointing that they are alive.

Relay lifecycle
---------------

.. code-block:: c

    static int
    relay_subscribe_f(va_list ap)
    {
        ...
        while (!fiber_is_cancelled()) {
            //
            // Wait for incoming data from remote
            // peer (it is Ack/Heartbeat message)
            double timeout = replication_timeout;
            fiber_cond_wait_deadline(&relay->reader_cond,
                                     relay->last_row_time + timeout);
            ...
            //
            // Send the heartbeat packet if needed
            if (ev_monotonic_now(loop()) - relay->last_row_time > timeout)
                relay_send_heartbeat(relay);

            //
            // Make sure that vlock has been updated
            // and the previous status is delievered.
            if (relay->status_msg.msg.route != NULL)
                continue;

            struct vclock *send_vclock;
            if (relay->version_id < version_id(1, 7, 4))
                send_vclock = &r->vclock;
            else
                send_vclock = &relay->recv_vclock;
            //
            // Nothing to do
            if (vclock_sum(&relay->status_msg.vclock) == vclock_sum(send_vclock))
                continue;

            static const struct cmsg_hop route[] = {
                {tx_status_update, NULL}
            };

            cmsg_init(&relay->status_msg.msg, route);
            vclock_copy(&relay->status_msg.vclock, send_vclock);
            relay->status_msg.relay = relay;
            cpipe_push(&relay->tx_pipe, &relay->status_msg.msg);

            /* Collect xlog files received by the replica. */
            relay_schedule_pending_gc(relay, send_vclock);
        }
        ...
    }

As expected we wait for heartbeat packet from remote peer first
(the ``relay_reader_f`` will wake us up via ``relay->reader_cond``).
Then we send our own heartbeat message if needed. And finally
we send the last received vclock from the remote peer. Same
time we notify xlog engine about wal files we no longer need
since they are propagated.

Note that WAL commits runs ``relay_process_wal_event`` by
self, still the even is delivered to main event loop and then
to the relay thread.
