---
title: "Entity Classes"
weight: 20
description: >
---

## Overview {#overview}

In Komapper, the Kotlin classes corresponding to database tables are called entity classes.

Mapping definitions using annotations are necessary to map entity classes to tables.

[KSP](https://github.com/google/ksp) parses the mapping definitions at compile-time and generates metamodels.

The metamodels are used in the construction and execution of queries.

## Entity class definitions {#entity-class-definition}

Entity classes must meet the following requirements:

- Be a Data Class
- Visibility is not private
- No type parameter

For example, suppose we have the following table definition:

```sql
create table if not exists ADDRESS (
  ADDRESS_ID integer not null auto_increment,
  STREET varchar(500) not null,
  VERSION integer not null,
  CREATED_AT timestamp,
  UPDATED_AT timestamp,
  constraint pk_ADDRESS primary key(ADDRESS_ID)
);
```

The entity class definition corresponding to the above table definition is as follows:

```kotlin
data class Address(
  val id: Int = 0,
  val street: String,
  val version: Int = 0,
  val createdAt: LocalDateTime? = null,
  val updatedAt: LocalDateTime? = null,
)
```

See [Data Types]({{< relref "data-type.md" >}}) for type mapping between properties and columns.

## Mapping definitions {#mapping-definition}

There are two ways to create a mapping definition:

1. Entity class itself has its own mapping definition (self mapping)
2. A separate class from the entity class has the mapping definition (separation mapping)

Only one way can be applied to the same entity class.

### Self mapping {#self-mapping-definition}

The entity class must satisfy the following conditions
in addition to the requirements described in the previous section:

- Annotated by `@KomapperEntity`

For example, applying self mapping to the `Address` class shown in the previous section 
would result in the following:

```kotlin
@KomapperEntity
data class Address(
  @KomapperId
  @KomapperAutoIncrement
  @KomapperColumn(name = "ADDRESS_ID")
  val id: Int = 0,
  val street: String,
  @KomapperVersion
  val version: Int = 0,
  @KomapperCreatedAt
  val createdAt: LocalDateTime? = null,
  @KomapperUpdatedAt
  val updatedAt: LocalDateTime? = null,
)
```

### Separation mapping {#separation-mapping-definition}

Mapping classes must meet the following requirements:

- Be a Data Class
- Visibility is not private
- No type parameter
- Annotated with `@KomapperEntityDef` and accepts entity class as argument
- No properties with names different from those defined in the entity class

For example, applying separation mapping to the `Address` class shown in the previous section
would result in the following:

```kotlin
@KomapperEntityDef(Address::class)
data class AddressDef(
  @KomapperId
  @KomapperAutoIncrement
  @KomapperColumn(name = "ADDRESS_ID")
  val id: Nothing,
  @KomapperVersion
  val version: Nothing,
  @KomapperCreatedAt
  val createdAt: Nothing,
  @KomapperUpdatedAt
  val updatedAt: Nothing,
)
```

For properties that do not appear in the separation mapping, the default mapping definitions apply.
In the above example, the street property that appears in the entity class is mapped to the STREET column
even though it does not appear in the separation mapping.

There are no restrictions on the types of properties in the separation mapping. 
We use `Nothing` in the above example.

## Metamodels {#metamodel}

From a mapping definition, a metamodel is generated 
in the form of an implementation of the `org.komapper.core.dsl.metamodel.EntityMetamodel` interface.

The generated metamodel instance will be an extension property of the `org.komapper.core.dsl.Meta` object.
Applications can use this property to construct queries.

```kotlin
// get a generated metamodel
val a = Meta.address

// define a query
val query = QueryDsl.from(a).where { a.street eq "STREET 101" }.orderBy(a.id)
```

### aliases {#metamodel-aliases}

In the above example, the name of the extended property is `address`.
However, this can be changed with the `@KomapperEntity` or `@KomapperEntityDef` aliases property.

```kotlin
@KomapperEntity(aliases = ["addr"])
data class Address(
  ...
)
```

Multiple names can be specified for the `aliases` property.
In this case, each name is exposed as a different instance.
The primary use cases that require multiple different instances are self-joins and sub-queries.

```kotlin
@KomapperEntity(aliases = ["employee", "manager"])
data class Employee(
  ...
)
```

For example, to get a list of managers, create the following query using the above definition:

```kotlin
val e = Meta.employee
val m = Meta.manager
val query: Query<List<Employee>> = QueryDsl.from(m)
  .distinct()
  .innerJoin(e) {
    m.employeeId eq e.managerId
  }
```

Even if you do not define a metamodel with a name in advance, 
you can use the `clone` function to achieve the same thing:

```kotlin
val e = Meta.employee
val m = e.clone()
val query: Query<List<Employee>> = QueryDsl.from(m)
  .distinct()
  .innerJoin(e) {
    m.employeeId eq e.managerId
  }
```

### clone {#metamodel-clone}

The `clone` function can be used to generate another metamodel based on an existing metamodel.
The primary use case is to copy data to a table with the same data structure but different names.

```kotlin
val a = Meta.address
val archive = a.clone(table = "ADDRESS_ARCHIVE")
val query = QueryDsl.insert(archive).select {
  QueryDsl.from(a).where { a.id between 1..5 }
}
```

If you want to expose the cloned metamodel like any other metamodel,
use the object to hold the instance and define the extension properties of the `Meta` object.

```kotlin
object MetamodelHolder {
  private val _addressArchive = Meta.address.clone(table = "ADDRESS_ARCHIVE")
  val Meta.addressArchive get() = _addressArchive
}
```

### define {#metamodel-define}

A default WHERE clause can be defined for a metamodel using the `define` function.
This is useful when you want to always use the same search criteria when using a particular metamodel.

```kotlin
object MetamodelHolder {
  private val _bostonOnly = Meta.department.define { d ->
    where {
      d.location eq "BOSTON"
    }
  }
  val Meta.bostonOnly get() = _bostonOnly
}
```

The `bostonOnly` metamodel above generates SQL with a WHERE clause, 
even though no search criteria are specified in the query.

```kotlin
val d = Meta.bostonOnly
val query = QueryDsl.from(d)
/*
select t0_.DEPARTMENT_ID, t0_.DEPARTMENT_NO, t0_.DEPARTMENT_NAME, t0_.LOCATION, t0_.VERSION from DEPARTMENT as t0_ where t0_.LOCATION = ?
*/
```

If the query has a WHERE clause, the search criteria are concatenated with the AND predicate.

```kotlin
val d = Meta.bostonOnly
val query = QueryDsl.from(d).where { d.departmentNo greaterEq 0 }
/*
select t0_.DEPARTMENT_ID, t0_.DEPARTMENT_NO, t0_.DEPARTMENT_NAME, t0_.LOCATION, t0_.VERSION from DEPARTMENT as t0_ where t0_.LOCATION = ? and t0_.DEPARTMENT_NO >= ?
*/
```

This feature is valid even if the defined metamodel is the target of the join.

```kotlin
val e = Meta.employee
val d = Meta.bostonOnly
val query = QueryDsl.from(e).innerJoin(d) {
  e.departmentId eq d.departmentId
}
/*
select t0_.EMPLOYEE_ID, t0_.EMPLOYEE_NO, t0_.EMPLOYEE_NAME, t0_.MANAGER_ID, t0_.HIREDATE, t0_.SALARY, t0_.DEPARTMENT_ID, t0_.ADDRESS_ID, t0_.VERSION from EMPLOYEE as t0_ inner join DEPARTMENT as t1_ on (t0_.DEPARTMENT_ID = t1_.DEPARTMENT_ID) where t1_.LOCATION = ?
*/
```

It is valid not only for SELECT statements but also for UPDATE and DELETE statements.

```kotlin
val d = Meta.bostonOnly
val query = QueryDsl.delete(d).all()
/*
delete from DEPARTMENT as t0_ where t0_.LOCATION = ?
*/
```

If you want to pass parameters to the default WHERE clause, you can define it as an extension function.
Note, however, that the metamodel will be a different instance each time.

```kotlin
object MetamodelHolder {
    fun Meta.locationSpecificDepartment(value: String) = Meta.department.define { d ->
        where {
            d.location eq value
        }
    }
}
```

Here is an example of calling the above extension function.

```kotlin
val d = Meta.locationSpecificDepartment("NEW YORK")
val query = QueryDsl.from(d)
val list = db.runQuery { query }
```

## List of annotations for classes {#annotation-list-for-class}

All annotations described here belong to the `org.komapper.annotation` package.

### @KomapperEntity

Indicates that the entity class has a mapping definition.
It has an [aliases]({{< relref "#metamodel-aliases" >}}) property.

```kotlin
@KomapperEntity(aliases = ["addr"])
data class Address(
  ...
)
```

### @KomapperEntityDef

Indicates that the class is a mapping definition. 
You can specify the entity and [aliases]({{< relref "#metamodel-aliases" >}}) properties.

```kotlin
@KomapperEntityDef(entity = Address::class, aliases = ["addr"])
data class AddressDef(
  ...
)
```

### @KomapperTable

Explicitly specifies the table name.

```kotlin
@KomapperEntityDef(Address::class)
@KomapperTable("ADDRESS", schema = "ACCOUNT", alwaysQuote = true)
data class AddressDef(
  ...
)
```

The `catalog` and `schema` properties indicates the name of the catalog or schema to which the table belongs.

If the `alwaysQuote` property is set to `true`, the identifier in the generated SQL will be quoted.

If the table name is not specified in this annotation, 
the name will be resolved according to the `komapper.namingStrategy` option in the annotation process.
See also [Options]({{< relref "annotation-processing#options" >}}).

## List of annotations for properties {#annotation-list-for-property}

All annotations described here belong to the `org.komapper.annotation` package.

### @KomapperId

Indicates that it is a primary key.
To represent a composite primary key, you may specify this annotation more than one in a single entity class.

### @KomapperSequence

Indicates that the primary key is generated by a database sequence.
Must always be given with `@KomapperId`.

The type of the property to which this annotation is given must be one of the following:

- Int
- Long
- UInt
- Value class with a property of one of the above types

```kotlin
@KomapperId
@KomapperSequence(name = "ADDRESS_SEQ", startWith = 1, incrementBy = 100)
val id: Int
```

The `name` property must be the name of the sequence.
You can also specify a catalog in the `catalog` property and a schema in the `schema` property.

The values of the `startWith` and `incrementBy` properties must match the database sequence definition.

If the `alwaysQuote` property is set to `true`, the identifier in the generated SQL will be quoted.

### @KomapperAutoIncrement

Indicates that the primary key is generated by the auto-increment column of the database.
Must always be given with `@KomapperId`.

The type of the property to which this annotation is given must be one of the following:

- Int
- Long
- UInt
- Value class with a property of one of the above types

### @KomapperVersion

Indicates that this is the version number used for optimistic locking.

When this annotation is specified, optimistic locking is performed for 
[UPDATE]({{< relref "query/QueryDsl/update" >}}) and 
[DELETE]({{< relref "query/QueryDsl/delete" >}}) operations.

The type of the property to which this annotation is given must be one of the following:

- Int
- Long
- UInt
- Value class with a property of one of the above types

### @KomapperCreatedAt

Indicates the timestamp at the time of insertion.

If this annotation is given, the timestamp is set to the property in 
the [INSERT]({{< relref "query/QueryDsl/insert" >}}) process.

The type of the property to which this annotation is given must be one of the following:

- java.time.LocalDateTime
- java.time.OffsetDateTime
- Value class with a property of one of the above types

### @KomapperUpdatedAt

Indicates that this is the timestamp at the time of update.

If this annotation is given, the timestamp is set to the property in the 
[INSERT]({{< relref "query/QueryDsl/insert" >}}) and 
[UPDATE]({{< relref "query/QueryDsl/update" >}}) process.

The type of the property to which this annotation is given must be one of the following:

- java.time.LocalDateTime
- java.time.OffsetDateTime
- Value class with a property of one of the above types

### @KomapperEnum

Explicitly specifies how to map the enum property to the column.

```kotlin
@KomapperEnum(EnumType.ORDINAL)
val color: Nothing // Assume that this color property corresponds to the Color enum class
```

The `type` property of `@KomapperEnum` can be one of the following:

EnumType.NAME
: map the `name` property of `Enum` class to the string type column.

EnumType.ORDINAL
: map the `ordinal` property of `Enum` class to the integer type column.

If `@KomapperEnum` is not specified, the mapping method is resolved 
according to the `komapper.enumStrategy` option in the annotation processing.

See also [Options]({{< relref "annotation-processing#options" >}}).

### @KomapperColumn

Explicitly specifies the name of the column to be mapped to the property.

```kotlin
@KomapperColumn(name = "ADDRESS_ID", alwaysQuote = true, masking = true)
val id: Nothing
```

If the `alwaysQuote` property is set to `true`, the identifier in the generated SQL will be quoted.

If the `masking` property is set to `true`, the corresponding data will be masked in the log.

If the column name is not specified in this annotation,
the name will be resolved according to the `komapper.namingStrategy` option in the annotation process.

See also [Options]({{< relref "annotation-processing#options" >}}).

### @KomapperIgnore

Indicates that it is not subject to mapping.

### @KomapperEmbeddedId

Indicates an [Embedded Value](https://www.martinfowler.com/eaaCatalog/embeddedValue.html)
for a composite primary key.

```kotlin
data class EmoloyeeId(val id1: Int, val id2: String)

@KomapperEntity
data Employee(@KomapperEmbeddedId val id: EmoloyeeId, val: name: String)
```

### @KomapperEmbedded

Indicates an [Embedded Value](https://www.martinfowler.com/eaaCatalog/embeddedValue.html).

```kotlin
data class Money(val amount: BigDecimal, val currency: String)

@KomapperEntity
data Employee(@KomapperId val id: Int, @KomapperEmbedded val: salary: Money)
```

### @KomapperEnumOverride

Applies [`@KomapperEnum`]({{< relref "#komapperenum" >}}) to an Enum property in an
[Embedded Value](https://www.martinfowler.com/eaaCatalog/embeddedValue.html).

```kotlin
enum class Currency { JPY, USD }

data class Money(val amount: BigDecimal, val currency: Currency)

@KomapperEntity
data Employee( 
  @KomapperId
  val id: Int,
  @KomapperEmbedded
  @KomapperEnumOverride("currency", KomapperEnum(EnumType.ORDINAL))
  val: salary: Money
)
```

### @KomapperColumnOverride

Applies [`@KomapperColumn`]({{< relref "#komappercolumn" >}}) to a property in an
[Embedded Value](https://www.martinfowler.com/eaaCatalog/embeddedValue.html).

```kotlin
data class Money(val amount: BigDecimal, val currency: String)

@KomapperEntity
data Employee( 
  @KomapperId
  val id: Int,
  @KomapperEmbedded
  @KomapperColumnOverride("amount", KomapperColumn("SALARY_AMOUNT"))
  @KomapperColumnOverride("currency", KomapperColumn("SALARY_CURRENCY"))
  val: salary: Money
)
