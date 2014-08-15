etckeeper
=========

Ansible role to install, configure, and use etckeeper.

[![Build Status](https://travis-ci.org/expansible/etckeeper.svg?branch=master)](https://travis-ci.org/expansible/etckeeper)

Requirements
------------

* Developed and tested with Ansible 1.5, but should work with 1.3 or later.
* Debian/Ubuntu system (python-apt needed) [Patches to support yum welcomed!]

Role Variables
--------------
There are three parameter variables (with defaults in the role tasks file):

* **install** - *boolean* (default is **false**)

  This should be set as **true** in the first play.

* **commit** - *boolean* (default is **true**)

  Enables notification to "Record changes in etckeeper" handler,
  which generates an etckeeper commit for the play.
  Rather than setting this to false, just omit the etckeeper role entirely
  (unless this is the first play).

* **precommit** - *boolean* (default is **true**)

  Enables the "Record unsaved changes in etckeeper" task,
  which generates an etckeeper commit before the play runs.
  You might want to set this to false if you want to combine the results
  of several plays in a single commit, but this is not generally advised.
  Note that precommit is always disabled if the *commit* variable is false. 

There are two variables (with defaults in ``defaults/main.yml``)
that control the commit messages used by etckeeper when *commit* is true:

* **etckeeper_message** - *string*
  (default is "changes from Ansible play running as {{ ansible_user_id }}")

  This message is used for commits that take place at the end of a play.

* **etckeeper_pre_message** - *string*
  (default is "saving uncommitted changes in /etc prior to ansible play")

  This message is used for commits that take place at the start of a play.
  These commits save any pending changes separately.

There is a configuration variable (with default in ``defaults/main.yml``)
that controls the version control system that etckeeper will use
if *install* is true and etckeeper has not previously been installed:

* **etckeeper_vcs** - *string  (default is "git", or "hg", "bzr", or "darcs")

  This determines the version control system that etckeeper will use.
  Although the etckeeper package default is Mercurial ("hg"),
  this Ansible role has only been tested with Git.
  If etckeeper has already been installed, this variable has no effect.

Dependencies
------------

None.


Example Playbook
-------------------------

This role (and nothing else) should be the first play in a playbook,
and ideally you would run it as the very first play on any server,
since the first time it runs it generates an initial commit
with (nearly) the entire contents of the ``/etc/`` directory.

Adding the etckeeper role to other plays generates etckeeper commits
at the end of the play (and possibly the start, if there are unsaved changes).
For accurate unsaved change detection, the etckeeper role should be the first.
Using the etckeeper role, the etckeeper commits run in a handler,
(once) after all tasks, but possibly before some other handlers.

Eventually, this software should add an ``etckeeper`` action module
so tasks can trigger immediate etckeeper commits in conditional tasks;
for now, you can use shell actions as illustrated in the example.

Running etckeeper commits in tasks allows more than one in a play,
but since all roles in a play run before any playbook tasks in the play,
breaking the playbook into many smaller plays with one role each
is still necessary to achieve finer-grained commits.

Here is an example playbook that uses the etckeeper role to perform
installations and commits, and also uses a shell action to perform commits.

    ---
    # This should be the first play in the playbook
    - hosts: all
      vars:
      - etckeeper_vcs: git
      roles:
      - { role: etckeeper, install: true }
      # Do not add any other roles to this play

    # This is the second play
    - hosts: all
      roles:
      - { role: etckeeper, etckeeper_message: '2nd play of playbook' }
      # additional roles here

    # This is the third play
    - hosts: all
      tasks:
      - name: Install and/or configure something
        command: cp /dev/null /etc/null
        register: result
      - name: Record changes for previous task in etckeeper commit
        shell: if etckeeper unclean; then etckeeper commit '3rd play pt. 1'; fi
        when: result|changed
      - name: Do something that doesn't change anything in /etc
        action: true
      - name: Do something else that could change something in /etc
        action: cp /dev/null /etc/zero
        notify: Record other changes for this play in etckeeper commit
     handlers:
     - name: Record other changes for this play in etckeeper commit
       shell: if etckeeper unclean; then etckeeper commit '3rd play pt. 2'; fi

License
-------

MIT (Expat) - see LICENSE file for details

Author Information
------------------

You can contact me at [alex.dupuy mac.com](mailto:alex.dupuy%40mac.com);
check out my other open source contributions at
[Ohloh](https://www.ohloh.net/accounts/dupuy/).
