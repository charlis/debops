.. Copyright (C) 2015-2020 Maciej Delmanowski <drybjed@gmail.com>
.. Copyright (C) 2015-2020 DebOps <https://debops.org/>
.. SPDX-License-Identifier: GPL-3.0-only

Usage guides
============

.. only:: html

   .. contents::
      :local:

Design of global variables in DebOps
------------------------------------

When Ansible playbooks are designed to be read-only, to be able to get the
updated versions, Ansible does not have good support for variables that are
supposed to be accessed by different roles at any point. The issues arise when
the user starts to use ``--tags`` or ``--skip-tags`` parameters to selectively run
parts of playbooks or roles – Ansible does not evaluate variables from roles that
weren't included, which can change the environment and break the idempotency.

The solution to use the :file:`group_vars/all/` directory in the playbook directory
doesn't work, because variables defined there cannot be overwritten by Ansible
inventory, thereby they cannot be changed as needed by the user. A separate
Ansible role could be used with variables defined in it's
:file:`defaults/main.yml`, but it would need to be a dependency of all the roles
that used these variables, so virtually all roles would need to use it for
consistency.

A different solution, which is implemented by the ``debops.core`` role, is to use
Ansible local facts defined on the remote hosts themselves as a data store for
variables that are meant to be visible to all roles at all times. Ansible
gathers these facts on any playbook execution and they are accessible from
anywhere in the playbook or roles.

To make the configuration easier to modify by the user, values for these local
facts are derived from ``debops.core`` default variables, which means that the user
can redefine them in the Ansible inventory. For consistency and idempotency
this role will take care to use existing values even if their definitions are
removed from the inventory.

By moving the variables to the remote host itself, ``debops.core`` does not need to
be included in all other roles as a dependency, and it can be simply executed
once at the start of the playbook. To resolve issues with missing dict keys
during Ansible runs, ``debops.core`` is "artificially required" by all other
DebOps roles. If the main DebOps playbook is used, this doesn't change
anything, but if roles are used separately, or from a custom playbook,
the ``debops.core`` role should be included at the start, preferably in a separate
play to make sure that Ansible re-gathers the local facts after the role has
configured them.

To make the local facts consistent and managed centrally, ``debops.core``
provides a custom set of fact scripts which are used to dynamically gather
certain facts about a given host. Any new custom fact scripts which is
independent of a specific role, will be included in this one.

Custom local facts
------------------

The ``debops.core`` role allows the user to specify custom variables which will be
configured in the Ansible local facts on a given host. Three levels of
variables that can be used:

:envvar:`core__facts`
  Dictionary which should be defined in the :file:`inventory/group_vars/debops_all_hosts/`
  group which applies to all hosts in the inventory.

:envvar:`core__group_facts`
  Dictionary which should be defined in the ``inventory/group_vars/*/``
  group to set variables on specific sets of hosts. Only one group level is
  supported.

:envvar:`core__host_facts`
  Dictionary which should be defined in ``inventory/host_vars/*/``
  for a particular host.

The key specifies the name of a variable in the ``ansible_local.core.*`` namespace, with
value being it's value. You can use normal YAML variables as values, even lists
and dictionaries.

All variables defined in the inventory will be merged in one namespace, more
specific variables overriding the less specific ones (global -> group -> host).

The role takes care to reuse already set local facts even if their definition
has been removed from the inventory, however changes in the inventory will override
local facts. It's best not to change already defined variables like file and
directory paths, because that might break already configured software if the
involved directories/files are not taken care of.

Additional variables can be used to manipulate facts defined on remote hosts:

:envvar:`core__remove_facts`
  List of fact names in ``ansible_local.core.*`` which will be
  removed if found.

:envvar:`core__reset_facts`
  Boolean. If set to ``True``, ``debops.core`` role will ignore facts already
  defined on remote hosts and recreate the ``ansible_local.core.*`` namespace
  using only facts defined in Ansible inventory.

Examples
~~~~~~~~

Create a set of custom facts:

.. code-block:: yaml

   core__facts:
     'fact_name': 'fact_value'
     'extra_list': [ 'list', 'of', 'values' ]
     'nested_dict':
       'some_key': 'some_value'

When above variables are defined they can be accessed using Jinja variables:

.. code-block:: yaml

   fact_name: '{{ ansible_local.core.fact_name }}'
   extra_list: '{{ ansible_local.core.extra_list | join(" ") }}'
   nested_dict: '{{ ansible_local.core.nested_dict.some_key }}'

Above code will work correctly if ``debops.core`` has been executed previously
on a host. If you want your role to be compatible with installations that don't
use it, you need to write your variable like this:

.. code-block:: yaml

   var: '{{ ansible_local.core.fact_name|d("fact_value") }}'

That way Ansible won't emit an error about missing dictionary keys at each
level of the ``ansible_local`` variable namespace.

Custom host tags
----------------

"Host tags" work similar to custom local facts. The difference is that this is
only a single list of items, merged from separate variables on all levels of
the inventory. You can set host tags using the variables:

:envvar:`core__tags`
  Global list of tags, should be defined in :file:`inventory/group_vars/debops_all_hosts/`

:envvar:`core__group_tags`
  List of tags for a specific group, should be defined in
  ``inventory/group_vars/*/``

:envvar:`core__host_tags`
  List of tags for a specific host, should be defined in
  ``inventory/host_vars/*/``

:envvar:`core__static_tags`
  Any list specified here will override already defined tags.

Tags can be accessed using the ``ansible_local.tags`` list variable. Other roles
can check if a given item is or is not present in this global list and perform
actions depending on that state.

Examples
~~~~~~~~

Check if a given value is in the tag list:

.. code-block:: yaml

   - name: Show debug output
     debug: msg="Test"
     when: ansible_local|d() and ansible_local.tags|d() and
           'value' in ansible_local.tags

Check if a given value is not in the tag list:

.. code-block:: yaml

   - name: Show debug output
     debug: msg="Test"
     when: ansible_local|d() and ansible_local.tags|d() and
           'value' not in ansible_local.tags

You can find a list of host tags in the documentation of various roles which use
them.

System administrator accounts
-----------------------------

Common feature in various services is creation of an administrator account. The
``debops.core`` role provides two Ansible local facts which can be used by
other roles to make creation of these accounts easier.

``ansible_local.core.admin_groups``
  List of the UNIX system groups which contains system administrator accounts.

``ansible_local.core.admin_users``
  List of the UNIX user accounts which are members of the above UNIX groups.
  These accounts should be used by the other Ansible roles to create
  administrator accounts if none were set by the user through the Ansible
  inventory.

You can use the corresponding role default variables to control what admin
accounts are available to other roles.

Examples
~~~~~~~~

Define list of admin accounts to create in the application:

.. code-block:: yaml

   application__admins: '{{ ansible_local.core.admin_users|d([]) }}'

Custom distribution and release facts
-------------------------------------

Ansible sometimes detects the installed OS distribution and release
incorrectly. For example, current Debian Testing release is not detected at
all, and the ``ansible_distribution_release`` variable is set to ``NA`` which,
if used in the roles, can break a lot of existing configuration.

The ``debops.core`` role provides alternative set of the
``ansible_distribution`` and ``ansible_distribution_release`` variables through
Ansible local facts, accessible as ``ansible_local.core.distribution`` and
``ansible_local.core.distribution_release``. They use the original Ansible
facts if they are not ``NA`` and refer to the ``ansible_lsb`` otherwise; they
can also be overridden through Ansible inventory. By using these local facts in
your roles, you can have a centralized place to control these facts if
necessary.

Examples
~~~~~~~~

In your role default variables, create separate variables that hold the
information about current distribution and release:

.. code-block:: yaml

   application__distribution: '{{ ansible_local.core.distribution|d(ansible_distribution) }}'

   application__distribution_release: '{{ ansible_local.core.distribution_release|d(ansible_distribution_release) }}'

.. _core__ref_unsafe_writes:

Global unsafe writes
--------------------

Many Ansible modules related to file operations support the ``unsafe_writes``
parameter to allow operations that might be dangerous or destructive in certain
conditions, but allow Ansible to work in specific environments, like
bind-mounted files or directories. The :envvar:`core__unsafe_writes` default
variable allows to activate this mode per-host using Ansible inventory, for all
roles that implement it.

To have an effect, roles that depend on the unsafe writes to function, should
use the parameter in relevant tasks, like this:

.. code-block:: yaml

   - name: Generate configuration file
     template:
       src: 'etc/application.conf.j2'
       dest: '/etc/application.conf'
       owner: 'root'
       group: 'root'
       mode: '0644'
       unsafe_writes: '{{ True if (core__unsafe_writes|d(ansible_local.core.unsafe_writes|d()) | bool) else omit }}'

Note that the way :envvar:`core__unsafe_writes` is checked and takes precedence
even from the context of another role is not otherwise done in DebOps.
This was done in this case to allow to only enable
:envvar:`core__unsafe_writes` when necessary without the need to run the
``debops.core`` role first and ensuring that it’s facts are made persistent as well.
