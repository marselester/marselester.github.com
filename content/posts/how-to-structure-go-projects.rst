============================
How to Structure Go Projects
============================

:date: 2018-09-28
:tags: golang, project structure, Domain-Driven Design
:category: Go
:slug: how-to-structure-go-projects

I came to Go from Django where the framework defines project layout, thus I wanted to know
how to structure my Go applications. After reading documentation and building a few Django projects,
you get a clear mental picture, as most of the questions are already answered.
That helps to keep the projects within a company consistent, so developers don't have to struggle
when they land on a new codebase. I was looking for a similar framework approach in Go,
but none of them felt right to me. For some reason the same concepts do not resonate with Go.
I didn't have better ideas, so I watched a few videos from conferences.
Here are the talks I found most helpful and I use them as guidelines:

- Ben Johnson's `Structuring Applications for Growth <https://www.youtube.com/watch?v=LMSbsW1Xpwg>`_
- Brian Ketelsen's `Write Code Like the Go Team <https://www.youtube.com/watch?v=MzTcsI6tn-0>`_,
  `slides <https://talks.bjk.fyi/gcru18-best.html#/>`_
- Kat Zien's `How Do You Structure Your Go Apps <https://www.youtube.com/watch?v=oL6JBUk6tj0>`_
- Peter Bourgon's `Ways To Do Things <https://www.youtube.com/watch?v=LHe1Cb_Ud_M>`_.
  I tried his ideas in `RascalDB 😜 <https://github.com/marselester/rascaldb/blob/master/rascaldb.go>`_
  to serialize concurrent DB actions.

I encourage you to watch those talks and go over code examples they provided.

The following is my takeaway which could be completely misleading and far from the original ideas
(just in case, you don't have to apply all of them together).
If you feel something doesn't bring enough value in your case, just skip it, trust your intuition.
Development requires many iterations to bounce ideas until the result becomes good enough.

Know Your Domain
----------------

I like to start a new project on A4 paper and dump everything I know in free form
(what the problem are you facing, can you solve it without writing code,
what are possible solutions (pros/cons), terminology, system requirements, API endpoints, sketches,
doodles, whatever gets your train of thoughts moving). Paper helps to get away from constraints of computer,
I can write and draw without any programs which would otherwise have taken time and stolen my brain cycles.
Paper helps me to stay focused — nothing blinks, pops up or rings, there is no urge to multitask
(check email/Slack, switch between editor and console as if there is something new).

Once you wrote everything down, you might realize there is not enough information to make progress.
So you can reach out to stake holders and get more insights using your notes.
Don't forget to write new info as well. It might take a few iterations when you finally establish
common terminology, refine the actual project's goal, and cut unnecessary features/requirements.

Now you have a mental picture of the project with clear deliverables.
Based on that I like to spend time to come up with a concise project/repository name which
summarizes nicely the project's goal and spirit (thesaurus comes to the rescue).
As a next step I usually create README file where insights from the paper are documented.
This time you want everyone in the world to understand what you learned about the project.
This helps you to iterate once more as you're documenting in README, and the end result
can be shared with coworkers (they might not understand your handwriting and doodles).
If they have questions, that means you have a room for improvement, since other people
don't have a context you obtained. Take your time and write it down, this will ensure that
you will be able to get help with the project much easier (when you're on vacation or
have to hand over the code). With some practice you'll anticipate questions and
provide answers in README, so it won't take as much time from your coworkers next time.
Keeping in mind that you are ultimately solving problems for your customers and
designing the system and writing code for your coworkers (not just for yourself)
helps to rigorously document as you go. Or else prepare to answer the same questions
over and over again. In order to get better at documenting, I recommend reading
`A beginner’s guide to writing documentation <http://www.writethedocs.org/guide/writing/beginners-guide-to-docs/>`_
and `Documentation Guide <http://www.writethedocs.org/guide/>`_.

Let's say you're building an API client, then it makes sense to start from
a usage example in the README, since most of the documentation work is done by API provider.
How will people download your library? How will they import it, does the path look concise and clear?
How will your end user configure a client? What use cases your users might have (custom API domain,
timeouts, logging, instrumentation)? Think of a few workflows and see how would you handle errors.
Best case scenario is using your own library to reveal pain points.
Check out `one of my attempts <https://github.com/marselester/bitgo-v2>`_ to find a decent layout of
an API library. The library has a configurable client (you can instantiate many of them, e.g., one per coin)
which allows to use a custom logger, http.Client, etc. It also has embedded services — things that know
how to speak with particular API endpoints to operate API resources.

Getting to the Point
--------------------

As you can see from the talks the main theme in Go application structure is heavily
influenced by Domain-Driven Design (DDD). Using the research done on your project,
you shall write down entities (as Go structs) of the domain model and services (as Go interfaces)
which perform operations over those entities.

I should mention that for simplicity's sake in my projects I decided to combine DDD "service" and
"repository" concepts under "service" term.

Let's proceed with listing minimal set of operations over the key entities to keep the scope small.
It helps to imagine that there are different service implementations,
an entity can be stored anywhere, for example, in Postgres, Kafka, memory, Redis,
JSON file, remote API. But keep in mind semantics that storage of choice provides:
getting a list of entities from Kafka and Postgres are hard to abstract (streaming vs quering).
Moreover, if you do so, you might create unnecessary constraints for yourself.
In reality, it is very unlikely that a project will change storages often,
since you've chosen them for a reason (the choice of the storage is influenced
by a problem you're solving, its constraints and guarantees you need).
Most likely you've already chosen the storages in advance, so make sure you
don't fight with interfaces you defined in your domain model. If you know your services
need to share the same SQL transaction, embrace it in interfaces, for instance:

.. code-block:: go

    type PaymentService interface {
      CreatePayment(ctx context.Context, tx *sql.Tx, *Payment) error
    }

Do you put all operations (add/list) into one service interface? I don't know.
You have to decide. If you add an entity to Postgres, you expect to list them
from Postgres as well. From this perspective "add" and "list" operations should be
under the same interface. If in-memory storage can't implement the service interface,
perhaps you don't have to enforce operation under the same interface,
then probably it's ok to split it.

Ben Johnson defines entities in a single package where domain model is described — the package
shouldn’t have third party dependencies. Whereas Kat Zien in her demo created a package per service:
"adder" package that adds beers and reviews, "lister" service lists beers and reviews.
Each package defines its own beer and review structs.

In my projects I isolate services implementation in a single package.
For example, if I stored beer reviews in Kafka, I would have a kafka package which exposes
a client and beer/review services embedded in it. The same applies to postgres package — two services use
the same db connection pool and beer/review entities might appear in the same db transaction.

During coding I combine the service implementations into a program (some server or
a ctl tool) kept in cmd directory. That helps me to validate design ideas and notice any awkward
component integrations. Similar to service implementations, try to think where the input and
output could be coming from/to: standard input/output, http, rpc, db.

An Example
----------

Now let's have a look at `distributed payment <https://github.com/marselester/distributed-payment>`_
demo project where I explored an idea of payment transaction
without atomic commit across 3 Kafka partitions.

The domain model is defined in the repository root (note, you can place your packages
in "internal" directory, so you don't mix them up with unrelated files):

- `wallet.go` has `Transfer`, `Payment` entities, and services `TransferService`, `PaymentService`
  which can create and list the entities. Since the project is based on Kafka,
  the interfaces reflect that (`partition`, `offset` params). The services accept `context.Context`
  as a first argument, because we should be able to tell implementations to cancel operation.
  `OpenTracing <https://github.com/opentracing/opentracing-go>`_ can leverage context as well.
  Pay attention to a pointer/value semantics (share or not) in the service interfaces.
  Since an entity in DDD terms has a unique identity, a pointer semantics was used,
  hence `*Transfer` is passed to `CreateTransfer()` and returned from `FromOffset()`. Have a look at
  `Design Philosophy On Data And Semantics <https://www.ardanlabs.com/blog/2017/06/design-philosophy-on-data-and-semantics.html>`_
  for more insights.
- `error.go` contains errors which are relevant to the whole domain model,
  `Failure is your Domain <https://middlemost.com/failure-is-your-domain/>`_.
  On implementation level there could be their own specific errors, for example, HTTP API errors in
  `rest/error.go` define JSON and validation errors.
- `log.go` borrows `Logger` interface from Go kit. Since logging is an integral part of the system,
  placing it nearby the domain model seems justified. There is
  `Standard logger interface <https://github.com/go-commons/commons/issues/1>`_ discussion
  where the consensus is to emit events instead of logging in the library.
  Best practices and examples of how to emit events is still an
  `open topic <https://github.com/go-commons/event/issues/1>`_ at the time of writing.

Implementations of the services defined in `wallet.go` are isolated in packages
by their dependency name, for example, kafka, rest, mock, rocksdb.

Package kafka implements wallet services and provides the Client access to them.
There were two design options: embed the services to the Client struct or
inject a service into each other. The example below would allow to have
a swappable `PaymentService` ("pg" refers to a Postgres implementation):

.. code-block:: go

    kafka.TransferService.PaymentService = pg.NewPaymentService()

On the other hand, grouping services in the Client would let services maintain DB transactions
by sharing the same `*sql.DB`. Here is `pg.Client` example:

.. code-block:: go

    // Client represents a client to the underlying PostgreSQL data store.
    type Client struct {
      Transfer *TransferService
      Payment  *PaymentService

      logger wallet.Logger

      db        *sql.DB
      transferQ map[string]string
      paymentQ  map[string]string
    }

In `distributed-signup <https://github.com/marselester/distributed-signup/blob/master/pg/user_service.go>`_
project a Client concept is baked into `UserService`, because it was the only service that
needed access to Postgres.

.. code-block:: go

    // UserService reprensets a service to store signed up users.
    type UserService struct {
      config Config

      pool *pgx.ConnPool
    }

Package rest is responsible for translating incoming HTTP requests to wallet domain and
then translating results from wallet model back to HTTP responses.
The package doesn't implement `TransferService` per se, it uses one in its Server.
The REST-style API server itself is put together in
`cmd/transfer-server <https://github.com/marselester/distributed-payment/blob/master/cmd/transfer-server/main.go>`_.

.. code-block:: go

    // Server represents an HTTP API handler for wallet services.
    // It wraps a TransferService so we can provide different
    // implementations, e.g., Kafka or a mock.
    type Server struct {
      *chi.Mux
      logger          wallet.Logger
      transferService wallet.TransferService
      wopts           walletOption
    }

Originally in `WTF Dial: HTTP API <https://medium.com/wtf-dial/wtf-dial-http-api-d8800ccd605f>`_
Ben Johnson explained how to implement API properly and isolate http dependencies in wtf/http package.

Package mock provides mock services to facilitate testing. For example, for most cases
we do not need Kafka implementation of a transfer service to be used in HTTP API testing.

Package rocksdb implements user requests deduplication using RocksDB to
memorise already processed request IDs. Requests deduplication is an integral part of
a distributed system, hence the domain model must embrace it.

Everything is connected in cmd directory. Note, that the domain package is used everywhere.

- `cmd/transfer-server` is HTTP API server to create money transfers which are stored in Kafka.
  It delegates the actual hard work to kafka and rest packages.
- `cmd/paymentd` program is responsible for creating incoming & outgoing payment pairs based on
  money transfer requests stored in Kafka.
- `cmd/accountantd` is the last program in the pipeline. It sequentially reads Kafka messages
  from `wallet.payment` topic, deduplicates messages by request ID, and applies the changes to
  the account balances. Deduplication is provided by rocksdb package mentioned above.

To wrap up, that's all I managed to recall :) I look forward for more talks on structuring Go applications.
