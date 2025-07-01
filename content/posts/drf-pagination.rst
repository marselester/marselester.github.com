========================================================
Django REST framework: pagination on PostgreSQL triggers
========================================================

:date: 2016-04-02
:tags: python, django, postgresql, django rest framework, pagination
:category: Python
:slug: drf-pagination

Django and Django REST Framework use SQL COUNT in pagination.
As your database grows SQL COUNT becomes too slow. Fortunately the frameworks
are well designed and allow to customize a way items are count.
Let me illustrate that on a typical "books" example.

.. code-block:: python

    class Author(models.Model):
        name = models.CharField()

        class Meta:
            db_table = 'author'


    class Book(models.Model):
        author = models.ForeignKey(Author, related_name='books')

        class Meta:
            db_table = 'book'

It has been working fine for some time but authors are so productive and wrote thousands of books.
As a result ``alice.books.count()`` takes a few seconds to execute.
One option is to store books count in ``Author`` model which might be helpful in
other SQL queries. Another one is to keep count somewhere else, e.g., Redis.

.. code-block:: python

    class Author(models.Model):
        name = models.CharField()
        books_count = models.PositiveIntegerField(null=True)

        class Meta:
            db_table = 'author'

Note that ``books_count`` is nullable due to existing data on production servers.
Setting a default value ``0`` on schema migration would take long to update
every author record, also it's hard to tell whether an author record is
already migrated (``books_count`` is calculated or not).

Now we have ``Author.books_count`` field which should keep track of new books.
You can implement that based on Django signals or SQL triggers.

Signals might fail, even with ``send_robust``. Updating a books count in
a signal receiver slows down an application (two to three DB queries and
acquiring a DB lock on author record):

.. code-block:: python

    @receiver(post_save, sender=Book, dispatch_uid='update_author_books_count')
    def update_author_books_count(sender, instance, created, **kwargs):
        if not created:
            return
        book = instance

        with atomic():
            author = Author.objects.select_for_update().get(pk=book.author_id)
            if author.books_count is None:
                author.books_count = author.books.count()
            else:
                author.books_count += 1
            author.save()

Moving an update logic from a signal receiver to Celery task doesn't look
promising -- ``books_count`` would be inconsistent between a task trigger and
its execution or simply Celery task might fail. Another issue is that
a model signal is not "attached" to DB transaction. For example,
a Celery task is run before a DB transaction is committed.
That is usually quick fixed with ``task_name.apply_async(countdown=1)``
to delay a task execution, but I would rather recommend something like
`django-transaction-hooks`_.

What about SQL trigger option? A bad thing is that a business logic leaks from an app to DB.
But with a good documentation and tests it's maintainable: SQL code is stored in
a data migration module.

.. code-block:: console

    $ python manage.py makemigrations --empty yourappname

.. code-block:: python

    from django.db import migrations

    CREATE_FUNCTION_SQL = """
    CREATE OR REPLACE FUNCTION update_books_count_on_author() RETURNS trigger AS $$
        BEGIN
            UPDATE author SET books_count =
                CASE
                    WHEN books_count IS NULL THEN (
                        SELECT count(*) FROM book WHERE author_id = author.id
                    )
                    ELSE books_count + 1
                END
            WHERE id = NEW.author_id;

            RETURN NEW;
        END;
    $$ LANGUAGE plpgsql;
    """

    CREATE_TRIGGER_SQL = """
    CREATE TRIGGER books_count_update
        AFTER INSERT ON book
        FOR EACH ROW
        EXECUTE PROCEDURE update_books_count_on_author();
    """


    class Migration(migrations.Migration):
        dependencies = []
        operations = [
            migrations.RunSQL(CREATE_FUNCTION_SQL),
            migrations.RunSQL(CREATE_TRIGGER_SQL),
        ]

Here is a pitfall though. When a new book is created and author is updated on the
same DB transaction, then ``books_count`` value might be overwritten.

.. code-block:: python

    with atomic():
        author = Author.objects.select_for_update().get(pk=author_id)
        Book.objects.create(author=author)
        author.name = 'Bob'
        author.save()

You can either explicitly list fields to update ``author.save(update_fields=['name'])``
or use `django-save-the-change`_. Let's document that in the model docstring.

.. code-block:: python

    from save_the_change.mixins import SaveTheChange


    class Author(SaveTheChange, models.Model):
        """Author, e.g., Terry Pratchett.

        :attribute books_count: How many books writer has. SQL COUNT is
            expensive operation, so we store calculated value and update it
            by SQL trigger (check a data migration module for details).
            It's important to save only fields that were updated in the model.
            Otherwise SQL trigger's results are overwritten by Django ORM.
            For example:

            1. author is requested with a lock (books_count = 1)
            2. new book is created
            3. SQL trigger updates author's books_count field (now it is 2)
            4. author instance is saved with the old value of books_count = 1.

            SaveTheChange mixin helps to prevent it.

        """

To benefit from ``books_count`` field in Django REST Framework we need
a custom pagination class which implements ``Paginator.count`` property.
The idea is to extract author ID from paginator's SQL, query a books count
from ``Author`` model and return it, instead of default
``Book.objects.filter(author_id=author_id).count()``.

.. code-block:: python

    from django.core.paginator import Paginator
    from rest_framework import pagination
    from rest_framework.viewsets import ReadOnlyModelViewSet


    class BookViewSet(ReadOnlyModelViewSet):
        pagination_class = BookPagination


    class BookPagination(pagination.PageNumberPagination):
        django_paginator_class = CachedBookCountPaginator


    class CachedBookCountPaginator(Paginator):
        @cached_property
        def count(self):
            """Return the total number of books, across all pages.

            It parses a SQL and learns what author ID was requested
            based on ``self.object_list.query``. After that we can get
            a cached books count from Author model.

            """
            # There is query.where, but I could't find an author ID easily.
            # Moreover query.where internals might be changed.
            sql = str(self.object_list.query)
            author_id = self._get_author_id_from_sql(sql)

            author = Author.objects.get(pk=author_id).only('books_count')
            # In case we got unsynced author, we fallback to SQL COUNT.
            if author.books_count is None:
                return self.query_count()

            return author.books_count

        def query_count(self):
            """Request books count from DB.

            We need this method to facilitate testing (mocks).

            """
            return super(CachedBookCountPaginator, self).count

    @classmethod
    def _get_author_id_from_sql(cls, sql):
        pass

I hope this helps. Cheers!

.. _django-transaction-hooks: https://django-transaction-hooks.readthedocs.org/en/latest/
.. _django-save-the-change: https://django-save-the-change.readthedocs.org/en/latest/
