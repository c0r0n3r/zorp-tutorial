Policy in Practice
==================

Layers
------

Different layers are handled separately in *Zorp*. Lower layer like transport and internet are handled by the kernel module *kZorp* and the application layer is handled by the user space daemon *Zorp*. As different network layer tasks are separated in the level of the operating system there are different administrative tools for them.

Kernel Module
^^^^^^^^^^^^^

There is command line tool named ``kzorp`` which can communicate with the kernel module *kZorp*, upload the objects (for example *zone*, *rule*) of the policy (``policy.py``, ``zones.py``) to the kernel space, download the actually stored objects from the kernel to display them or evaluate the existing *zones*, or *rule* by the given traffic parameters.

.. versionadded:: 5.0

Command line tool ``kzorp`` renamed to ``kzorp-clinet`` as it is a client of the *kZorp* kernel module. Rename of the tool also makes possible avoiding the overload of the *kZorp* concept.

Zorp Instance
^^^^^^^^^^^^^

There are so many control information relates to a *Zorp* user-space daemon instance, like log level, statistical or monitoring information. Instances can be handled by a command line tool named ``zorpctl`` both individually or together.

Zone
----

*Zone* is one of the fundamental types of a *Zorp* policy which can be useful either in a simple case or very sophisticated network topology. Let see how a *zone* hierarchy can be created from scratch.

.. literalinclude:: configs/internet_zones.py
  :language: python


.. literalinclude:: configs/intranet_zones.py
  :language: python
