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
via network message (the full process about message exchnage
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
the WAL is enabled and register the concumer. The consumers makes sure
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
prepares datat to be sent to the remote replica. First we get a read view
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

        // The WAL range (start_vclock; stop_vclock)
        relay_final_join(io->fd, header->sync, &start_vclock, &stop_vclock);

We fetch master's node vclock (the ``replicaset.vclock`` is updated
by WAL engine upon on commit when data is already written to the storage)
and send it out and start sending the vlock range from ``start_vclock``
to ``stop_vclock`` together with rows bound to the range.
