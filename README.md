## [![DebOps project](http://debops.org/images/debops-small.png)](http://debops.org) reprepro

[![Travis CI](http://img.shields.io/travis/debops/ansible-reprepro.svg?style=flat)](http://travis-ci.org/debops/ansible-reprepro) [![test-suite](http://img.shields.io/badge/test--suite-ansible--reprepro-blue.svg?style=flat)](https://github.com/debops/test-suite/tree/master/ansible-reprepro/)  [![Ansible Galaxy](http://img.shields.io/badge/galaxy-debops.reprepro-660198.svg?style=flat)](https://galaxy.ansible.com/list#/roles/1593)

`debops.reprepro` role is used to create and manage local APT repositories.
Source and binary Debian packages can be uploaded to this repository using
`dput` command, using WebDAV over HTTPS. Repositories are available over
HTTP as well as HTTPS, with optional IP/CIDR network restrictions. You need
to enable these repositories on other hosts separately, for example using
`debops.apt` role.

### Installation

This role requires at least Ansible `v1.7.0`. To install it, run:

    ansible-galaxy install debops.reprepro

### Documentation

More information about `debops.reprepro` can be found in the
[official debops.reprepro documentation](http://docs.debops.org/en/latest/ansible/roles/debops.reprepro.html).


### Role dependencies

- `debops.secret`

### Are you using this as a standalone role without DebOps?

You may need to include missing roles from the [DebOps common
playbook](https://github.com/debops/debops-playbooks/blob/master/playbooks/common.yml)
into your playbook.

[Try DebOps now](https://github.com/debops/debops) for a complete solution to run your Debian-based infrastructure.





### Authors and license

`reprepro` role was written by:
- Maciej Delmanowski | [e-mail](mailto:drybjed@gmail.com) | [Twitter](https://twitter.com/drybjed) | [GitHub](https://github.com/drybjed)

License: [GPLv3](https://tldrlegal.com/license/gnu-general-public-license-v3-%28gpl-3%29)

***

This role is part of the [DebOps](http://debops.org/) project. README generated by [ansigenome](https://github.com/nickjj/ansigenome/).
