# Vivaldi
This document contains a high-level description on the inner workings of the new
data-mapping ORM / ODM codenamed *Vivaldi*.

*Vivaldi* is an extremely modularised persistence management mechanism. This 
document aims to serve as an overview by providing short introductions to the 
individual modules and placing them in the context of the purpose they are meant 
to fulfil.

## Introduction
*Vivaldi* isn't really an ORM. Well, it is, but it's not **just** an ORM.
After all, ORM stands for `Object-Relational Mapper`, while *Vivaldi* is more of 
an `Object-Anything Mapper`.

Being an object-to-anything mapper for PHP projects (and, in particular, *my* 
PHP projects) *Vivaldi* makes no attempt to follow [JPA](https://en.wikipedia.org/wiki/Java_Persistence_API). 

Rather than a database mapper, *Vivaldi* is a `persistence manager`. Its main 
purpose is to convert domain models to- and from storage, without caring too 
much about which storage mechanism is used.

### History
Although there are many ORM systems available for the PHP environment, all of 
them with **one** exception are based on the [*Active Record* pattern](https://en.wikipedia.org/wiki/Active_record_pattern).
The *Vivaldi* project was born from a disagreement with the status quo.

#### Active Record Pattern
The author of this document is of the humble opinion that the Active Record 
pattern is, on itself, unsuitable to manage the persistence of proper domain 
models.
Active Record implementations couple the models to the ORM, causing a vendor 
lock-in and a breach of the single responsibility principle, as well as making 
the system a lot harder to test.

To mitigate this problem, some developers build two models instead of one: they 
make their ORM-coupled models responsible for producing the domain models, and 
build repositories that initiate that process.

From the perspective of a repository, thit flow looks somewhat like this:
```php
class UserRepository implements AllUsers
{
    public function withId(int $id): User
    {
        return OrmUserModel::findById($id)->toDomainModel();
    }
}
```

The OrmUserModel would extend from the ORM's classes, but add a method in the 
spirit of:
```php
class OrmUserModel extends MyOrmVendor\Orm
{
    public function toDomainModel(): User
    {
        return new User(
            new UserId($this->id),
            new UserName($this->first_name, $this->last_name), 
            new Address($this->street, $this->number, new PostalCode($this->code))
        );
    }
}
```

The advantage of this way working is that the domain model itself remains 
completely free of persistence-related responsibilities, and of any kind of ORM 
restrictions.

Disadvantages include a lot of "manual" work to map the objects. 
It introduces the need to maintain **two** representations of each domain 
concept.

Such setup also allows for little to no support of common persistence management 
features such as lazy loading, identity management or change tracking.
Without such features, repositories end up maintaining their own caches, 
implementing `save` methods and multiple versions of `toDomainModel`-ish methods.

In order to completely handle the persistence of a proper domain model, only a 
[data mapper](https://en.wikipedia.org/wiki/Data_mapper_pattern) can succeed.

#### Data Mapper Pattern
This is where the single exception comes into play. The one other ORM for PHP 
that's not Active Record is the [Doctrine project](https://www.doctrine-project.org/).

Doctrine, though, leaves a lot to be desired. It has a reputation of, while 
being incredibly useful, also being slow, complex and inflexible.

In consequence, an interesting situation exists: the Doctrine project 
**sucks** - and yet it is (or was*) the **best** ORM for PHP.
Being the *only* data mapper has been at the same time the recipe for Doctrine's 
success, and it's downfall. The lack of competition has assured its position as 
number one, but also removed the need for improvement.

Doctrine has had the monopoly on the market of data mapper ORM's for around 
eight years. The *Vivaldi* project is of the opinion that monopolies should not 
exist for such extended amount of time.

*The *Vivaldi* project is not quite ready yet for production, or even anything 
that looks like it, at the time of writing this document.

### Opinions and assumptions
Like most software, *Vivaldi* is rather opinionated. It provides an abstraction 
layer for handling the persistence of domain models - focusing on the types of 
models it likes best.

*Vivaldi* prefers thin models, with decent value objects.
The entities may or may not be mutable - *Vivaldi* aims to support both styles.

In order to effectively map a domain model to its data store(s), *Vivaldi* makes 
several assumptions about the model:
- Entities only reference scalars, value objects and other entities (or 
  collections of the aforementioned) - but never services, repositories or 
  other things that don't belong in an entity
- Objects in the domain model are *userland* classes (no stdClass objects and 
  the like)
- Properties in the model are properly declared; *Vivaldi* will not take dynamic 
  properties into account when committing or rolling back units of work
- *Vivaldi* knows about the classes that make up the domain model

In terms of its own internal source code, the *Vivaldi* project keeps to the 
following guidelines and design preferences:
- Objects that don't have an urgent need for mutability, are kept immutable
- Decisions that can be moved to deploy time are not performed at runtime
- Knowledge about the actual data store remains at the very edges of the system

The *Vivaldi* organisational structure consists of several layers. On the most 
detailed level, source code is placed in methods. These methods are placed in 
classes, to be used by the individual objects.
Those classes are organised in packages, forming a more coarse grained 
organisational layer.

Each of these elements (methods, classes and packages) has a single 
responsibility. A method will generally be responsible for a particular action, 
while a class is responsible for one specific type of operation.
A package is responsible for a particular purpose. 
Consisting of several classes, which in turn consist of several methods, the 
package groups the operations into a collective structure aimed at providing one 
specific benefit.

## Contents
The document recognises several particular stages in the persistence workflow:
- [Loading the entities](#loading-the-entities) from the data store into memory
- [Detecting the changes](#change-management) that are being made to the domain
- [Saving the updates](#saving-the-updates) back to the data store

Outside of this particular flow, but important nonetheless, the document 
contains a section on the [Mapping Configuration](#mapping-the-domain).

The document concludes with a [note on testability](#note-on-testability).

Notice that many of the mentioned modules do not exist yet. Urls to the project 
pages may result in a 404 status. The modules that are not yet created at the 
time of writing this document are marked with an asterisk.

# Loading the entities

Entity access is provided by the repositories. The repository invokes a finder, 
which tries to fetch the entities. If the entities are not in the identity map, 
a loader fetches the data from the source and calls upon a deserializer to 
transform them into objects.

## Repositories
Repositories are the primary access point to access the previously stored 
entities of the domain.

Usually, repositories will implement the interfaces that are defined in the 
domain layer; as such they are often handcrafted to comply with that interface.

Other data mapping ORM implementations often let userland repository 
implementations interact with the data store through a custom query language 
like [HQL](https://docs.jboss.org/hibernate/orm/3.3/reference/en/html/queryhql.html) 
or [DQL](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/dql-doctrine-query-language.html).

Such object query languages are meant to form a bridge between object-thinking 
and sql-thinking.

### Specification
There is currently no intention to build a query language for *Vivaldi*.

The *Vivaldi* project aims to be as agnostic as possible about the source of the 
data, and does not merely aim to provide a bridge between objects and sql, but 
rather to provide a persistence mechanism for any kind of data store.

Instead, userland repositories can communicate their wishes by passing a 
[specification](https://github.com/Stratadox/Specification) object to the base 
repository.

Specifications are used as a filter for the entities: only the entities that the 
specification is satisfied by, get retrieved from the base repository.

```php
final class ArePremium extends Specification
{
    public static function customers(): self
    {
        return new self();
    }

    public function isSatisfiedBy($customer): bool
    {
        return $customer instanceof Customer
            && $customer->spentAtLeast(Money::EUR('500.00'));
    }
}
```

Such specifications are lightweight objects that can easily be passed around, 
produced by a factory or composed together:

```php
$newAndPremium = $customers->that(ArePremium::customers()->and(AreNew::customers()));
```

The advantage of using specifications rather than a sql-inspired object query 
language is that they are first and foremost meant to filter objects **in 
memory**. This means that even if the particular data store is unable to accept 
a more effective querying mechanism, we can always fall back to loading more 
data than strictly required and applying the filters on the loaded entities.

### Sorting
Together with the specification, the [base repository](#repository) accepts an 
optional [sorting definition](https://github.com/Stratadox/Sorting).

The resulting entities are sorted using this definition.

### Slicing
A [slice](https://github.com/Stratadox/Slice)* is a simple value object that 
denotes the offset to start loading at, and the amount of entities in the slice.

If [sorting](#sorting) is used to generate a sql `sort by` clause, the slices 
are used to produce the `limits`. Like most components of the *Vivaldi* project, 
these slices are also capable of slicing in-memory collections.

### Repository
The [base repository](https://github.com/Stratadox/Repository)* can be adapted 
by userland repositories to match the interfaces from the domain.

Although the base repository is final (and can therefore not be inherited from)
custom repository classes can use it by using the [adapter pattern](https://en.wikipedia.org/wiki/Adapter_pattern):

```php
final class RegisteredCustomerRepository implements AllRegisteredCustomers
{
    private $customers;

    public function __construct(Repository $customers)
    {
        $this->customers = $customers;
    }

    public function thatArePremiumCustomers(): Customers
    {
        return $this->customers->that(ArePremium::customers());
    }

    public function whoRegisteredAfter(Date $date): Customers
    {
        return $this->customers->that(RegisteredAfter::the($date), Sort::descendingBy('registrationDate'));
    }

    public function register(Customer $newCustomer): void
    {
        $this->customers->add($newCustomer);
    }
}
```

#### Flow
Each base repository gets injected with a reference to the [Unit of Work](#unit-of-work), and to 
a [finder](#finder). Such finder holds a reference to a [deserializer](#deserialization), 
which in turn references a [hydrator](#hydration).

Finders are linked together in a [chain of responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern).
This allows the finding mechanism to probe the fastest data sources first, only 
connecting to slower data sources if the quicker ones don't have the data.

For example, when loading a User from a system with a sql database and a 
key-value cache, the following might happen:

Control first flows to the in-memory finder for the Users. If the User is not 
loaded into memory yet, control is passed to the finder that looks for User 
entries in the [cache](#cache).

If the cache is hit, the finder calls upon a [loader](#loader), which uses its 
deserializer to transform the cached data into a domain entity. The deserializer 
is preconfigured to perform that specific task: transforming the cached data 
into a User entity, with its properties, value objects, proxies and eager 
relationships.

In case the finder encountered a cache miss instead, control is given to the 
next finder. This one looks for the User in the database (or, more specifically, 
calls upon a [Sql selector](#sql-selector) to query the user data) and gives 
that data to a [table loader](#table-loader), which resolves joined tables and 
[deserializes](#deserialization) the input into an object with eagerly loaded 
relationships.

>Notice that two separate deserializers are used in the example: the deserializer 
for loading a User from the cached data differs from the deserializer used in 
loading the same User from a sql source.
>
>Having a different deserializer per loading operation allows for loading data 
from multiple differing formats, without having to apply heavy logic at runtime 
to find out how to map the data.

Benefits of using chained, preconfigured finders per entity type include:
- No need for big decision trees in the loading process: the objects already 
  know exactly what to do;
- The ability to configure alternative data sources, such as a cache or an 
  online API;
- The ability to have different entities retrieved from different types of data 
  sources (eg. loading one entity from sql and another from file)

A disadvantage could be a perceived inflexibility at runtime. Once a finder is 
configured to load a particular sql statement and deserialize it into an object 
with `x` eager relationships, it cannot be altered at runtime to eagerly load `y`
relations instead.

#### Sometimes eager loading
Sometimes, eager loading is only required in some circumstances. If, for example, 
some of the customers from the repository our previous example sometimes need 
balance updates, and usage metrics tell us eager loading would be very useful in 
only that particular scenario, an alternative repository with a specific eager 
loading configuration can be used alongside the default.

```php
final class RegisteredCustomerRepository implements AllRegisteredCustomers
{
    private $customers;
    private $customersWithTransactions;

    public function __construct(Repository $customers, Repository $customersWithTransactions)
    {
        $this->customers = $customers;
        $this->customersWithTransactions = $customersWithTransactions;
    }

    public function thatArePremium(): Customers
    {
        return $this->customers->that(ArePremium::customers());
    }

    public function whoNeedBalanceUpdates(): Customers
    {
        return $this->customersWithTransactions->that(NeedBalanceUpdates::today());
    }
}
```

Alternatively, separate repositories can be used for different use cases.

#### Repository methods
The base repository exposes methods to `find` existing entities, to `add` new 
entities and to `remove` old ones.

The repository also exposes an `update` method, but this should not be confused 
with `save`. Repositories don't have a `save` method, because persisting the 
changes caused by the work that was done is the responsibility of the Unit of 
Work, not of the repository.

Instead, the `update` method exists in order to accommodate for immutable 
entities. Entities can be immutable, but in order to recognise any changes, the 
repository must be notified when the entity changes object reference.

Base repositories themselves are mostly [facades](https://en.wikipedia.org/wiki/Facade_pattern). 
Operations like add, update and remove are redirected to the Unit of Work, so 
that it can update its view on the current state.
Operations that find entities are rerouted to a [finder](#finder), which 
determines how to fetch the entity.

## Finder
[Finders](https://github.com/Stratadox/EntityFinder)* are responsible for 
figuring out the best way to fetch entities. 

If an entity is already loaded into the [identity map](#identity-management), 
the finder should get it from there.
When the entity is not in memory yet, it is looked for in the [next best 
place](#cache), or the [next best after that](#data-fetching).

This is an easy process when searching for entities by their identifiers, but it 
becomes somewhat [more complex](#bulk-finder) when entities are loaded based on 
a specification.

Unless the entity is fetched from memory, it needs to be [loaded into memory](#loader) 
in order to be useful. 

## Bulk Finder
A [bulk finder](https://github.com/Stratadox/BulkEntityFinder)* is also 
responsible for finding entities, but instead of finding by identifier like the 
regular [finders](#finder), bulk finders load all entities that match a certain 
[specification](#specification).

The bulk finder will generally convert the specification object into a [sql 
select statement](#sql-selector) or [other selection mechanism](#data-fetching), 
depending on the data store.

However, once we can be reasonably sure that the entities that would be fetched 
are already in memory, involving the data store is more expensive than simply 
using the specification object as filter on the entities that have already been 
loaded.

Such scenario would happen if, for example:
- All entities of this type are already in memory,
- The same specification was already fetched,
- The specification was fetched as part of a composed specification,

## Cache
Sometimes called "second-level cache", the cache is a fast(er) way to retrieve 
an entity.

To *Vivaldi*, it's just a non-sql [data store](#data-fetching).

## Data fetching
Like finders, [data retrievers](https://github.com/Stratadox/DataRetrieval)* 
come in two flavours: those that load single entities, and those that load by 
specification.

The packages listed below are some of the data fetching mechanisms that are 
planned.

### Redis reader
The [Redis Reader](https://github.com/Stratadox/RedisReader)* uses a [redis 
client](https://github.com/phpredis/phpredis) to fetch the data of an entity. 

### Session reader
The [Session Reader](https://github.com/Stratadox/SessionReader)* loads entity 
data from the [session](https://secure.php.net/manual/en/book.session.php).

### Graph traverser
The [graph traverser](https://github.com/Stratadox/GraphTraversal)* transforms 
the specification into a Cypher query, and loads it using a [neo4j 
client](https://github.com/graphaware/neo4j-php-client).

### SOLR querier
The [SOLR querier](https://github.com/Stratadox/SolrQuerier)* transforms the 
specification into a SOLR query.

### Sql selector
The [sql selector](https://github.com/Stratadox/SqlSelector)* transforms the 
specification into a SQL select query.

### Misc readers
Other fetchers might include:
- A REST API reader, mapped to `get` requests
- A mongodb reader
- CSV?
- Excel?
- A/R orms?
- Etc.

## Loader
A [loader](https://github.com/Stratadox/Loader)* is an abstraction for several 
packages that load the data from a data store into memory, using the [identity 
map](#identity-management) to prevent double-loading any entities that are 
already in memory.

The end result, with fully [deserialized](#deserialization) entities and an 
updated identity map, is returned to the repository, which passes it on the 
Unit of Work to update the before-state.

### Table Loader
The [table loaders](https://github.com/Stratadox/TableLoader) transform the 
result of a (potentially joined) select query into objects.

## Deserialization
[Deserialization](https://github.com/Stratadox/Deserializer) transforms the 
[fetched](#data-fetching) data into a domain entity.

In most cases, the deserializer objects will already know how to deserialize the 
data they receive. However, when inheritance is involved, the concrete class to 
instantiate depends on the data that is retrieved.

The deserialization process consists of two stages: instantiation and hydration.

### Instantiation
An [instantiator](https://github.com/Stratadox/Instantiator) produces the empty 
object. Although of the right type, it's still a blank slate where data is 
concerned.

### Hydration
A [hydrator](https://github.com/Stratadox/Hydrator) fills the object with data. 
The hydrator can be [mapped](https://github.com/Stratadox/HydrationMapping) in 
order to transform the input data to fit the object model. This may involve 
transforming scalar input data into value objects or creating 
[proxies](https://github.com/Stratadox/Proxy) for lazily loaded relationships.

# Change management
To save the changes, the changes need to be detected first. There are two ways 
of doing so: either by intercepting changes to all properties whenever they 
change, or by tracing the old and new state and computing the differences 
between them on commit. *Vivaldi* implements the second mechanism, through its 
[state management](#state-management) package.

Differentiating the extracted before- and after states has a slight disadvantage 
over intercepting modifications in terms of performance. This small performance 
drawback is acceptable due to the many benefits the mechanism has to offer.
For starters, it's arguably cleaner, given that the alternative involves the 
alteration of production code at deploy time to secretly couple it to the 
persistence mechanism without anyone noticing. This also makes it easier to 
implement and understand.
In terms of functionality, it allows for the special feature of `immutable 
entities`.
Since the state of the model is based only on the data that was found while 
extracting, it has no direct relation to the object instance itself.

This means that an entity `Foo` with id `123` and a property named `bar` with 
value `baz`, might be replaced by a new instance of `Foo`, also with id `123` 
but now with the value `qux` in its property `bar`.
If the new instance is given back to the repository with an [update](#repository-methods) 
call, the Unit of Work will simply treat it as a state change to the entity.

## Unit of Work
A [Unit of Work](https://github.com/Stratadox/UnitOfWork)* is an in-memory 
representation of a transaction. It manages the current unit of the work that 
the application is performing.

The unit of work can be committed to the database or rolled back in memory.

When committing a unit of work, the changes between the original state and the 
current state are saved to the data store. In case a unit of work is rolled back,
the in-memory object model is reverted to the original state.

Due to the intricacies of lazy-loading, the Unit of Work is one of the few 
mutable objects of the ORM, due to the following circumstances:
- The unit of work is responsible for keeping the original state of the entities, 
  so that [a state extractor](#state-management) can determine the changes that 
  were made.
- Entities may be loaded **after** changes have been made to the domain model.
- The lazily loaded relationship already existed: it should not be considered 
  "new", even though it was not in memory when the work started.

In order to accommodate these requirements, the Unit of Work must update its 
views on the original state of the domain model whenever an entity is lazily 
loaded.

### State management
The [entity state](https://github.com/Stratadox/EntityState) is a snapshot of 
the state of the domain model at a given time. A state extractor recursively 
inspects the entities and their value objects, capturing their state in a 
structure that is optimised for ease of comparison.

In extracting the state, the assumption is made that every property that points 
to an object that is not found in the [identity map](#identity-management) is a 
value object. Such objects are considered part of the state of the entity.

### Identity management
[Identity management](https://github.com/Stratadox/IdentityMap) is done by 
saving a reference to each entity in an immutable map of maps. The first level 
of indexation is done on a class level, the second based on identifier.

# Saving the updates

## Saviour
A [saviour](https://github.com/Stratadox/Saviour)* takes the list of changes and 
determines which saving mechanism to pass the individual entity changes to.
 
Multiple savers can be configured per entity, to update multiple data sources at 
the same time. This allows to, for example, write all the data to the database, 
some of it to the cache and some of it to a full text search engine.

The individual writers are preconfigured to respond to certain keys from the 
[state changes](#state-management). For instance, a [sql updater](#sql-updater) 
might respond to the reduction of a `count(array:cars)` response key by 
generating the `delete` clause needed to remove the data from the unused value 
objects, or by removing the references from a "join table" - depending on 
whether the cars are entities or value objects in the domain model.
Which strategy to follow is a decision that can in most cases be made at deploy 
time, although some runtime parsing may be required for more complex object 
structures.

### Redis writer
The [redis writer](https://github.com/Stratadox/RedisWriter)* uses a [redis 
client](https://github.com/phpredis/phpredis) to update the data of an entity. 

### Session writer
The [session writer](https://github.com/Stratadox/SessionWriter)* loads entity 
data from the [session](https://secure.php.net/manual/en/book.session.php).

### Graph updater
The [graph updater](https://github.com/Stratadox/GraphUpdater)* updates the 
graph using a [neo4j client](https://github.com/graphaware/neo4j-php-client).

### SOLR updater
The [SOLR updater](https://github.com/Stratadox/SolrUpdater)* updates the SOLR 
data source.

### Sql updater
The [sql updater](https://github.com/Stratadox/SqlUpdater)* translates the changes 
to entities into insert-, update- and delete statements.

### Misc writers
Other writers might include:
- A REST API writer, using `post` and `put`
- A mongodb writer
- CSV?
- Excel?
- A/R orms?
- Etc.

## Orphan removal
Orphaned value objects are killed off by default; for entities it is only rarely 
required.

When orphan removal *is* required for entities, it follows a slightly more 
complicated process than for value objects. As opposed to value objects, the 
entities are referenced by identifier. 

### Value object orphan removal
For value objects, we can simply copy the data when two objects reference the 
same value object. If, for example, two `Order` objects both hold a reference to 
the same `Price` value object, it should make no difference if we make them 
reference two separate value objects when we get the `Order` objects back from 
the data persistence mechanism, as long as they still have the correct values.

Due to the comparison-by-value, we can store the references to the value object 
as were they separate values altogether. Due to storing them as different values, 
we can remove the one value without removing the other value.

This inherent property of value object persistence makes it so that we can 
simply remove any value object that has been dereferenced by its owner.

Orphan removal for value objects is executed by the responsible [saviour](#saviour).

### Entity orphan removal
Entities are a completely different matter. When removing an entity from the 
data store, we don't just remove the referenced copy, but the original entity 
data itself.

Simply removing each entity once it gets dereferenced could be sufficient for 
many applications, but can be insufficient in certain cases.

Consider the following example:
- A `Contact Person` is modeled as entity that should be removed when nothing
  references it anymore.
- An `Support Team` has a number of `Contact Persons`.
- A `Customer` has one `Contact Person`.

If we would remove the `Contact Person` from the database once it gets 
disconnected from their last `Account`, it may lead the `Customer` in an invalid 
state: after all, we've just removed their `Contact Person`.

In order to prevent reaching invalid states due to a too eager removal of some 
not-actually-orphaned children, a [`refcount`](https://en.wikipedia.org/wiki/Reference_counting)
is maintained for entities that are mapped with orphan removal.

Reference sources can be white- or blacklisted, so that cyclical references can 
be excluded from the count.

The [reference counter](https://github.com/Stratadox/ReferenceCounter)* supports 
the tracking of reference counts by providing decorators for the 
[deserialization process](#deserialization) and the [saving process](#saviour).
It works by stripping the reference count from the input, keeping track of it 
privately. On the saving side, it combines the original count with the removals 
from the result. The decorated saviour receives an adapted result with the 
orphans removed and the updated reference count added.

# Mapping the domain
Yeah, that will be the hardest part. Or, at least, the hardest part to automate, 
otherwise simply the most annoying part...

It follows from *Vivaldi*'s [design decisions](#opinions-and-assumptions) that 
most of the "heavy lifting" will be done in this phase. Rather than passing a 
bunch of mapping objects around at runtime, each part of the persistence toolset 
is pre-configured when applying the mapping: during deploy.

Eventually, as much of the mapping will be automated as much as possible - but 
since the system as a whole could also work without, *Vivaldi*'s initial version 
will require people to write their queries by hand - plus setting up a whole 
bunch of [hydration mapping](#hydration) objects and possibly more such nasty 
manual labour.

# Note on testability
The system will be able to provide many benefits in terms of testability.

Having a persistence layer that is decoupled from the data source already makes
writing unit tests a lot easier, since an in-memory repository can be substituted 
in order to make testing fast.

A potential problem with this kind of testing is that the mapping itself is often 
not tested, or only at the very last moment or by hand.

One of the planned features is to auto-generate a set of integration tests for 
the entities, by creating `n` randomised entities and persisting them into a test 
environment of the data store(s).
In a new unit of work, the entities are loaded from the database and compared 
against copies of the original entities. If they differ, a problem with the 
mapping is detected.

An additional set of auto-generatable integration tests applies to the mapping 
of the [specifications](#specification).
Since these specification objects are initially built to filter objects in 
memory, they can also serve as automated tests for the mapping configuration: 
we can automatically load all the entities of a class into memory, filtered in 
memory by the specification object, and compare that collection with the results 
of the mapped fetching strategy (read: sql select query). When the results are 
not exactly the same, there is an apparent problem with the mapping.
