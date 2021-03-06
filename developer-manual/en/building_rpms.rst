.. _buildrpm-section:

=============
Building RPMs
=============

The entire RPM building process is automatized and is run by
:command:`build-rpm` command, provided by `nethserver-devbox` package. 

The process has been successfully tested on

* NethServer 6.x
* Fedora 17 and later, with ``mock <= 1.35``, until `Bug 2879`_ will be fixed.

.. _bug 2879: http://dev.nethserver.org/issues/2879

Let assume you have a ``Projects/`` directory where you have cloned
some git repositories. The :command:`build-rpm` command requires that each project directory:

* is a git repository
* contains a file named after it, plus ``.spec`` or ``.spec.in`` extension
* has a tag representing the current version number: ``X.Y.Z``

For example, a simple directory layout would be:

* ``Projects/``

  * ``projA/``

    * ``.git/``
    * ``projA.spec.in``

  * ``projB/``

    * ``.git/``
    * ``projB.spec``



Spec template file
==================

.. WARNING:: 
   The `.spec.in` template format is deprecated and will be removed on
   version `2.0.0`. Use only `.spec` format for new projects.

A :file:`.spec.in` template file syntax is the same as RPM :file:`.spec` files,
where

* :dfn:`Version` and :dfn:`Release` tag values are placeholders that are substituted during the build process: ::
  
    Version: @VERSION@
    Release: @RELEASE@

* :dfn:`%changelog` section is at the bottom of the file.

.. _rpm_prepare_env:

Prepare your development machine
================================

On **NethServer**, you can simply install nethserver-devbox, by typing: ::

  yum install nethserver-devbox

Then copy :file:`/usr/share/doc/nethserver-devbox-<version>/config.sample` to :file:`Projects/config` and edit it.

On **Fedora**, clone nethserver-devbox repository somewhere in your filesystem: ::

  git clone https://github.com/nethesis/nethserver-devbox.git

Then install some packages marked as "Requires" in :file:`nethserver-devbox.spec`: 

* mock
* expect
* tar
* fuse-iso fuse 
* genisoimage
* git
* rpm-build
* intltool
* isomd5sum
* syslinux
* rpmdevtools


Add :command:`build-rpm` and :command:`build-iso` commands to your :file:`PATH`. For instance create symlinks in your :file:`~/bin` directory: ::

  ln -s <nethserver-devbox-dir>/build-rpm  ~/bin/
  ln -s <nethserver-devbox-dir>/build-iso  ~/bin/

Copy :file:`config.sample` to :file:`Projects/config` and edit it.

Fetch external sources
======================

The git repository may not be the only code source.  External source tarballs
can be addressed by `Source` tags in the spec file, according to
Fedora `Packaging:SourceURL`_ guidelines.

.. _`Packaging:SourceURL`: http://fedoraproject.org/wiki/Packaging:SourceURL

The external sources can be fetched by issuing the following command: ::

  spectool -g <specfile>

Each external source tarball must be verifiable by its `SHA` string in file
:file:`SHA1SUM` placed at the repository root.

Build the RPM
=============

The build process uses Mock_ and must be run as a non privileged user
in the `mock` system group.  Add your user with: ::

  usermod -a -G mock <username>

The build-rpm script

* creates the tarball and the :file:`.spec` file for the given package name (if starting from a `.spec.in` template)
* verifies external source tarballs SHA hashes against :file:`SHA1SUM`
* builds the source and binary RPMs
* signs RPMs with your GPG key (``-s`` or ``-S <KEYID>`` options)
* copy RPMs to a local yum repository  (if ``REPODIR`` directory exists)
* publish RPMs to a remote yum repository (``-p`` option. Configure ``PUBLISH_*`` parameters and ssh access)

The script can execute one or more tasks listed above in the same run. Intermediate files are written to ``WORKDIR``. ::
  
  build-rpm
  Usage: build-rpm [-cousp] [-S <gpgkeyid>] [[-D <key>=<value>] ... ] <package_name> ...
   
.. _Mock: http://fedoraproject.org/wiki/Projects/Mock


Development release
===================

If you want to create a package with a development release, just execute from the :file:`Projects/` directory: ::

  build-rpm <package>

The system will search for the first available tag inside the git
repository and will calculate the version and release values (see
:command:`git describe`). This means **the tag must exist**!

* For ``.spec.in`` file, RPM ``Version`` is last git tag, RPM ``Release`` has the form ``<commits_from_tag>.0git<commit_hash>.<DIST>``.
* For ``.spec`` file, the ``%{dist}` macro value has the form ``.<commits_from_tag>git<commit_hash>.<DIST>``.


Stable release
==============

When you are ready for a production release, the :command:`release-rpm` command helps you in the following tasks:

* Fetch changelog info from git and relate commits with issues from Redmine installation at ``REDMINE_URL``.
* Update the changelog section in ``spec`` or ``spec.in`` file (whatever applies).
* Review and commit the changelog.
* Create a (signed) git tag.

The commit and tag are added locally, thus they need to be pushed to your
upstream git repository, once reviewed.

::

  release-rpm
  Usage: release-rpm [-s] [-T X.Y.Z] <git repo>

For instance:

::

  release-rpm -s -T 1.2.3 nethserver-base

Your ``$EDITOR`` program (or git core.editor) is opened automatically to adjust the commit message. The same text is used as tag annotation. 

To abort at this point, save an empty message.

When ``build-rpm`` is executed on a tagged version
``<commits_from_tag>git<commit_hash>`` form is stripped.


Old releases
============

If you want to create a RPM with a specific version, use ``git
checkout`` to set the source tree to that version then proceed as
usual: ::

  cd <package>
  git checkout <versionrefernce>
  cd ..
  build-rpm <package>  


Sign the RPM
============

Just execute:

::

  build-rpm -s 

or

::

  build-rpm -S  

If a password is not set in :file:`config` file, you can set
``SIGN_KEYRING_NAME`` and ``SIGN_KEYRING_ID`` to fetch the secrets
from gnome-keyring. The :command:`print-gnome-keyring-secret` command
reads the secrets from gnome-keyring.


Publish the RPM
===============

.. note::  
  The nethserver-devbox package must be installed on the remote
  machine (``PUBLISH_HOST``). In the repository root directory
  (``PUBLISH_DIR``), create a ``Makefile`` symbolic link to
  :file:`repository.mk` .

Copy the package to the remote server using SSH:

::

  build-rpm -p 

After the RPMs have been built, they are copied to ``PUBLISH_HOST`` into
``PUBLISH_DIR``. Then :command:`make` is run on the remote machine directory to
update the yum repository.

Known problems
==============

When using mock on a VirtualBox (or KVM) virtual machine, the system can
lock with error similar to this one:

::

    ... BUG: soft lockup - CPU#0 stuck for 61s! [yum:xxx] ... (repeating)..

The bug is reproducible with kernel 2.6.32-431.x.
To avoid the problem, downgrade the kernel:

::

    wget http://vault.centos.org/6.4/os/x86_64/Packages/kernel-2.6.32-358.el6.x86_64.rpm
    yum localinstall kernel-2.6.32-358.el6.x86_64.rpm

Then reboot and choose the 2.6.32-358 kernel.
