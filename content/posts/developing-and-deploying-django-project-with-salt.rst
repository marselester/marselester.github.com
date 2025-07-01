====================================================
Developing & Deploying Django project with SaltStack
====================================================

:date: 2013-11-28
:tags: python, django, vagrant, salt, saltstack
:category: Infrastructure
:slug: developing-and-deploying-django-project-with-saltstack

Eventually you will need to deploy project,
but deployment was not considered in the `previous post`_. Let's find it out.

Server configuration is different from local, thus environments will be needed
(at least production **prod** and development **dev**). Salt uses **base**
environment by default.

Environments are set in ``minion.conf`` because masterless minion is used.
When no environment is set, Salt assumes following configuration:

.. code-block:: yaml

  file_roots:
    base:
      - /srv/salt

  pillar_roots:
    base:
      - /srv/pillar

Salt will look for states in ``/srv/salt`` and pillars in ``/srv/pillar``.

Let's introduce **dev** and **prod** environments.

.. code-block:: yaml

    file_roots:
      base:
        - /srv/salt_roots/salt/base

      dev:
        - /srv/salt_roots/salt/dev
        - /srv/salt_roots/salt/base

      prod:
        - /srv/salt_roots/salt/prod
        - /srv/salt_roots/salt/base

    pillar_roots:
      base:
        - /srv/salt_roots/pillar/base

      dev:
        - /srv/salt_roots/pillar/dev
        - /srv/salt_roots/pillar/base

      prod:
        - /srv/salt_roots/pillar/prod
        - /srv/salt_roots/pillar/base

Each environment describes paths where to find files.
File is searched in the first directory and if it is not found it is
searched in the second one.

States and pillars which were described in `previous post`_ are splitted
to environment folders. Most of files are located in ``base`` folder,
because they are the same to **dev** and **prod** environments.
You can find `complete example`_ on GitHub.

What Production Environment Introduces
--------------------------------------

There is a difference between **prod** and **dev** project's source code
delivering. Development environment assumes that code is shared by Vagrant.
Production environment needs to get code from private git repository.
In order to do it user *jon_snow* must have ssh keys. Here I don't use
private repository, but keys are still needed to log in to production server
by Fabric.

.. code-block:: yaml

    jon_snow:
      user.present

    jon_snow public key:
      ssh_auth:
        - present
        - user: jon_snow
        - source: salt://user/id_rsa.pub
        - require:
          - user: jon_snow

    jon_snow secret key:
      file:
        - managed
        - name: /home/jon_snow/.ssh/id_rsa
        - source: salt://user/id_rsa
        - user: jon_snow
        - group: jon_snow
        - mode: 600
        - require:
          - ssh_auth: jon_snow public key

``prod/source_code.sls`` fetches source code from GitHub.
I have tried to use built in ``git.latest`` Salt state but it requires
to configure git name and email on server. I solved this issue by using
git hard reset.

.. code-block:: yaml

    git:
      pkg.installed

    github.com:
      ssh_known_hosts:
        - present
        - user: jon_snow
        - fingerprint: 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48

    messaging repository clone:
      cmd:
        - run
        - unless: test -d /home/jon_snow/messaging_repository
        - user: jon_snow
        - name: >
                  git clone https://github.com/marselester/abstract-internal-messaging.git /home/jon_snow/messaging_repository
        - require:
          - pkg: git
          - ssh_known_hosts: github.com

    messaging latest source code:
      cmd:
        - run
        - cwd: /home/jon_snow/messaging_repository
        - user: jon_snow
        - name: >
                  git fetch origin &&
                  git reset --hard origin/master
        - require:
          - cmd: messaging repository clone

      file:
        - symlink
        - name: {{ pillar['website_src_dir'] }}
        - target: /home/jon_snow/messaging_repository
        - force: True
        - user: jon_snow


Fabric as Deployment Runner
---------------------------

Last thing to do is to run deployment of particular Salt environment
**salt_env** in particular host **target**. Fabric is good at it.

I want Fabric to:

- bootstrap Salt on given **target**;
- upload ``minion.conf`` to given **target** to ``/etc/salt/minion``;
- upload Salt configs to given **target** how it is described
  in ``minion.conf`` (to ``/srv/salt_roots``);
- invoke ``salt-call`` on given **target** with particular **salt_env**
  (here is ``prod``).

  .. code-block:: console

      $ salt-call state.highstate env=prod

Let's try to deploy **prod** environment to VM. I assume that you have
``messaging`` directory (see `Developing Django project with SaltStack`_).
You need to clone `Deploy Abstract Internal Messaging System`_ in it
and start VM.

.. code-block:: console

    $ cd messaging
    $ git clone https://github.com/marselester/abstract-internal-messaging-deploy.git
    $ cd abstract-internal-messaging-deploy/vagrant
    $ vagrant up

In order to interact with VM you should add ``11.22.33.44 messaging-part-2``
to ``/etc/hosts``.

Install Fabric and set up Salt masterless minion in VM.

.. code-block:: console

    $ cd messaging/abstract-internal-messaging-deploy
    $ pip install -r requirements.txt
    $ fab target:local setup_masterless_minion

Finally deploy **prod** environment in VM.

.. code-block:: console

    $ fab target:local salt_env:prod deploy

Conclusion
----------

Besides convenient developing, we got ability to deploy arbitrary environments
to arbitrary targets. It gives us opportunity to test deployment itself.
You can improve this solution by adding ability to specify git branch
which will be deployed.

As in `previous post`_ project should work:

.. code-block:: console

    $ curl -i messaging-part-2

.. _Developing Django project with SaltStack:
.. _previous post: https://marselester.com/developing-django-project-with-saltstack.html
.. _Deploy Abstract Internal Messaging System:
.. _complete example: https://github.com/marselester/abstract-internal-messaging-deploy
