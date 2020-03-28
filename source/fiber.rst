.. vim: ts=4 sw=4 et

Fibers
======

Introduction
------------

Tarantool uses cooperative multitasking via that named fibers.
They are similar to well known threads except they are not running
simultaneously but must be scheduled explicitly.

Fibers are coupled into cords (in turn the cords are real threads,
thus fibers inside a cord are executing in a sequent way while several
cords may run simultaneously).

When we create a fiber we provide a function to be invoked upon fiber's
start, lets call it a fiber function to distinguish from other
service routines.

Event library
-------------

The heart of Tarantool fibers schedule is ``libev`` library. As its
name implies the library provides an loop which get interrupted when an
event arrive. We won't jump into internals of ``libev`` but provide
only a few code snippets to draw a basic workflow

.. code-block:: c

    ev_run (int flags) {
        do {
            //...
            backend_poll(waittime);
            //...
            EV_INVOKE_PENDING;
        }
    }

The ``backend_poll`` invokes low level service call (such as ``epoll`` or
``select`` depending on operating system, please see their man pages)
to fetch events. Evens are explicitly passed to queue via API ``libev``
provides to programmers. The service call supports ``waittime`` timeout,
so if no events available a new wait iteration started or polling to be precise.
The typical minimal ``waittime`` is 1 millisecond.

The ``EV_INVOKE_PENDING`` fetches events to process and runs associated
callbacks for this iteration. Once callbacks are finished a new iteration
begins.

Pushing new event into the queue is provided by ``ev_feed_event`` call
and Tarantool use it under the hood.

Fibers engine
-------------

The main cord defined as

.. code-block:: c

    static struct cord main_cord;
    __thread struct cord *cord_ptr = NULL;
    #define cord()    cord_ptr
    #define fiber()   cord()->fiber
    #define loop()    (cord()->loop)

Note the ``cord()``, ``fiber()`` and ``loop()`` helper macros.
They are used frequently to access currently executing cord,
fiber and event loop.

The cord structure is the following (note that we are posting stripped
versions of structures in sake of simplicity)

.. code-block:: c

    struct cord {
        // Running fiber
        struct fiber        *fiber;
        // Libev loop
        struct ev_loop      *loop;
        // Fiber's ID map
        struct mh_i32ptr_t  *fiber_registry;
        // All fibers
        struct rlist        alive;
        // Fibers ready for execution
        struct rlist        ready;
        // Fibers for recycling
        struct rlist        dead;
        // Scheduler
        struct fiber        sched;
        // Cord name
        char                name[FIBER_NAME_MAX];
    }

The most important members are:
 - ``fiber`` is a currently executing fiber
 - ``sched`` is a service fiber which schedules all other fibers in the cord

The fiber structure is the following

.. code-block:: c

    struct fiber {
        // The fiber to be scheduled
        // when this one yields
        struct fiber    *caller;
        // Fiber ID
        uint32_t        fid;
        // To link with cord's
        // @alive or @dead lists
        struct rlist    link;
        // To link with cord's @ready list
        struct rlist    state;
        // Fibers waiting for this
        // instance to finish.
        struct rlist    wake;
        // Fiber function, its
        // arguments and return code
        fiber_func      f;
        va_list         f_data;
        int             f_ret;
    }

When Tarantool starts it creates the main cord

.. code-block:: c

    main(int argc, char **argv)
        fiber_init(fiber_cxx_invoke);
            fiber_invoke = fiber_cxx_invoke;
            main_thread_id = pthread_self();
            main_cord.loop = ev_default_loop();
            cord_create(&main_cord, "main");

Don't pay attention on ``fiber_cxx`_invoke`` for now, it is just
a wrapper to run a fiber function.

The cord creation is the following

.. code-block:: c

    cord_create(&main_cord, "main");
        cord() = cord;
        cord->id = pthread_self();
        rlist_create(&cord->alive);
        rlist_create(&cord->ready);
        rlist_create(&cord->dead);
        cord->fiber_registry = mh_i32ptr_new();
        cord->sched.fid = 1;
        fiber_set_name(&cord->sched, "sched");
        cord->fiber = &cord->sched;
        ev_async_init(&cord->wakeup_event, fiber_schedule_wakeup);
        ev_idle_init(&cord->idle_event, fiber_schedule_idle);

When the cord is created the *scheduler fiber* ``cord->sched``
becomes its primary one. Think of it as a main fiber which will
switch all other fibers in this cord.

Note that here we setup ``cord()`` macro to point to ``main_cord``,
thus ``fiber()`` will point to main cord scheduler fiber and
``loop()`` will be ``ev_default_loop``.

Abstract description is not very usefull so lets look how Tarantool
boots in interactive console mode (the mode is not really important
here but rather a call graph).

.. code-block:: c

    main
        fiber_init(fiber_cxx_invoke);
        tarantool_lua_run_script
            script_fiber = fiber_new(title, run_script_f);
                fiber_new_ex
                    cord = cord();
                    fiber = mempool_alloc()
                    coro_create(..., fiber_loop,...)
                    rlist_add_entry(&cord->alive, fiber, link);
                    register_fid(fiber);

Here we create a new fiber to run ``run_script_f`` fiber
function. ``fiber_new`` allocates a new fiber instance
(actually there is a fiber cache so that if a previous fiber
already finished its work and exited we can reuse it without
calling ``mempool_alloc`` but this is just an optimization
for speed sake), then we chain it into the main cord's
``alive`` list and register in fiber IDs pool.

One of the clue here is ``coro_create`` call, where "coro"
stands for "coroutine". Coroutines are implemented via ``coro``
library. On Linux it simply handles hardware context to reload
registers and jump into desired function. More precisely the heart of "coro"
library is ``coro_transfer(&from, &to)`` routine which remembers current
point of execution (``from``) and transfer flow to the new instruction
pointer provided (``to`` which is created during ``coro_create``).

Note that the fiber function is wrapped by ``fiber_loop``.
This is because the fiber function itself may not call scheduler
explicitly but we have to pass execution to others fibers, thus
we simply call fiber function manually inside ``fiber_loop``
and reschedule then.

.. code-block:: c

    fiber_loop(MAYBE_UNUSED void *data)
        ...
        fiber->f_ret = fiber_invoke(fiber->f, fiber->f_data);
        fiber->flags |= FIBER_IS_DEAD;
        while (!rlist_empty(&fiber->wake)) {
            ...
            fiber_wakeup(f);
                ...
                rlist_move_tail_entry(&cord->ready, f, state);
                ...
            ...
        }
        if (!(fiber->flags & FIBER_IS_JOINABLE))
            fiber_recycle(fiber);
        fiber->f = NULL;
        fiber_yield();

Some fibers may wait for others to be finished, for this sake we
move them to ``ready`` list of the cord first, then we try to
put the fiber into a cache pool to recycle it (thus don't allocate
memory again) via ``fiber_recycle`` and finally we move execution
flow back to the scheduler fiber via ``fiber_yield``.

Fibers do not start execution automatically, for this sake we have
to call ``fiber_start``. Thus back to Tarantool startup

.. code-block:: c

    tarantool_lua_run_script
        script_fiber = fiber_new(title, run_script_f);
        fiber_start(script_fiber, ...)
            fiber_call(...)
                fiber_call_impl(...)
                    coro_transfer(...)
        ev_run(loop(), 0);

Here once the fiber is created we kick it to execute. This is done
indside ``fiber_call_impl` which uses ``core_transfer``
routine to jump into ``fiber_loop`` and invoke ``run_script_f``
inside.

The ``run_script_f`` shows a good example how to give execution
back to scheduler fiber and continue

.. code-block:: c

    run_script_f
        ...
        fiber_sleep(0.0);
        ...

When ``fiber_sleep`` is called the ``coro`` switch execution
to the scheduler fiber

.. code-block:: c

    fiber_sleep(double delay)
        ...
        fiber_yield_timeout(delay);
            ...
            fiber_yield();
                cord = cord();
                caller->caller = &cord->sched;
                coro_transfer(&caller->ctx, &callee->ctx);

Once ``coro`` jumped into scheduler fiber another fiber is
choosen to execute. At some moment scheduler return execution
to the point after ``fiber_sleep(0.0)`` and we step up back
to ``tarantool_lua_run_script`` and run main event loop
``ev_run(loop(), 0)``. Now all future execution will be driven
by ``libev`` and by events we supply into the queue.

The full description of the fiber API is provided in Tarantool
manual but we mention a few just to complete this introduction:

 - ``cord_create`` to create a new cord;
 - ``fiber_new`` to create a new fiber but not run it;
 - ``fiber_start`` to execute a fiber immediately;
 - ``fiber_cancel`` to cancel execution of a fiber;
 - ``fiber_join`` to wait for a cancelled fiber;}
 - ``fiber_yield`` to switch execution to another fiber,
   the execution will back to the point after this call later.
   By later we mean that some other fiber will call ``fiber_wakeup``
   on this fiber, until then it won't be scheduled. This is the key
   function of fibers switch;
 - ``fiber_sleep`` to sleep some time giving execution
   to another fibers;
 - ``fiber_yield_timeout`` to give execution to another
   fibers with some timeout value;
 - ``fiber_reschedule`` give execution to another fibers.
   In contrast with plain ``fiber_yield`` we are moving self
   to the end of cord's ``ready`` list. We will grab execution
   back when all fibers already waiting for execution are
   processed.
