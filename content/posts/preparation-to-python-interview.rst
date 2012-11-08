===============================
Preparation to Python Interview
===============================

:date: 2012-11-02
:tags: python, interview
:category: Python

I decided to collect a little more information and experience during
preparation to Python developer interview. These are some information and
links which seemed important to me. Maybe it will be helpful.

How does it usually go?
-----------------------

What kind of projects did you participate in?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

What did you do at your previous job? It is expected that you will told
the essence in simple words.

Tricky question
~~~~~~~~~~~~~~~

It is expected that you will search for solution of task independently.
Reasonings aloud are welcomed.

Writing code
~~~~~~~~~~~~

Interviewer is interested in critical analysis of code. For example,
efficiency of used data structures, algorithm's complexity evaluation.

Design
~~~~~~

In this step it is important to ask as much as possible about a task before
starting to look for solution.

Provocative question
~~~~~~~~~~~~~~~~~~~~

You have to stay in your lane.

Questions
---------

Basic
~~~~~

Probably interviewer starts with basic questions. Let us see example::

    funcs = []
    for i in range(5):
        def f():
            print i
        funcs.append(f)

    for f in funcs:
        f()

I guess he examines knowledge about namespace. It will print ``4`` five times.
To explain that you supposed to know about LEGB_ rule. Also you should know
that variable search in enclosed scope will be done later, **after call** of
enclosed functions. They all get same value -- value of ``i`` in last
iteration.

Next example prints numbers from 0 to 4::

    funcs = []
    for i in range(5):
        def f(i=i):
            print i
        funcs.append(f)

    for f in funcs:
        f()

It happens because default value is stored **when enclosed function was
created**.

There is another popular question related with default value of function.
N.B. ``names`` is mutable::

    def f(names=[]):
        names.append('some name')

`Visualizing code execution`_ and `quiz of non-trivial features of Python`_
will be helpful to prepare for questions which are mentioned above.

What do you think about this code?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    d = {1: 'a', 2: 'b', 3: 'c'}


    def safe_get(d, key, value):
        if key in d.keys():
            return d[key]
        return value

    print safe_get(d, 0, 'blah')

Apparently interviewer expects these answers:

- it repeats functionality of ``d.get(0, 'blah')``.
- it uses ``if key in d.keys()`` instead of ``if key in d``.
- actually it is looking for ``key`` in ``[1, 2, 3]`` (for Python 2).
  Therefore O(n) is worse than O(1) for dictionary lookup.

It would be appropriate to read about `data structures`_ and their
`time complexity`_, about collections_, itertools_ and `sorting`_.

Advanced knowledge
~~~~~~~~~~~~~~~~~~

These articles are for complete picture of Python:

- `Python Types and Objects`_
- `What is a metaclass in Python?`_
- `Python Attributes and Methods`_, `Пользовательские атрибуты в Python`_

Misc
~~~~

Questions:

- `What Every New Python/Django Web Developer Should Know in 3 Months <http://pragmaticstartup.wordpress.com/2012/08/25/what-every-new-pythondjango-web-developer-should-know-in-3-months/>`_
- `Python Interview Questions <https://groups.google.com/d/topic/comp.lang.python/rhW_rIYY5HM/discussion>`_
- `Python interview questions and answers <http://www.techinterviews.com/python-interview-questions-and-answers>`_
- `Что спрашивают на собеседовании в Яндекс? <http://habrahabr.ru/qa/5783/>`_
- `Вопросы и задания по Python <http://pyobject.ru/blog/2010/02/04/python-quiz/>`_

Useful articles:

- `Dealing with engineers that frequently leave their jobs <http://programmers.stackexchange.com/questions/43409/dealing-with-engineers-that-frequently-leave-their-jobs>`_
- `Get that job at Google <http://steve-yegge.blogspot.ch/2008/03/get-that-job-at-google.html>`_
- `Hidden features of Python <http://stackoverflow.com/questions/101268/hidden-features-of-python>`_

.. _LEGB: http://stackoverflow.com/questions/291978/short-description-of-python-scoping-rules
.. _Visualizing code execution: http://pythontutor.com
.. _quiz of non-trivial features of Python: https://alexbers.com/python_quiz/
.. _data structures: http://habrahabr.ru/post/128457/
.. _time complexity: http://wiki.python.org/moin/TimeComplexity
.. _collections: http://docs.python.org/3/library/collections.html
.. _itertools: http://docs.python.org/3/library/itertools
.. _sorting: http://wiki.python.org/moin/HowTo/Sorting/
.. _Python Types and Objects: http://www.cafepy.com/article/python_types_and_objects/
.. _What is a metaclass in Python?: http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python/6581949#6581949
.. _Python Attributes and Methods: http://www.cafepy.com/article/python_attributes_and_methods/
.. _Пользовательские атрибуты в Python: http://habrahabr.ru/post/137415/
