.. vim: ts=4 sw=4 et

WAL file binary format
======================

General overview
----------------

The WAL file starts with ``meta`` information block following by ``xrow`` entries.

 +--------------+
 |  meta block  |
 +--------------+
 |  xrows block |
 +--------------+
 |  xrows block |
 +--------------+

Meta information block
----------------------

The ``meta`` block ends up by ``\n\n`` marker. Each entry inside the block
is ``\n`` separated records. The block is started with obligatory ``signature``,
``version`` followed by ``key:value`` array.

 +-------------+
 |  signature  |
 +-------------+
 |  version    |
 +-------------+
 |  key: value |
 +-------------+

 The ``signature`` either ``XLOG`` for regular WAL files or ``SNAP`` for
 snapshots. For example an empty WAL file might look like

 +-------------------------------------------------------+
 | ``XLOG\n``                                            |
 +-------------------------------------------------------+
 |  ``0.13\n``                                           |
 +-------------------------------------------------------+
 | ``Version: 2.5.0-136-gef86e3c99\n``                   |
 +-------------------------------------------------------+
 | ``Instance: 123a579d-4994-4e7c-a52f-6b576a8988d7\n``  |
 +-------------------------------------------------------+
 | ``VClock: {1: 8}\n``                                  |
 +-------------------------------------------------------+
 | ``PrevVClock: {}\n``                                  |
 +-------------------------------------------------------+
 | ``\n``                                                |
 +-------------------------------------------------------+

 Note the double ``\n\n`` formed by last entry and ``PrevVClock`` entry.
 This represents the end of meta information block.

Data blocks (xrows)
-------------------

The meta block carries only service information about content of the
WAL file. The real data appended into the WAL file in xrows records.
Structure of each record is the following

+--------------+
| fixed header |
+--------------+
| xrows header |
+--------------+
|  xrows data  |
+--------------+
| xrows header |
+--------------+
|  xrows data  |
+--------------+
|     ...      |
+--------------+

The ``fixed header`` defined as

.. code-block:: c

    struct xlog_fixheader {
        log_magic_t magic;
        uint32_t    crc32p;
        uint32_t    crc32c;
        uint32_t    len;
        // padding to 19 bytes
    };

``magic`` is one of the following constants

+----------------+-------------------+
| ``0xd5ba0bab`` | uncompressed rows |
+----------------+-------------------+
| ``0xd5ba0bba`` | compressed rows   |
+----------------+-------------------+
| ``0xd510aded`` | end of file       |
+----------------+-------------------+

The ``xlog_fixheader`` should have length of 19 bytes on disk so right
afther the structure the padding is pushed. ``magic`` points how xrows
are kept - either they are compressed (with zstd compression library)
or not.

Aside from this the ``magic`` may have end of file represeting that there
is no more data. Thus when read ``xlog_fixheader`` structure from disk
we need to read 4 bytes first and make sure it is not EOF value.

The ``len`` value stores the number of bytes occupied by xrow headers
with data. If xrows are compressed we should read ``len`` bytes and
decompress them.

Decompressed stream consists of xrow header plus xrow data pairs.
Header represented as

.. code-block:: c

    struct xrow_header {
        uint32_t        type;
        uint32_t        replica_id;
        uint32_t        group_id;
        uint64_t        sync;
        int64_t         lsn;
        double          tm;
        int64_t         tsn;
        bool            is_commit;
        int             bodycnt;
        uint32_t        schema_version;
        struct iovec    body[XROW_BODY_IOVMAX];
    };

Each ``xrow_header`` describes the ``body`` data, ie the requests to be
executed (like insert, update and etc). Note that ``xlog_fixheader::len``
may cover not one ``xrow_header`` but a series of headers and xrow requests.

Tarantool provides a ``tarantoolctl`` tool to show the content of files
in a human readable form.

.. code-block:: shell

    $./src/tarantool ./extra/dist/tarantoolctl cat --show-system ../dumper/examples/00000000000000000008.snap
    ---
    HEADER:
      timestamp: 1592050034.6865
      type: INSERT
    BODY:
      space_id: 272
      tuple: ['cluster', '442100e0-b90f-4d5b-92d7-98ad52ed4919']
    ---
    HEADER:
      lsn: 1
      type: INSERT
      timestamp: 1592050034.6865
    BODY:
      space_id: 272
      tuple: ['max_id', 512]
    ...
    HEADER:
      lsn: 517
      type: INSERT
      timestamp: 1592050034.6865
    BODY:
      space_id: 512
      tuple: [2, 'Scorpions', 2015]
    ---
    HEADER:
      lsn: 518
      type: INSERT
      timestamp: 1592050034.6865
    BODY:
      space_id: 512
      tuple: [3, 'Ace of Base', 1993]

Another example is more detailed example for same file

.. code-block:: shell

    $ /ttdump examples/00000000000000000008.snap

    meta: 'SNAP'
    meta: '0.13'
    meta: 'Version: 2.5.0-136-gef86e3c99'
    meta: 'Instance: 123a579d-4994-4e7c-a52f-6b576a8988d7'
    meta: 'VClock: {1: 8}'
    fixed header
    -------
      magic 0xba0bbad5 crc32p 0 crc32c 0x66db6462 len 6055
    -------
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 0 tm 1.592e+09 tsn 0 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 55
    -------
    key: 0x10 'space id' value: 272
    key: 0x21 'tuple' value: {cluster, 442100e0-b90f-4d5b-92d7-98ad52ed4919}
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 1 tm 1.592e+09 tsn 1 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 19
    -------
    key: 0x10 'space id' value: 272
    key: 0x21 'tuple' value: {max_id, 512}
    ...
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 515 tm 1.592e+09 tsn 515 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 48
    -------
    key: 0x10 'space id' value: 320
    key: 0x21 'tuple' value: {1, 123a579d-4994-4e7c-a52f-6b576a8988d7}
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 516 tm 1.592e+09 tsn 516 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 21
    -------
    key: 0x10 'space id' value: 512
    key: 0x21 'tuple' value: {1, Roxette, 1986}
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 517 tm 1.592e+09 tsn 517 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 23
    -------
    key: 0x10 'space id' value: 512
    key: 0x21 'tuple' value: {2, Scorpions, 2015}
    xrow header
    -------
      type 0x2 (INSERT) replica_id 0 group_id 0 sync 0 lsn 518 tm 1.592e+09 tsn 518 is_commit 1 bodycnt 1 schema_version 0x4027e6
        iov: len 25
    -------
    key: 0x10 'space id' value: 512
    key: 0x21 'tuple' value: {3, Ace of Base, 1993}
    -------
