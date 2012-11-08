======================================================================
Краткий обзор инфраструктуры для разработки reusable Django приложений
======================================================================

:date: 2012-06-13
:tags: python, django, infrastructure
:category: Python

Начиная впервые разрабатывать веб-приложения на новом фреймворке программист
зачастую сталкивается с некоторыми трудностями. При разработке отчуждаемых
веб-приложений на Django к этим проблемам необходимо отнести организацию
файлов в проекте, обнаружение тестов, вопросы пакетирования приложений и
организации автоматизированного тестирования. В данной статье приведены пути
решения этих проблем.

Важно знать различия между двумя способами разработки Django приложений.
Первый способ заключается в создании приложения внутри каталога проекта с
помощью команды ``manage.py startapp``. Второй способ – разработка
независимого приложения. Автономное приложение проще использовать и
распространять [#Bennett]_.

Системы пакетирования
=====================

Для того чтобы Django смог найти изолированное приложение, нужно чтобы оно
находилось в Python пути. Python путь – это список каталогов, в которых Python
ищет модули, всякий раз исполняя оператор ``import``. При установке
интерпретатора Python создается директория ``site-packages`` и добавляется в
Python путь. Все сторонние приложения устанавливаются в ``site-packages``.

Python имеет систему пакетов, которая позволяет распространять программы и
библиотеки в стандартном формате, что помогает легко устанавливать и
использовать их. Помимо распространения пакетов Python также обеспечивает
централизованный сервис для контрибуции пакетов. Этот сервис называется Индекс
Python Пакетов (PyPI). Он позволяет разработчикам распространять пакет
большому сообществу с небольшими затратами усилий. Использование пакетов дает
такие преимущества, как:

- управление зависимостями пакетов;
- получение информации об установленных пакетах (например, версия пакета);
- удаление пакетов;
- поиск пакетов по PyPI [#Ziadé]_.

Текущее состояние систем пакетирования показано на рисунке. Если
разрабатываемому пакету необходим модуль *Setuptools*, рекомендуется
установить *Distribute*, который является обновленной версией оригинального
*Setuptools*. *Distribute* был создан так как *Setuptools* больше не
поддерживается разработчиками. Модуль *distutils* является частью стандартной
библиотеки Python и обеспечивает основу для пакетирования. В будущем
*distutils2* заменит *Setuptools* и *distutils*, а также устранит
необходимость в *Distribute*. До завершения работ над *distutils2*
разработчикам рекомендуют использовать *distutils* [#Ziadé]_.

.. image:: http://guide.python-distribute.org/_images/state_of_packaging.jpg

Управление экосистемой пакетирования
====================================

Для манипулирования экосистемой пакетирования были разработаны такие
инструменты, как Pip и Virtualenv. Pip – это инструмент для установки и
управления Python пакетами. С его помощью можно:

- устанавливать пакеты, например, ``pip install Django``;
- показывать список установленных пакетов и их версии, например,
  ``pip freeze`` отобразит ``Django==1.3.1, pep8==0.6.1,
  virtualenv==1.7.1.2``;
- устанавливать пакеты определенных версий, например,
  ``pip install 'Mark-down>2.0,<2.0.3'``;
- обновлять пакеты, например, ``pip install --upgrade Django``;
- удалять пакеты, например, ``pip uninstall Markdown``.

Для того чтобы не смешивать экспериментальные пакеты со стабильными пакетами,
разработчики используют Virtualenv. Данный инструмент позволяет создавать
изолированные Python окружения и устанавливать в них пакеты, не модифицируя
системное окружение Python. Например, ``virtualenv --no-site-packages my_env``
создаст окружение ``my_env`` без пакетов, ``source my_env/bin/activate``
активирует окружение ``my_env``, а ``deactivate`` отключит его.

Метаданные проекта
==================

Наименьший проект на Python состоит из двух файлов. Файл ``setup.py``, который
содержит метаданные о проекте и файл, который содержит Python код для
реализации функциональности проекта. В ``setup.py`` есть только три поля,
необходимые для заполнения: *name*, *version* и *packages*. Поле *name* должно
быть уникальным для публикации в PyPI. Поле *version* следит за различными
версиями проекта. Поле *packages* описывает, где размещен Python код проекта
[#Ziadé]_. Ниже приведен пример файла ``setup.py``, который также включает
информацию о лицензии и описание проекта::

    #!/usr/bin/env python
    from distutils.core import setup

    setup(
        name='django-todo',
        version='0.1dev',
        long_description=open('README.rst').read(),
        licence='MIT license',
        packages=[
            'todo',
            'todo.templatetags',
        ]
    )

Структура репозитория
=====================

Для удобства разработки в Django существует стандарт организации проекта и
стиль кодирования [#Django]_. При разработке приложений автор в большей
степени опирается на них, а также на практический опыт компании Lincoln Loop
[#LincolnLoop]_. Руководство по стилю кодирования и автоматизация имеет важное
значение для цикла разработки. Структура репозитория в той же степени является
важной частью архитектуры проекта. Рассмотрим некоторые особенности:

- ``README.rst`` должен содержать описание проекта в формате reST;
- ``LICENSE`` должен содержать полный текст лицензии;
- аргумент *install_requires* в ``setup.py`` нужен для перечисления
  зависимостей проекта;
- ``MANIFEST.in`` – это шаблон, который определяет, какие файлы должны быть
  включены в пакет, например, ``include README.rst``;
- в ``requirements.txt`` следует указывать зависимости, необходимые для
  участия в разработке проекта (тестирование, генерация документации)
  [#Reitz]_;
- ``todo/__init__.py``;
- ``todo/models.py``;
- ``docs/conf.py``;
- ``docs/index.rst``;
- ``tests/__init__.py``;
- ``tests/models.py``.

Обнаружение и запуск тестов
===========================

Тесты не следует распространять вместе с модулем, так как это приводит к
увеличению сложности для конечных пользователей – наборы тестов требуют
дополнительных зависимостей. На конференции *PyCon US 2012* Карл Майер
предложил решение [#Meyer]_, которое позволило отделить тесты от приложений в
проекте и реализовать обнаружение и запуск всех тестов из каталога ``tests``.
Автор применил данное решение для организации тестов в многоразовых Django
приложениях [#app_skeleton]_. В корне репозитория располагается скрипт
``runtests.py``, который запускает тесты::

    #!/usr/bin/env python
    import os
    import sys

    os.environ['DJANGO_SETTINGS_MODULE'] = 'tests.settings'

    from django.test.utils import get_runner
    from django.conf import settings


    def runtests():
        TestRunner = get_runner(settings)

        test_runner = TestRunner(verbosity=1, interactive=True, failfast=False)
        failures = test_runner.run_tests([])
        sys.exit(failures)

    if __name__ == '__main__':
        runtests()

Настройки для их запуска указаны в файле ``tests/settings.py``::

    import os

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
        }
    }
    INSTALLED_APPS = (
        'app_name',
    )

    BASE_PATH = os.path.dirname(os.path.dirname(__file__))
    TEST_DISCOVERY_ROOT = os.path.join(BASE_PATH, 'tests')

    TEST_RUNNER = 'tests.runner.DiscoveryDjangoTestSuiteRunner'

    FIXTURE_DIRS = (
        os.path.join(TEST_DISCOVERY_ROOT, 'fixtures'),
    )

Обнаружение тестов осуществляется во всех файлах, которые находятся в каталоге
``tests`` и название которых совпадает с *models.py*, *tests.py* или
*test\*.py*.

Автоматизация тестирования
==========================

Для автоматизации тестирования Python проектов автор использует инструмент
tox. Он может быть использован:

- для проверки, что пакеты устанавливаются правильно в разных версиях Python;
- для запуска тестов в каждой из сред;
- в качестве интерфейса для сервера непрерывной интеграции, например, Jenkins.

Ниже приведен пример конфигурации ``tox.ini`` со средами *Python 2.6*,
*Python 2.7* и *Django 1.3* [#django_todo]_::

    [tox]
    envlist=py26,py27,dj13

    [testenv]
    deps=
        django==1.4.0
        git+https://github.com/rbarrois/factory_boy.git
        webtest
        django-webtest

    commands=python runtests.py

    [testenv:dj13]
    deps=
        django==1.3.1
        git+https://github.com/rbarrois/factory_boy.git
        webtest
        django-webtest

Окружение *testenv* является средой по умолчанию. В ней описаны пакеты с
указаниями версий, которые необходимы для тестирования проекта (в данном
случае это фреймворк Django версии 1.4.1, последние версии инструментов для
тестирования – factory_boy, webtest, django-webtest).

.. [#Bennett] Bennett B. Practical Django Projects.

.. [#Ziadé] Ziadé T. `The Hitchhiker's Guide to Packaging
   <http://guide.python-distribute.org/>`_.

.. [#Django] Django community. `Django Coding Style
   <https://docs.djangoproject.com/en/dev/internals/contributing/writing-code/
   coding-style/>`_.

.. [#LincolnLoop] Lincoln Loop company. `Django Best Practices
   <http://lincolnloop.com/django-best-practices/>`_.

.. [#Reitz] Reitz K. `Repository Structure and Python
   <http://kennethreitz.com/repository-structure-and-python.html>`_.

.. [#Meyer] Meyer C. `Testing and Django
   <http://carljm.github.com/django-testing-slides/>`_ at PyCon US 2012.

.. [#app_skeleton] Мавлеткулов М. `Reusable Django app skeleton
   <https://github.com/marselester/reusable-django-app-skeleton>`_.

.. [#django_todo] Мавлеткулов М. `Система управления цепочками задач
   <https://github.com/marselester/django-todo>`_.
