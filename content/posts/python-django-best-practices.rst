=========================================
Соглашения по разработке на Python/Django
=========================================

:date: 2012-06-29
:tags: python, django, best practices
:category: Python
:slug: links-to-best-practices-of-python-django

Во время разработки я часто сверяюсь с известными мне соглашениями,
стараюсь следовать рекомендациям. Цитировать их не имеет смысла -- лучше
приведу ссылки.

`PEP 8 -- Style Guide for Python Code
<http://www.python.org/dev/peps/pep-0008/>`_.

`Code Like a Pythonista: Idiomatic Python
<http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html
#long-lines-continuations>`_.
В нем я нашел ответы на вопросы форматирования длинных строк::

    expended_time = (self.finish_date() - self.start_date
                     + datetime.timedelta(days=1))

`Google Python Style Guide
<http://google-styleguide.googlecode.com/svn/trunk/pyguide.html>`_.

`Coding style <https://docs.djangoproject.com/en/dev/internals/contributing/
writing-code/coding-style/>`_ из документации Django.

`Django Best Practices <http://lincolnloop.com/django-best-practices/>`_ от
компании Lincoln Loop.

`The Hitchhiker’s Guide to Python!
<http://docs.python-guide.org/en/latest/index.html>`_ от Kenneth Reitz.

`The Hitchhiker’s Guide to Packaging <http://guide.python-distribute.org/>`_
от Tarek Ziadé.

`Elements of Web Architecture <http://www.s7labs.com/learn/ewa/>`_ от s7labs.

`Твиты <http://twitter.com/getpy>`_ с ссылками на интересные статьи по Python
от `@czheng <https://twitter.com/czheng>`_.

Рекомендации по именованию от Макконелла
========================================

Читать как мантру.

Именование переменных
---------------------

Имя должно полно и точно описывать сущность. Формулирование сути переменной в
словах.

Имя чаще всего описывает проблему реального мира, а не ее решение на языке
программирования.

Значимая часть имени переменной располагаться в начале и читается первой.
Спецификаторы вычисляемых значений находятся в конце::

    revenue_total
    expense_average

Не используйте "Number" из-за двойного смысла (количество и номер). Лучше::

    customer_count
    customer_index

Именование булевых переменных в утвердительной форме::

    done
    success или ok
    found
    error

Имена должны легко читаться.

Избегайте имен, имеющих:

- похожие значения: ``file_number, file_index``;
- похожие звучания: ``wrap, rap``;
- похожие имена: плохо -- ``client_recs, client_reps``, хорошо --
  ``client_records, clients_reports``;
- цифры: ``file1, file2``.

Не используйте "магические числа", заменяйте их абстрактными сущностями::

    CYCLES_NEEDED = 5

Именование функций
------------------

Если функция возвращает значение, в имени должно быть указано описание
возвращаемого значения.

Глагол *get* здесь излишен::

    cos()

    pen.current_color()
    user.last_id()

Для именования процедуры используйте **глагол + объект**, над которым
выполняется действие::

    print_document()
    repaginate_document()

    document.print()
    order_info.check()
    monthly_revenue.calc()
