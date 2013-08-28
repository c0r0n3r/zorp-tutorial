Configuration debugging
=======================

After a configuration change there are several cases when the system does not function as expected. Typical problems can be detected with some diagnostics that are described in the following sections.

Obsolete configuration
----------------------

Problem
```````

It is a common problem that an *instance* runs with an obsolete configuration, because you have modified the policy, but forgot to reload the *instance*.

Diagnostics
```````````

The actual status of a *Zorp* instance can be displayed by using ``zorpctl`` with the ``status`` command. With the ``instance_name`` argument, only the status of the given *instance* will be printed, otherwise ``zorpctl`` prints the state of each and every instance.

.. code:: bash

  zorpctl status [instance_name]

The output of the ``zorpctl status`` command is the following. In case of the first line, the instance is running with an obsolete configuration, which means that the *policy* file is newer than the running *instance*. In case of the second line, the instance is not running at all, which typically means that because of a configuration problem, the instance could not be started.

::

  Process http#0: running, policy NOT reloaded, 5 threads active, pid 2180
  Process http#0: not running

Solution
````````

If the only problem is that you forgot to reload the *instance*, run the ``zorpctl reload`` command with the name of the instance as the argument. If the *instance* is not running, it can be started with the ``zorpctl start`` command.

.. code:: bash

  zorpctl reload instance_name
  zorpctl start instance_name

.. IMPORTANT:: Any changes to the ``instances.conf`` require *restart*.

.. CAUTION:: Instance *restart* aborts the active connection.

.. WARNING:: Active connections are not affected by *reload*. The traffic that was accepted by the *policy* before the *reload*, but is now rejected by the actual *policy*, is not aborted by the *reload*.

Result
``````

After *reload* or *start*, something similar to the following status report is displayed. The process identifier (pid) will be the same if an instance *reload* had been accomplished, but should be different after a *restart*.

::

  Process http#0: running, 9 threads active, pid 19302

Configuration reload
--------------------

Problem
```````

In exceptional cases it might happen that the *policy* in the user space (*Zorp*) and the *policy* in the kernel space (*KZorp*) differs from each other.

Diagnostics
```````````

The configuration that is actually stored in *KZorp* can be dumped by using the ``kzorp`` tool.

.. code:: bash

  kzorp --dispatchers
  kzorp --zones
  kzorp --services

The certain configuration items (dispatchers, zones, services) can be dumped separately or at the same time.

.. code:: bash

  kzorp -dzs

The dumped configuration should be the same that is stored in the configuration file, assuming that the *instances* run the actual configuration.

Solution
````````

The solution is the same that it was in case of `obsolete configuration`_.

Result
``````

After the *reload* or *restart*, the *policy* in the user space (*Zorp*) and the *policy* in the kernel space (*KZorp*) must be the same.

Missing zone
------------

Problem
```````

A typical situation is when a *zone* in a newly created *rule* in the *policy* does not match the actual traffic.

Diagnostics
```````````

If you know that the parameters of the traffic cause the problem, it can be evaluated by *KZorp* to decide which *zone* matches in the policy by using the ``kzorp`` command line tool.

.. code:: bash

  kzorp -e tcp 10.10.0.1 1.2.3.4 eth1 --src-port 1234 --dst-port 21

The most commonly encountered problem is that there is no *zone* that matches the traffic, or an unexpected *zone* matches instead. In this case the result of the evaluation will be similar to the following:

::

  evaluating tcp 10.10.0.1:1234 -> 1.2.3.4:21 on eth1
  Client zone: not found <1>
  Server zone: unexpected.zone <2>
  Service: ftp/ftp_readonly
  Dispatcher: ftp/dsp/dispatch:0

1. The *zone* contains the source address (2nd argument of ``evaluate`` option)
2. The *zone* contains the destination address (3rd argument of ``evaluate`` option)

.. NOTE:: If there is no *zone* in the *policy*, the *Client zone* and *Server zone* will always be *not found*.

Solution
````````

If *zone* is not found, add a new *zone*, or extend an existing one with a small subnetwork (for example ``10.10.0.0/16``), or add a *zone* (for example with the name *internet*) with the largest possible subnetworks (``0.0.0.0/0`` and/or ``::0/0``), which means a fallback if there is no other matching *zone*.

If another *zone* was found instead of what was expected there are two possibilities:

1. The subnetwork in the expected *zone* is too small and the IP address of the traffic is outside the subnetwork and also the *zone*. In this case

  * increase the size of the subnetwork by decreasing its prefix to make it large enough to contain the IP address (for example ``10.10.0.0/16`` instead of ``10.10.10.0/24``) or
  * add another subnetwork to the *zone* that contains the IP address (for example ``10.10.20.0/24`` or ``10.10.20.30/32``) or
  * add another *zone* to the *rule* that already contains the IP address

2. The expected *zone* is too large and there is another *zone* with a more specific subnetwork. In this case

  * set the actually matching *zone* as the child of the expected *zone*, but keep in mind that any other *rule* with the parent *zone* will apply to this *zone* from now on as well.
  * add a more specific subnetwork to the expected *zone* that is in the actually matching *zone*

Result
``````
After fixing the subnetworks and the hierarchy of *zone* s, the result of the evaluation should contain the expected *zone* s.

::

  evaluating tcp 10.10.0.1:1234 -> 1.2.3.4:21 on eth1
  Client zone: intranet.devel
  Server zone: internet
  Service: ftp/ftp_readwrite
  Dispatcher: ftp/dsp/dispatch:0

Missing dispatcher
------------------

Problem
```````

Despite the fact that the *zone* s are the expected *zone* s or we do not have any *zone* in our *policy*, it may happen that no *dispatcher* is found because there are other conditions in the *rule* that do not match.

Diagnostics
```````````

The diagnostics method is the same that it was `last time <missing zone_>`_ in the case of missing *zone* s.

.. code:: bash

  kzorp -e tcp 10.10.0.1 1.2.3.4 eth1 --src-port 1234 --dst-port 21

In this case, the *zone* s are what were expected, but neither the *dispatcher*, nor the *service* are found.

::

  evaluating tcp 10.10.0.1:1234 -> 1.2.3.4:21 on eth1
  Client zone: expected.zone
  Server zone: expected.zone
  Service: not found <1>
  Dispatcher: not found <2>

1. The *service* that was started by the *rule* that matches the given traffic.
2. The *dispatcher* that matches the traffic.

Solution
````````

Consider other conditions of the *rule*. They are ordered so that the first is the most probable to cause the problem.

1. Check the source and destination subnetwork conditions in the *rule* (``src_subnet``, ``dst_subnet``) in the same way that you did in case of missing *zone*.
2. Check the source interface condition in the *rule* (``src_iface``), also check (for example with *tcpdump*) the traffic that is actually on this interface.
3. Check the source and destination port in the *rule* (``src_port``, ``dst_port``), especially port ranges.
4. Check the protocol number.

Result
``````

After the fix, *service* and *dispatcher* in the evaluation should contain the the expected ones.

::

  evaluating tcp 10.10.0.1:1234 -> 1.2.3.4:21 on eth1
  Client zone: not found
  Server zone: not found
  Service: ftp/ftp_readwrite
  Dispatcher: ftp/dsp/dispatch:0


Disappearing traffic
--------------------

Problem
```````

Everything seems fine, *policy* is up-to-date in *Zorp*, evaluation result is correct, but *service* does not start.

Diagnostics
```````````

Add Netfilter rules to the ``raw`` table, which makes possible to trace the route of the desired traffic in IPTables. If the traffic in question is *TCP*, where the destination is ``1.2.3.4:21`` use the following commands.

.. code:: bash

  iptables -A PREROUTING -t raw -p tcp -d --dport 21 -j TRACE
  iptables -A OUTPUT -t raw -p tcp -d --dport 21 -j TRACE

.. NOTE:: Do not forget to load ``ipt_LOG`` module with the command ``modprobe ipt_LOG``.

.. TIP:: You can prefix the generated log by appending ``--log-prefix "some prefix"``, which makes it easy to find them in your log.

Solution
````````

Follow the route of the traffic and find the last Netfilter rule where it appears. Depending on the type of the rule you can modify your Netfilter policy (for example found rule jumps to *DROP* target), or continue debug in *KZorp* as you can read in the :ref:`kernel-debugging` section.

Result
``````

After finding the problematic Netfilter rule, the *service* functions as expected.
