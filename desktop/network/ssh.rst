SSH - Secure Shell
==================

.. image:: openssh-logo.*
    :alt: OpenSSH Logo
    :align: right
    :width: 300px

.. contents::
  :local:

`OpenSSH <https://www.openssh.com/>`_ is the premier connectivity tool for
remote login with the :term:`SSH` protocol. It encrypts all traffic to eliminate
eavesdropping, connection hijacking, and other attacks. In addition, OpenSSH
provides a large suite of secure tunneling capabilities, several authentication
methods, and sophisticated configuration options.

The OpenSSH suite consists of the following tools:

* Remote operations are done using ssh, scp, and sftp.
* Key management with ssh-add, ssh-keysign, ssh-keyscan, and ssh-keygen.
* The service side consists of sshd, sftp-server, and ssh-agent.

OpenSSH is developed by a few developers of the OpenBSD Project and made
available under a BSD-style license.

.. note::

    The following is valid for *OpenSSH_8.2p1 Ubuntu-4, OpenSSL 1.1.1f  31 March 2020*
    as shipped with *Ubuntu 20.04 LTS "Focal Fossa"*.
    See the `OpenSSH release notes <https://www.openssh.com/releasenotes.html>`_
    for changes since the 7.6 release that came with Ubuntu 18.04.

.. contents::
    :depth: 1
    :local:
    :backlinks: top


SSH Server
----------

On Ubuntu Desktop the SSH server is not installed by default::

    $ sudo apt install ssh molly-guard

For configuration, see our :doc:`SSH server configuration </server/ssh-server>`.


SSH Client
----------

System-Wide Client Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

    The following configuration options can also be just kept in your personal
    user settings in the :file:`~/.ssh/config` file. But if you services or
    scripts running under other users or if this system is used by multiple user
    profiles, it might be easier to maintain a system-wide configuration.


The system-wide default client settings are stored in
:file:`/etc/ssh/ssh_config`. The options are described in the
`ssh_config(5) <https://manpages.ubuntu.com/manpages/focal/en/man5/ssh_config.5.html>`_
man page.

.. literalinclude:: ../config-files/etc/ssh/ssh_config

In case you where wondering about the `HashKnownHosts` options, I suggest
reading `Joey’s Blog about this <https://blog.joeyhewitt.com/2013/12/openssh-hashknownhosts-a-bad-idea/>`_

Specific settings for certain domains and networks, like your own, friends or
customers might be better placed in their onwn files for easier maintenance and
distribution. Tha's what the "Include" statement and the
:file:`/etc/ssh/ssh_config.d/` directory are for.

Create a file like :file:`/etc/ssh/ssh_config.d/example.net.conf` and make
changes according your needs:

.. literalinclude:: ../config-files/etc/ssh/ssh_config.d/local.conf

.. literalinclude:: ../config-files/etc/ssh/ssh_config.d/example.net.conf


User Configuration
^^^^^^^^^^^^^^^^^^

The client settings for users are stored in
:file:`/etc/ssh/ssh_config`. The options are the same as described in the
`ssh_config(5) man page <https://manpages.ubuntu.com/manpages/focal/en/man5/ssh_config.5.html>`_

In the file :file:`~/.ssh/config` you can customize your client (like specific
user names) or add 3rd-party systems which are not covered by system-wide
settings:

.. literalinclude:: ../config-files/.ssh/config

Again we create several include files for different networks.

In the file :file:`~/.ssh/config.d/local.conf` we set options for discoverable
hosts in our LAN:

.. literalinclude:: ../config-files/.ssh/config.d/local.conf

The file :file:`~/.ssh/config.d/example.net.conf` contains settings for our own
servers:

.. literalinclude:: ../config-files/.ssh/config.d/example.net.conf

.. code-block:: ini



    #
    # 3rd-party systems
    #
    Host kissy.example.org
        Port 54393
        User johnd

    Host holden.example.org
        Port 51193

    Host github.com
        User git

    Host *.synology.me
        VerifyHostKeyDNS No




OpenSSH Trust in DNSSEC
-------------------------

In the previous section, we have set our SSH client to verify the servers SSH
public key with the fingerprints published in DNS trough the **VerifyHostKeyDNS**
configuration option. Unfortunately this wont work out of the box, as the
following tests will show:

Set this to your own servers hostname for the following checks to work::

    $ export TEST_HOST=dolores.example.net

Let's check if the fingerprints of our server are present in DNS and that these
DNS records are secured by DNSSEC::

    $ dig $TEST_HOST SSHFP | egrep "ad|$"
    flags: qr rd ra ad

The **ad** flag in the DNS answer stands for "**a**\ uthenticated **d**\ ata"
and confirms that the DNS records requested have been successfully verified with
valid DNSSEC signatures. But the OpenSSH client will still insist, that the
fingerprints, while visible, are not to be trusted::

    $ ssh -v $TEST_HOST logout 2>&1 | egrep "found .* in DNS|$"
    debug1: found 2 insecure fingerprints in DNS

This is caused by the GNU C library and its not just a simple bug, but a rather
complex trust issue described in
`Glibc support encryption by modifying the DNS <http://www.newfreesoft.com/programming/glibc_support_encryption_by_modifying_the_dns_5365/>`_.

Unless :file:`/etc/resolv.conf` contains **edns0** and **trust-ad** as
configuration options, programs who use the GNU C library (glibc) like OpenSSH
and many others, aren't able to see that the :term:`DNSSEC` validation was
successful.

Nowadays the :file:`/etc/resolv.conf` file is managed by systemd, NetworkManager,
the resolvconf service or whatever you use as your
:doc:`local DNS resolver <unbound>`. Its therefore no longer possible and not
recommended to change anything in this file manually.

As as described in the
`manpage for resolv.conf(5) <https://manpages.ubuntu.com/manpages/focal/en/man5/resolv.conf.5.html>`_
as sn alternative, the options can also be set as space separated list in the
**RES_OPTIONS** environment variable, :

Let's try this out::

    $ RES_OPTIONS="edns0 trust-ad"
    $ ssh -v $TEST_HOST logout 2>&1 | egrep "found .* in DNS|$"
    debug1: found 2 secure fingerprints in DNS


.. warning::

    The following system-wide configuration settings should only be made, if
    trust your DNS resolvers and providers.
    `dnssec-trigger <unbound.html#unbound-and-dnssec-trigger>`_ can help to
    establish this trust.


Bash Environment
^^^^^^^^^^^^^^^^

To set this as a system-wide default for terminal sessions and shell scripts,
add the following file to :file:`/etc/profile.d` directory:

.. code-block:: sh

    #!/usr/bin/env bash
    #
    # Let programs who use the GNU C library (glibc) see if DNS answers are
    # authenticated by DNSSEC. Required for OpenSSH to trust in DNSSEC-signed
    # SSHFP records.
    # See man resolv.conf(5)

    # Set the value, preserve existing if already set.
    export RES_OPTIONS="$RES_OPTIONS edns0 trust-ad"


Systemd Environment
^^^^^^^^^^^^^^^^^^^

Gnome desktop applications, like remote SFTP folders in Nautilus, may not read
your bash environment, as they are not running in your terminal session. Since
these are managed by Systemd, we set these trough a `systemd environment file
generator
<https://manpages.ubuntu.com/manpages/focal/en/man7/systemd.environment-generator.7.html>`_.

Create the systemd user environment directory::

    $ sudo mkdir -p /etc/systemd/user-environment-generators

Create the file
:file:`/etc/systemd/user-environment-generators/90res-options` or
:download:`download it <../config-files/etc/systemd/user-environment-generators/90res-options>`

.. literalinclude:: ../config-files/etc/systemd/user-environment-generators/90res-options

It needs to be executable::

    $ sudo chmod +x /etc/systemd/user-environment-generators/90res-options


See also
--------

You may also look at these related pages:

 * :doc:`/desktop/secrets/yubikey/yubikey_ssh`
 * :doc:`/desktop/secrets/gpg/index`
 * :doc:`/desktop/secrets/keys`
 * :doc:`/server/ssh-server`
