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

Decompressed stream consists of xrow header plus xrow data pairs. Header
represented as

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

