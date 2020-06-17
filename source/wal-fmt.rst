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
