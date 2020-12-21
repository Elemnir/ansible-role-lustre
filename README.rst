======================
 Ansible Role: lustre
======================

This ansible role installs and configures both the Server and Client sides of a Lustre filesystem. Installation is accomplished via RPM-based repositories to allow for custom builds, but the defaults install Lustre 2.12.5 targetting a CentOS 7.8 system from the WhamCloud-provided repositories. LNET configuration utilizes the `lnet.service` systemd service, as well as configures the `/etc/lnet.conf` configuration file to define networks.

----------------
 Role Variables
----------------

Key role variables are documented with their default values below. See ``defaults/main.yml`` for a full list.

::

    lustre_run_server: False
    lustre_run_client: False

These variables control which components of the Lustre stack to install on a system. One or the other must be set to True for the role to function correctly.

::

    lustre_lnet_nets:
      - name: tcp1
        interface: eth0

This list of dictionaries defines the LNET networks to configure on the system. Currently no tuning parameters or multiple interfaces per network are supported. 

::

    lustre_fsname: "testfs"

The name of the filesystem to create.

::

    lustre_server_targets: []

This list of dictionaries defines the storage targets of the filesystem. Each dictionary in the list supports the following keys:

* **Required** ``label`` - The label for the storage target as it will appear in the ``/etc/ldev.conf``.

* **Required** ``path`` - Path to the storage device containing the target. For example, ``/dev/sdb``.

* **Required** ``local`` - Hostname of the primary MGS/MDS/OSS which should mount the target.

* ``foreign`` - Optionally define a failover MGS/MDS/OSS which should be listed in the ``/etc/ldev.conf``. 

* ``backfs`` - Optionally specify whether the target is formatted with ``ldiskfs`` or ``zfs``. Defaults to ``ldiskfs`` if unspecified.

* ``mkfs_opts`` - Optionally specify the parameters to be used for (re)formatting the storage target by calling ``mkfs.lustre {{ mkfs_opts }} {{ path }}``. The ``--reformat``, ``--fsname``, and ``--backfstype`` parameters are already specified. This is **very** dangerous, only executes if you specify the tag ``execute-lustre-mkfs`` in your playbook invocation, and will pause to confirm this was desired before executing.


::

    lustre_client_mounts: []

This list of dictionaries defines the mount(s) for Lustre on the clients. Each dictionary in the list supports the following keys:

* **Required** ``name`` - The path where the filesystem should be mounted.

* **Required** ``src`` - The source URL for the filesystem to be mounted.

* ``opts`` - A string containing filesystem options to be set when mounting the filesystem. Defaults to none.

* ``state`` - Optionally specify an alternate state to target for this client mount. Defaults to ``mounted``.


------------------
 Example Playbook
------------------

::

    ---
    - hosts: mds1,oss1
      vars:
        lustre_run_server: True
        lustre_fsname: "testfs"
        lustre_server_targets:
          - label: testfs-MDT0000
            path: /dev/sdb
            local: mds1
            mkfs_opts: --mgs --mdt --index 0
          - label: testfs-OST0000
            path: /dev/sdb
            local: oss1
            mkfs_opts: --ost --mgsnode mds1@tcp1 --index 0
      roles:
        - lustre

    - hosts: client
      vars:
        lustre_run_client: True
        lustre_client_mounts:
          - name: /lustre/testfs
            src: mds1@tcp1:/testfs
      roles:
        - lustre
