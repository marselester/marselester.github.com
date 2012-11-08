============================
Разделение настроек в Django
============================

:date: 2012-06-29
:tags: python, django, settings
:category: Python
:slug: splitting-settings-in-django

В `Django wiki <https://code.djangoproject.com/wiki/SplitSettings>`_ собраны
различные способы разделения настроек. Мне нравится `вариант
<http://senko.net/en/django-quickstart-skeleton-project/>`_, описанный в блоге
Senko Rašić::

    settings/
    ├── __init__.py
    ├── base.py
    ├── development.py
    ├── local.py
    └── production.py

``base.py`` содержит общие настройки для ``development.py`` и
``production.py``, например::

    ADMINS = ()
    MANAGERS = ADMINS

    TIME_ZONE = 'Asia/Yekaterinburg'
    # ...

``production.py`` содержит настройки для эксплуатации. Как минимум, необходимо
выключить режим отладки::

    from .base import *

    DEBUG = False
    TEMPLATE_DEBUG = DEBUG
    # ...

``development.py`` содержит настройки, необходимые для разработки. Настройки
должны быть **одинаковы** и **полезны для всех участников** процесса
разработки, например::

    from .base import *

    DEBUG = True
    TEMPLATE_DEBUG = DEBUG
    # ...

Все индивидуальные настройки разработчика необходимо вынести в ``local.py``.
Например, это могут быть настройки подключения к БД, любимые инструменты
разработки и так далее::

    from .development import *

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': "developer's db name",
            'USER': "developer's db user",
            'PASSWORD': "developer's db password",
            'HOST': ''
            'PORT': '',
        }
    }

На production сервере ``local.py`` обычно содержит настройки подключения к
БД::

    from .production import *

``local.py`` не должен отслеживаться VCS.

Для Django 1.4 в файлах ``repo-name/project_name/wsgi.py`` и
``repo-name/manage.py`` нужно указать путь до ``local.py``::

    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "project_name.settings.local")

У меня не получилось разместить ``local.py`` на `heroku
<http://www.heroku.com/>`_, чтобы файл не находился под контролем git (плохо
искал?). Самое простое решение -- избавиться от ``local.py``, в ``manage.py``
указать ``project_name.settings.development``, а в ``wsgi.py`` --
``project_name.settings.production``. Еще можно использовать
``heroku config``::

    $ heroku config:add SECRET_KEY=my_unique_secret_key
