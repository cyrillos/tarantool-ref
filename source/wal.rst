.. vim: ts=4 sw=4 et

Write-ahead logging
===================

Write-ahead logging WAL is one of the most important parts of
databases which implements atomicity and durability. The base
idea of logging is to write the data to the storage before make
this data accessible by others.

We will reference 2.x series of Tarantool source code.

Journal subsystem
-----------------

The journal subsystem provides the following API calls:

 - ``journal_create`` to create a new journal;
 - ``journal_set`` to destroy old journal and set a new one;
 - ``journal_entry_new`` to create a new journal entry;
 - ``journal_write`` to write a journal entry.

There might be only one active journal at a time which is
referred via ``current_journal`` variable (thus we can
create as many journals as we need with ``journal_create``
but only one must be activated via ``journal_set``).

The ``journal_entry_new`` requires a callback to be associated
with the journal entry to be called when entry processing is complete
(with success or failure). Since we use journal to process database
transactions we pass ``txn_entry_complete_cb`` as a callback
and a pointer to the transaction itself as a data.

To explain in details how transactions are processed on journal
level we need to look at transaction structure (short version
to be precise, only with the memebers we're interesed in)
