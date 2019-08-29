Basics
======

Initial post-install actions to make a freshly installed FreeBSD
system nicely integrate into an Ansible ecosystem:

  - Bootstrap pkg(8) using the default FreeBSD repositories (using raw mode)
  - Install python (using raw mode)
  - Create local user accounts
  - Manage authorized_keys for user accounts
  - Configure sshd(8) for key-based login only
  - Optionally lock all use passwords, using keys exclusively

Once run, subsequent use of ansible against the host will be
authenticated by SSH keys, and use sudo for the become method.  

The sudo configuration defaults to allowing members of group wheel to
run any commands as root authenticated by their SSH key.  In order for
this to work, you will need to load your SSH key into an agent and
configure ansible to forward the agent to the remote host, typically
by adding settings to `ansible.cfg`:

```
[privilege_escalation]
become_flags = -H -S

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ForwardAgent=yes

```

Requirements
------------

In order to use passwords with SSH, the sshpass(1) program needs to be
installed on the ansible host.  This is available in ports --
`security/sshpass` or it can be compiled from source, available from

  https://sourceforge.net/projects/sshpass/files/latest/download

`sshpass` is only needed until SSH key-based authentication has been set
up, which is done by this role.

On the ansible clients, this module assumes that:

  * sshd(8) is enabled
  * there is a non-root user account present on the system that you can login
    to
  * that account can escalate privileges to root by some means
  * there is a pkg repository accessible from the system

Exact details depend on how the system was installed:

  * Installing from FreeBSD installation media

    During installation you will be prompted to set a root password,
    and also given the option to enable sshd(8) as well as to create
    one or more local user accounts.  In this case we need to use the
    standard su(1) application for initial root access and to give
    both login and become passwords.

  * Installing on the cloud

    AWS and Azure both default to creating a local user (typically
    'ec2-user' for AWS) which has full password-less sudo access.

Role Variables
--------------

```
basics_initial_become_method: su
```

How to gain root privileges before we can guarrantee that sudo(1) is
installed.  su(1) is appropriate for the results from the default
installation media.  If using on a cloud host, typically this can be
set to `sudo`.

```
basics_initial_become_flags: ''
```

Flags required for the `basics_initial_become_method` above.  When
we're using su(1) this should be empty but may need to be set
differently for alternate become methods.

```
basics_visudo:  /usr/local/sbin/visudo
```

Fully qualified path to the `visudo` command installed by the sudo package.


```
basics_sudoers_d_wheel:	/usr/local/etc/sudoers.d/wheel
```

Installation path for sudoers configuration snippet permitting members
of group wheel to run any commands via sudo.

```
basics_sudo_pam: /usr/local/etc/pam.d/sudo
```

Installation path for a pam configuration snippet that adds
pam_ssh_agent_auth as one of the methods to authenticate to sudo(8)

```
basics_sudo_authorized_keys: /etc/security/sudo_authorized_keys
```

Installation path for file containing a list of all the keys permitted
for SSH key based authentication to sudo(8).

```
basics_users: []
```

An array of user accounts to be created.  Each entry in the array is a
YAML dict with the following available fields:

```
basics_users:
  - user: matthew
    comment: Matthew Seaman
    shell: /bin/tcsh
    group: matthew
    other_groups:
      - wheel
      - operator
    password: '*'
    ssh_pub_keys:
      - 'ssh-ed25519 AAAAC3Nza...'
    auth_sudo: true
    update_password: true 

```

Only the `user` field is mandatory: the primary `group` will default
to the same name as the user, `shell` and `comment` will be empty
(which implies a shell of /bin/sh under FreeBSD).  `password` contains
the _password_ _hash_ for the user account, and defaults to '*' if
unset, meaning no password for the account -- which is fine if we're
using ssh-key based auth for both login and sudo.

This code will not change a pre-existing password unless
`update_password` is set to true, but it may move the user's home
directory.  The default is to only modify an account password when
creating the account.

The `auth_sudo` field is a boolean: a true value indicates adding the
user's SSH public keys to the keys allowed for sudo
authentication. (Default: true)

Dependencies
------------

None

Example Playbook
----------------

We don't try and gather facts for hosts since that assumes the
availability of python _before_ this role has had an opportunity to
install it.

```
 - hosts: all
   gather_facts: no
   become: yes
   roles:
     - basics
```

This will need to be run with a command line like so:
```
  % ansible-playbook -K -k -u username playbooks/basics.yaml 
```

Of course, `-u username` can be omitted if the remote user name we're
logging-in as happens to be the same as the local user.

License
-------

BSD

Author Information
------------------

Matthew Seaman
