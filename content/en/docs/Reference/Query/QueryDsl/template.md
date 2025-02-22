---
title: "TEMPLATE Queries"
linkTitle: "TEMPLATE"
weight: 50
description: >
---

## Overview {#overview}

TEMPLATE queries are constructed using SQL templates.

The TEMPLATE query is an optional feature not included in the core module.
To use it, the following dependency declaration must be included in your Gradle build script:

```kotlin
val komapperVersion: String by project
dependencies {
    implementation("org.komapper:komapper-template:$komapperVersion")
}
```

{{< alert title="Note" >}}
The above dependency declaration is not necessary when using [Starters]({{< relref "../../Starter" >}}).
{{< /alert >}}

{{< alert title="Note" >}}
The `komapper-template` module uses reflection internally.
{{< /alert >}}

## fromTemplate

To issue a SELECT statement, call the `fromTemplate`, `bind`, and `select` functions as follows:

```kotlin
val sql = "select * from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
    .bind("street", "STREET 10")
    .select { row: Row ->
        Address(
            row.getNotNull("address_id"),
            row.getNotNull("street"),
            row.getNotNull("version")
        )
    }
```

The `fromTemplate` function accepts a string of [SQL template]({{< relref "#sql-template" >}}).

The `bind` function binds a value to a [bind variable directive]({{< relref "#sql-template-bind-variable-directive" >}}).

The `select` function converts a `Row` to any type using the given lambda expression.
`Row` is a thin wrapper around `java.sql.ResultSet` and `io.r2dbc.spi.Row`.
It has functions to retrieve values by column label or index.
Note that the index starts from 0.

### selectAsEntity

If you want to receive the result as a specific entity, call `selectAsEntity`.
For the first argument, specify the entity's metamodel.
The SELECT clause in the SQL template must include columns corresponding to all properties of the entity.

In the following example, the result is received as an `Address` entity.

```kotlin
val sql = "select address_id, street, version from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
  .bind("street", "STREET 10")
  .selectAsEntity(a)
```

By default, entities are mapped based on the order of columns in the SELECT list.
However, by passing `ProjectionType.NAME` as the second argument to `selectAsEntity`, you can map based on the column names.

```kotlin
val sql = "select street, version, address_id from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
  .bind("street", "STREET 10")
  .selectAsEntity(a, ProjectionType.NAME)
```

When you annotate the entity class you want to receive as a result with `@KomapperProjection`,
you can use a dedicated extension function to write your code more concisely as follows:

```kotlin
val sql = "select address_id, street, version from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
  .bind("street", "STREET 10")
  .selectAsAddress()
```

```kotlin
val sql = "select street, version, address_id from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
  .bind("street", "STREET 10")
  .selectAsAddress(ProjectionType.NAME)
```

### options {#select-options}

To customize the behavior of the query, call the `options` function.
The `options` function accept a lambda expression whose parameter represents default options.
Call the `copy` function on the parameter to change its properties:

```kotlin
val sql = "select * from ADDRESS where street = /*street*/'test'"
val query: Query<List<Address>> = QueryDsl.fromTemplate(sql)
    .options {
        it.copy(
            fetchSize = 100,
            queryTimeoutSeconds = 5
        )
    }
    .bind("street", "STREET 10")
    .select { row: Row ->
        Address(
            row.getNotNull("address_id"),
            row.getNotNull("street"),
            row.getNotNull("version")
        )
    }
```

The options that can be specified are as follows:

escapeSequence
: Escape sequence specified for the LIKE predicate. The default is `null` to indicate the use of Dialect values.

fetchSize
: Default is `null` to indicate that the driver value should be used.

maxRows
: Default is `null` to indicate use of the driver's value.

queryTimeoutSeconds
: Query timeout in seconds. Default is `null` to indicate that the driver value should be used.

suppressLogging
: Whether to suppress SQL log output. Default is `false`.

Properties explicitly set here will be used in preference to properties with the same name that exist
in [executionOptions]({{< relref "../../database-config/#executionoptions" >}}).

## executeTemplate

To issue a DML(Data Manipulation Language) statement, call the `executeTemplate` and `bind` functions as follows:

```kotlin
val sql = "update ADDRESS set street = /*street*/'' where address_id = /*id*/0"
val query: Query<Long> = QueryDsl.executeTemplate(sql)
    .bind("id", 15)
    .bind("street", "NY street")
```

The `executeTemplate` function accepts a string of [SQL template]({{< relref "#sql-template" >}}).

The `bind` function binds a value to a [bind variable directive]({{< relref "#sql-template-bind-variable-directive" >}}).

If a duplicate key is detected during query execution,
the `org.komapper.core.UniqueConstraintException` is thrown.

### returning {#executeTemplate-returning}

By using the `returning` function, you can execute an update DML and also retrieve the results. After executing the `returning` function, you can use the `select` and `selectAsEntity` functions mentioned in [fromTemplate]({{< relref "#fromtemplate" >}}).

```kotlin
val sql = """
    insert into address
        (address_id, street, version)
    values
        (/*id*/0, /*street*/'', /*version*/0)
    returning address_id, street, version
""".trimIndent()
val query: Query<Address> = QueryDsl.executeTemplate(sql)
    .returning()
    .bind("id", 16)
    .bind("street", "NY street")
    .bind("version", 1)
    .select { row: Row ->
        Address(
            row.getNotNull("address_id"),
            row.getNotNull("street"),
            row.getNotNull("version")
        )
    }
    .single()
```

{{< alert title="Note" >}}
The above SQL template uses the RETURNING clause, which is supported by PostgreSQL and other databases. Please note that the SQL for returning results from update DML varies depending on the DBMS.
{{< /alert >}}

### options {#execute-options}

To customize the behavior of the query, call the `options` function.
The `options` function accept a lambda expression whose parameter represents default options.
Call the `copy` function on the parameter to change its properties:

```kotlin
val sql = "update ADDRESS set street = /*street*/'' where address_id = /*id*/0"
val query: Query<Long> = QueryDsl.executeTemplate(sql)
    .bind("id", 15)
    .bind("street", "NY street")
    .options {
        it.copy(
            queryTimeoutSeconds = 5
        )
    }
```

The options that can be specified are as follows:

escapeSequence
: Escape sequence specified for the LIKE predicate. The default is `null` to indicate the use of Dialect values.

queryTimeoutSeconds
: Query timeout in seconds. Default is `null` to indicate that the driver value should be used.

suppressLogging
: Whether to suppress SQL log output. Default is `false`.

Properties explicitly set here will be used in preference to properties with the same name that exist
in [executionOptions]({{< relref "../../database-config/#executionoptions" >}}).

## Commands {#command}

A command is a feature that treats an SQL template and parameters as a single unit. You define an SQL template by annotating a class with the `org.komapper.annotation.KomapperCommand` annotation, and define the parameters in the class properties. How the SQL result is handled is expressed by inheriting from a specific class.

Below is an example of a command that retrieves multiple records.

```kotlin
@KomapperCommand("""
    select * from ADDRESS where street = /*street*/'test'
""")
class ListAddresses(val street: String): Many<Address>({ selectAsAddress() })
```

When you define a command and run the build, an extension function named `execute` is generated in the `QueryDsl`. Therefore, the above command can be executed as follows:

```kotlin
val query: Query<List<Address>> = QueryDsl.execute(ListAddresses("STREET 10"))
```

Commands have the advantage of enabling SQL template validation at compile time compared to using methods like [fromTemplate]({{< relref "#fromtemplate" >}}) or [executeTemplate]({{< relref "#executetemplate" >}}). Specifically, the following can be achieved:

- Detect syntax errors in SQL templates. For example, if `/*%end*/` is missing, a compile-time error occurs.
- Detect unused parameters. If unused parameters are found, a warning message is output. You can suppress this warning by annotating the parameter with `org.komapper.annotation.KomapperUnused`.
- Validate the types and members of parameters. For instance, if `name` is a parameter of type `String`, attempting to access a non-existent member in the SQL template like `/* name.unknown */` will result in a compile-time error.

{{< alert title="Note" >}}
A command class can be defined as a top-level class, a nested class, or an inner class, but not as a local class.
{{< /alert >}}

There are five types of commands:

- One
- Many
- Exec
- ExecReturnOne
- ExecReturnMany

### One {#command-one}

A command that retrieves a single record inherits from `org.komapper.core.One`.

```kotlin
@KomapperCommand("""
    select * from ADDRESS where address_id = /*id*/0
""")
class GetAddressById(val id: Int): One<Address?>({ selectAsAddress().singleOrNull() })
```

The type parameter of `One` specifies the type of the value to be retrieved. The constructor of `One` takes a lambda expression to process the search result. What can be done within the lambda is the same as the `select` and `selectAsEntity` functions mentioned in [fromTemplate]({{< relref "#fromtemplate" >}}).

The above example assumes that the `Address` class is annotated with `@KomapperProjection`. Therefore, the `selectAsAddress` function is used to convert the result to the `Address` class.

### Many {#command-many}

A command that retrieves multiple records inherits from `org.komapper.core.Many`.

```kotlin
@KomapperCommand("""
    select * from ADDRESS where street = /*street*/'test'
""")
class ListAddresses(val street: String): Many<Address>({ selectAsAddress() })
```

The type parameter of `Many` specifies the type representing a single record to be retrieved. The constructor of `Many` takes a lambda expression to process the search result. What can be done within the lambda is the same as the `select` and `selectAsEntity` functions mentioned in [fromTemplate]({{< relref "#fromtemplate" >}}).

The above example assumes that the `Address` class is annotated with `@KomapperProjection`. Therefore, the `selectAsAddress` function is used to convert the result to the `Address` class.

### Exec {#command-exec}

A command that executes an update DML inherits from `org.komapper.core.Exec`.

```kotlin
@KomapperCommand("""
    update ADDRESS set street = /*street*/'' where address_id = /*id*/0
""")
class UpdateAddress(val id: Int, val street: String): Exec()
```

### ExecReturnOne {#command-exec-return-one}

A command that executes an update DML and returns a single record inherits from `org.komapper.core.ExecReturnOne`.

```kotlin
@KomapperCommand("""
    insert into ADDRESS
        (address_id, street, version)
    values
        (/* id */0, /* street */'', /* version */1)
    returning address_id, street, version
""")
class InsertAddressThenReturn(val id: Int, val street: String): ExecReturnOne<Address>({ selectAsAddress().single() })
```

The type parameter and constructor of `ExecReturnOne` are similar to those of [One]({{< relref "#command-one" >}}).

### ExecReturnMany {#command-exec-return-many}

A command that executes an update DML and returns multiple records inherits from `org.komapper.core.ExecReturnMany`.

```kotlin
@KomapperCommand("""
    update ADDRESS set street = /*street*/'' returning address_id, street, version
""")
class UpdateAddressThenReturn(val id: Int, val street: String): ExecReturnMany<Address>({ selectAsAddress() })
```

The type parameter and constructor of `ExecReturnMany` are similar to those of [Many]({{< relref "#command-many" >}}).

### Partial {#command-partial}

Partial is a feature for reusing parts of an SQL template.
A partial is defined as a top-level `const` with the `org.komapper.annotation.KomapperPartial` annotation.

```kotlin
@KomapperPartial(
    """
    /*%if pagination != null */
    limit /* pagination.limit */0 offset /*pagination.offset*/0
    /*%end*/
    """,
)
private const val paginationPartial = ""
```

To use a partial, define the command in the same file where the partial is defined, and reference the partial in the SQL template using the `/*> partialName */` notation.

```kotlin
@KomapperCommand(
    """
    select * from address order by address_id
    /*> paginationPartial */
    """,
)
class UsePartial(val pagination: Pagination?) : Many<Address>({ selectAsAddress() })

class Pagination(val limit: Int, val offset: Int)
```

{{< alert title="Note" >}}
One partial cannot reference another partial.
{{< /alert >}}

## SQL templates  {#sql-template}

In SQL template, directives such as bind variables and conditional branches are expressed as SQL comments.
Therefore, you can paste a string from the SQL template into a tool
such as [pgAdmin](https://www.pgadmin.org/) to execute it.

For example, an SQL template containing a conditional branch and a bind variable is written as follows:

```sql
select name, age from person where
/*%if name != null*/
  name = /*name*/'test'
/*%end*/
order by name
```

In the above SQL template, if `name != null` is true, the following SQL is generated:

```sql
select name, age from person where name = ? order by name
```

Conversely, if `name != null` is false, the following SQL is generated:

```sql
select name, age from person order by name
```

{{< alert title="Note" >}}
In the example above, did you notice that if `name != null` is false, 
the generated SQL does not contain a WHERE clause?

The SQL template does not output WHERE, HAVING, GROUP BY, or ORDER BY clauses
if they do not have any SQL identifiers or keywords.

Thus, there is no need to always include `1 = 1` in the WHERE clause 
to prevent incorrect SQL from being generated:

```kotlin
select name, age from person where 1 = 1  // unnecessary
/*%if name != null*/
  and name = /*name*/'test'
/*%end*/
order by name
```
{{< /alert >}}

### Bind variable directives  {#sql-template-bind-variable-directive}

To represent bind variables, use bind variable directives.

Bind variable directives are simple SQL comments enclosed in `/*` and `*/`.
They require test data immediately after the directive.

In the following example, `/*name*/` is the bind variable directive, 
and the following `'test'` is the test data:

```sql
where name = /*name*/'test'
```

Test data exists only to preserve correct SQL syntax. It is not used by the application.
In the process of parsing the template, test data is removed and bind variables are resolved.
Finally, the above template is converted to SQL as follows:

```sql
where name = ?
```

To bind a value to an IN clause, the bound value must be `kotlin.collections.Iterable`.
In the following example, `names` is `Iterable<String>`, and the following `('a', 'b')` is the test data:

```sql
where name in /*names*/('a', 'b')
```

To bind a `Pair` value to an IN clause, the bound value must be `kotlin.collections.Iterable<Pair>`
In the following example, `pairs` is `Iterable<Pair<String, String>>`, 
and the following `(('a', 'b'), ('c', 'd'))` is the test data:

```sql
where (name, age) in /*pairs*/(('a', 'b'), ('c', 'd'))
```

### Literal variable directives {#sql-template-literal-variable-directive}

To represent literals, use literal variable directives.

Literal variable directives are SQL comments enclosed in `/*^` and `*/`.
They require test data immediately after the directive.

In the following example, `/*^myLiteral*/` is the literal variable directive,
and the following `'test'` is the test data:

```sql
where name = /*^myLiteral*/'test'
```

Test data exists only to preserve correct SQL syntax. It is not used by the application.
In the process of parsing the template, test data is removed and literal variables are resolved.
Finally, the above template is converted to SQL as follows:

```sql
where name = 'abc'
```

### Embedded variable directives {#sql-template-embedded-variable-directive}

To embed sql fragments, use embedded variable directives.

Embedded variable directives are SQL comments enclosed in `/*#` and `*/`.
Unlike other variable directives, they do not require test data immediately after the directive.

In the following example, `/*# orderBy */` is the embedded variable directive:

```sql
select name, age from person where age > 1 /*# orderBy */
```

In the example above, if the `orderBy` expression evaluates to `order by name`, 
the template is converted to the following SQL:

```sql
select name, age from person where age > 1 order by name
```

### if directives {#sql-template-if-directive}

To start conditional branching, use if directives.

If directives are SQL comments enclosed in `/*%if` and `*/`.

A conditional branch must begin with an if directive and 
end with an [end directive]({{< relref "#sql-template-end-directive" >}}).

In the following example, `/*%if name != null*/` is the if directive:

```kotlin
/*%if name != null*/
  name = /*name*/'test'
/*%end*/
```

You can also put an else directive between an if directive and an end directive:

```kotlin
/*%if name != null*/
  name = /*name*/'test'
/*%else*/
  name is null
/*%end*/
```

### for directives {#sql-template-for-directive}

To start loop processing, use for directives.

For directives are SQL comments enclosed in `/*%for` and `*/`.

A loop process must begin with a for directive and
end with an [end directive]({{< relref "#sql-template-end-directive" >}}).

In the following example, `/*%for name in names */` is the for directive:

```sql
/*%for name in names */
employee_name like /* name */'hoge'
  /*%if name_has_next */
/*# "or" */
  /*%end */
/*%end*/
```

In the `/*%for name in names */` directive, the `names` express an `Iterable` object 
and the `name` is an identifier for each element of the `Iterable` object.

Between the for and end directives, a special variable is available.
The special variable returns a boolean value indicating whether the next iteration should be executed.
The name of the special variable is a concatenation of the identifier and `_has_next`.
In the above example, the name of the special variable is `name_has_next`.

### end directives {#sql-template-end-directive}

To end conditional branching and loop processing, use end directives.

End directives are SQL comments expressed as `/*%end*/`.

### Parser-level comment directives {#sql-template-parser-level-comment-directive}

Using a parser-level comment directive allows you to include comments in an SQL template 
that will be removed after the template is parsed.

To express parser-level comments, you can use the syntax `/*%! comment */`.

Suppose you have the following SQL template:

```sql
select
  name
from
  employee
where /*%! This comment will be removed */
  employee_id = /* employeeId */99
```

The above SQL template is parsed into the following SQL:

```sql
select
  name
from 
  employee
where
  employee_id = ?
```

### Expressions {#sql-template-expression}

Expressions in the directives can perform the following:

- Execution of operators
- Property access
- Function call
- Class reference
- Use of extension properties and functions

#### Operators {#sql-template-expression-operator}

The following operators are supported. Semantics are the same as for operators in Kotlin:

- `==`
- `!=`
- `>=`
- `<=`
- `>`
- `<`
- `!`
- `&&`
- `||`

These can be used as follows:

```kotlin
/*%if name != null && name.length > 0 */
  name = /*name*/'test'
/*%else*/
  name is null
/*%end*/
```

#### Property accesses {#sql-template-expression-property-access}

To access properties, use `.` or `?.` as follows:

```kotlin
/*%if person?.name != null */
  name = /*person?.name*/'test'
/*%else*/
  name is null
/*%end*/
```

`?.` is equivalent to the safe call operator of Kotlin.


#### Function calls {#sql-template-expression-function-invocation}

Functions can be called as follows:

```kotlin
/*%if isValid(name) */
  name = /*name*/'test'
/*%else*/
  name is null
/*%end*/
```

#### Class references {#sql-template-expression-class-reference}

You can refer to a class by using the notation `@fully qualified name of the class@`.

For example, if the `example.Direction` enum class has an element named `WEST`, 
it can be referenced as follows:

```kotlin
/*%if direction == @example.Direction@.WEST */
  direction = 'west'
/*%end*/
```

#### Extension properties and functions {#sql-template-expression-extensions}

The following extension properties and functions provided by Kotlin are available by default:

- `val CharSequence.lastIndex: Int`
- `fun CharSequence.isBlank(): Boolean`
- `fun CharSequence.isNotBlank(): Boolean`
- `fun CharSequence.isNullOrBlank(): Boolean`
- `fun CharSequence.isEmpty(): Boolean`
- `fun CharSequence.isNotEmpty(): Boolean`
- `fun CharSequence.isNullOrEmpty(): Boolean`
- `fun CharSequence.any(): Boolean`
- `fun CharSequence.none(): Boolean`

```kotlin
/*%if name.isNotBlank() */
  name = /*name*/'test'
/*%else*/
  name is null
/*%end*/
```

The following extension functions defined by Komapper are also available:

- `fun String?.asPrefix(): String?`
- `fun String?.asInfix(): String?`
- `fun String?.asSuffix(): String?`
- `fun String?.escape(): String?`

For example, if you call the `asPrefix` function, 
the string `"hello"` becomes `"hello%"` and can be used in a prefix search:

```kotlin
where name like /*name.asPrefix()*/
```

Similarly, calling the `asInfix` function converts it to a string for an infix search, 
and calling the `asSuffix` function converts it to a string for a suffix search.

The `escape` function escapes special characters.
For example, it converts a string `"he%llo_"` into a string like `"he\%llo\_"`.

{{< alert title="Note" >}}
The `asPrefix`, `asInfix`, and `asSuffix` functions perform escaping internally,
so there is no need to call the `escape` function in addition.
{{< /alert >}}
