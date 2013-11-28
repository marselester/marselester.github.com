========================================
Developing Django project with SaltStack
========================================

:date: 2013-11-09
:tags: python, django, vagrant, salt, saltstack
:category: Infrastructure
:slug: developing-django-project-with-saltstack

Let's use `Messaging System`_ as an example of Django project. I want it to
run in VirtualBox which is managed by Vagrant. Infrastructure management
is provided by SaltStack.

I advise you to create separate folder for repositories (currently there
is only one) of project and clone `Messaging System`_ there.
Also I need a folder for Vagrant files and prefer to name it ``vagrant``.

.. code-block:: console

    $ mkdir messaging
    $ cd messaging
    $ git clone https://github.com/marselester/abstract-internal-messaging.git
    $ mkdir abstract-internal-messaging/vagrant

In order to make friends of Vagrant and SaltStack, I need to install
Salty Vagrant:

.. code-block:: console

    $ vagrant plugin install vagrant-salt

I want Vagrant to:

- share source code folder ``messaging/abstract-internal-messaging``;
- share states and pillars of SaltStack
  ``messaging/abstract-internal-messaging/vagrant/salt/roots``;
- run minion masterless by ``salt-call state.highstate``;
- assign convenient host name, here it is "messaging-part-1".

``vagrant/Vagrantfile`` config responses for these requirements:

.. code-block:: ruby

    # -*- mode: ruby -*-

    Vagrant.configure('2') do |config|
      config.vm.box = 'precise64'
      config.vm.box_url = 'http://files.vagrantup.com/precise64.box'

      config.vm.provider :virtualbox do |v|
        v.customize ['modifyvm', :id, '--name', 'messaging-part-1']
        v.customize ['modifyvm', :id, '--memory', 1024]
      end

      config.vm.hostname = 'messaging-part-1'

      config.vm.network :private_network, ip: '1.2.3.4'

      config.vm.synced_folder '..', '/home/vagrant/abstract-internal-messaging'
      config.vm.synced_folder 'salt/roots', '/srv'

      config.vm.provision :salt do |salt|
        salt.minion_config = 'salt/minion.conf'
        salt.run_highstate = true
        salt.verbose = true
      end
    end

As you could notice, I have not had the ``vagrant/salt`` folder yet.
Let's create following directory structure (actually you can just
``git checkout salted-vagrant``):

.. code-block:: console

    $ tree abstract-internal-messaging/vagrant
    abstract-internal-messaging/vagrant
    |-- Vagrantfile
    `-- salt
        |-- minion.conf
        `-- roots
            |-- pillar
            |   |-- top.sls
            |   |-- postgresql.sls
            |   `-- website.sls
            `-- salt
                |-- top.sls
                |-- user.sls
                |-- source_code.sls
                |-- python.sls
                |-- redis.sls
                |-- postgresql
                |   |-- init.sls
                |   `-- pg_hba.conf
                `-- website
                    |-- django.sls
                    |-- wsgiserver.sls
                    |-- webserver.sls
                    |-- local.py.template
                    |-- supervisord.conf
                    |-- gunicorn.conf.py
                    `-- nginx.conf

``minion.conf`` contains only one string:

.. code-block:: yaml

    file_client: local


It means "Don't look for master server when running ``salt-call``".
It will assume that the local system has all of the file and pillar resources.

Rest of files I will consider separately.

User and Source Code states
---------------------------

``salt/user.sls`` creates user "jon_snow".

.. code-block:: yaml

    jon_snow:
      user.present

``salt/source_code.sls`` makes shared source code available in jon_snow's home
by creating symlink.

.. code-block:: yaml

    messaging source code:
      file:
        - symlink
        - name: /home/jon_snow/abstract-internal-messaging
        - target: /home/vagrant/abstract-internal-messaging
        - force: True

Hardcoded path can be replaced by ``{{ pillar['website_src_dir'] }}``,
which will be introduced further.

Python state
------------

``salt/python.sls`` installs Python 2, pip and Virtualenv.

.. code-block:: yaml

    python2:
      pkg:
        - installed
        - names:
          - python-dev
          - python

    pip:
      pkg:
        - installed
        - name: python-pip
        - require:
          - pkg: python2

    virtualenv:
      pip:
        - installed
        - require:
          - pkg: pip

Redis and PostgreSQL states
---------------------------

``salt/redis.sls`` installs Redis.

.. code-block:: yaml

    redis-server:
      pkg.installed

``salt/postgresql/init.sls`` installs PostgreSQL 9.1, copies ``pg_hba.conf``,
starts ``postgresql`` service, creates user and database based on pillar's
data.

.. code-block:: yaml

    postgresql:
      pkg:
        - installed
        - names:
          - postgresql-9.1
          - python-dev
          - libpq-dev

      service.running:
        - watch:
          - file: /etc/postgresql/9.1/main/pg_hba.conf
        - require:
          - pkg: postgresql

      file.managed:
        - name: /etc/postgresql/9.1/main/pg_hba.conf
        - source: salt://postgresql/pg_hba.conf
        - user: postgres
        - group: postgres
        - mode: 644
        - require:
          - pkg: postgresql

    postgresql-database-setup:
      postgres_user:
        - present
        - name: {{ pillar['postgresql_user'] }}
        - password: {{ pillar['postgresql_password'] }}
        - createdb: True
        - user: postgres
        - require:
          - service: postgresql

      postgres_database:
        - present
        - name: {{ pillar['postgresql_db'] }}
        - encoding: UTF8
        - lc_ctype: en_US.UTF8
        - lc_collate: en_US.UTF8
        - template: template0
        - owner: {{ pillar['postgresql_user'] }}
        - user: postgres
        - require:
          - postgres_user: postgresql-database-setup

``pillar/postgresql.sls``

.. code-block:: yaml

    postgresql_user: jon_snow
    postgresql_password: ghost
    postgresql_db: jon_snow

``salt/postgresql/pg_hba.conf``

.. code-block:: yaml

    # This file controls: which hosts are allowed to connect, how clients
    # are authenticated, which PostgreSQL user names they can use, which
    # databases they can access. Records take one of these forms:
    #
    # local DATABASE        USER            METHOD  [OPTIONS]
    local   jon_snow        jon_snow        md5

    # Database administrative login by Unix domain socket
    local   all             postgres                                peer

    # TYPE  DATABASE        USER            ADDRESS                 METHOD

    # "local" is for Unix domain socket connections only
    local   all             all                                     trust
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5

Website's Django state
----------------------

``salt/website/django.sls`` creates virtual environment and installs
project's dependencies there. It copies django settings, collects static
files and migrate db as well.

.. code-block:: yaml

    {{ pillar['website_venv_dir'] }}:
      file:
        - directory
        - user: jon_snow
        - group: jon_snow
        - makedirs: True

      virtualenv:
        - managed
        - no_site_packages: True
        - distribute: True
        - requirements: {{ pillar['website_requirements_path'] }}
        - user: jon_snow
        - no_chown: True
        - require:
          - pip: virtualenv
          - file: {{ pillar['website_venv_dir'] }}

    django settings:
      file:
        - managed
        - name: {{ pillar['website_settings_path'] }}
        - source: salt://website/local.py.template
        - template: jinja

    django-admin collectstatic:
      module:
        - run
        - name: django.collectstatic
        - bin_env: {{ pillar['website_venv_dir'] }}
        - settings_module: messaging.settings.local
        - pythonpath: {{ pillar['website_src_dir'] }}
        - noinput: True
        - require:
          - virtualenv: {{ pillar['website_venv_dir'] }}
          - file: django settings

    django-admin migrate:
      module:
        - run
        - name: django.syncdb
        - bin_env: {{ pillar['website_venv_dir'] }}
        - settings_module: messaging.settings.local
        - pythonpath: {{ pillar['website_src_dir'] }}
        - migrate: True
        - require:
          - virtualenv: {{ pillar['website_venv_dir'] }}
          - file: django settings

State above actively uses pillar file ``pillar/website.sls``:

.. code-block:: yaml

    website_venv_dir: /home/jon_snow/venv
    website_venv_activate_path: /home/jon_snow/venv/bin/activate
    website_src_dir: /home/jon_snow/abstract-internal-messaging
    website_requirements_path: /home/jon_snow/abstract-internal-messaging/requirements_tests.txt
    website_settings_path: /home/jon_snow/abstract-internal-messaging/messaging/settings/local.py
    website_static_dir: /home/jon_snow/abstract-internal-messaging/collected_static/

    website_gunicorn_bin_path: /home/jon_snow/venv/bin/gunicorn
    website_gunicorn_conf_path: /home/jon_snow/gunicorn.conf.py

Here is ``salt/website/local.py.template``:

.. code-block:: python

    # coding: utf-8
    from .dev import *


    DATABASES = {
        'default': {
            'ENGINE': "django.db.backends.postgresql_psycopg2",
            'NAME': "{{ pillar['postgresql_db'] }}",
            'USER': "{{ pillar['postgresql_user'] }}",
            'PASSWORD': "{{ pillar['postgresql_password'] }}",
        }
    }

    SECRET_KEY = 'some secret key'

Website's WSGI Server state
---------------------------

``salt/website/wsgiserver.sls`` provides supervisored gunicorn running.

.. code-block:: yaml

    supervisord conf:
      file:
        - managed
        - name: /etc/supervisor/conf.d/website_gunicorn.conf
        - source: salt://website/supervisord.conf
        - template: jinja

    gunicorn conf:
      file:
        - managed
        - name: {{ pillar ['website_gunicorn_conf_path'] }}
        - source: salt://website/gunicorn.conf.py
        - user: jon_snow
        - group: jon_snow

    supervisor:
      pkg:
        - installed

    supervisored gunicorn:
      supervisord:
        - running
        - name: website_gunicorn
        - update: True
        - restart: True
        - watch:
          - file: supervisord conf
          - file: gunicorn conf
        - require:
          - pkg: supervisor

``salt/website/gunicorn.conf.py``

.. code-block:: python

    # coding: utf-8
    import multiprocessing


    bind = '127.0.0.1:5000'
    workers = multiprocessing.cpu_count() * 2

``salt/website/supervisord.conf``

.. code-block:: yaml

    [program:website_gunicorn]
    command = {{ pillar['website_gunicorn_bin_path'] }} -c {{ pillar['website_gunicorn_conf_path'] }} messaging.wsgi:application
    directory = {{ pillar['website_src_dir'] }}
    user = vagrant
    autostart = true
    autorestart = true
    redirect_stderr = True
    stdout_logfile = /var/log/supervisor/website_gunicorn.log

Website's Web Server state
--------------------------

``salt/website/webserver.sls`` installs latest stable Nginx, copies config
file and runs service.

.. code-block:: yaml

    nginx:
      pkgrepo:
        - managed
        - name: deb http://nginx.org/packages/ubuntu/ precise nginx
        - key_url: http://nginx.org/keys/nginx_signing.key

      pkg:
        - installed
        - require:
          - pkgrepo: nginx

      service:
        - running
        - watch:
          - pkg: nginx
          - file: /etc/nginx/nginx.conf

    /etc/nginx/nginx.conf:
      file:
        - managed
        - source: salt://website/nginx.conf
        - user: root
        - group: root
        - mode: 644
        - template: jinja

Notable in ``salt/website/nginx.conf`` is ``sendfile off``. It fixes
`trouble <http://jeremyfelt.com/code/2013/01/08/clear-nginx-cache-in-vagrant/>`_
when Nginx runs in a virtual machine environment.

.. code-block:: yaml

    worker_processes 2;


    events {
        worker_connections 1024;
    }


    http {
        include mime.types;
        default_type application/octet-stream;

        sendfile off;

        keepalive_timeout 65;

        server {
            listen 80;
            server_name messaging-part-1;

            location /static/ {
                alias {{ pillar['website_static_dir'] }};
            }

            location / {
                proxy_pass http://localhost:5000;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }

            # Redirects server error pages to the static page /50x.html
            error_page 500 502 503 504 /50x.html;
            location = /50x.html {
                root html;
            }
        }
    }

Top states
----------

There is only one thing to do — to add top files and this tree is over.

``salt/top.sls``

.. code-block:: yaml

    base:
      '*':
        - user
        - source_code
        - python
        - redis
        - postgresql
        - website.django
        - website.wsgiserver
        - website.webserver

``pillar/top.sls``

.. code-block:: yaml

    base:
      '*':
        - postgresql
        - website

Conclusion
----------

This article was mostly about states examples.
It's time to test configuration.

.. code-block:: console

    $ cd abstract-internal-messaging/vagrant
    $ vagrant up

In order to reach website you can add ``1.2.3.4 messaging-part-1`` to
``/etc/hosts``. Now it should work:

.. code-block:: console

    $ curl -i messaging-part-1

Tests should be passed as well:

.. code-block:: console

    $ vagrant ssh
    $ source /home/jon_snow/venv/bin/activate
    (venv)$ cd /home/jon_snow/abstract-internal-messaging
    (venv)$ ./manage.py test

.. _Messaging System: https://github.com/marselester/abstract-internal-messaging
