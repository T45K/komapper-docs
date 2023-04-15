---
title: "Databases"
weight: 10
description: >
---

## Overview {#overview}

To access a database with Komapper, an instance of `JdbcDatabase` or `R2dbcDatabase` is required.
Here, these are collectively referred to as Database instances.

The Database instances are responsible for transaction control and query execution.

## Instantiation {#instantiation}

The method of creating a Database instance differs when using JDBC or R2DBC.

### When using JDBC {#instantiation-for-jdbc}

To create a Database instance from a URL, write as follows:

```kotlin
val db: JdbcDatabase = JdbcDatabase("jdbc:h2:mem:example;DB_CLOSE_DELAY=-1")
```

To specify a username and password in addition to the URL, write as follows:

```kotlin
val db: JdbcDatabase = JdbcDatabase(
  url = "jdbc:h2:mem:example;DB_CLOSE_DELAY=-1", 
  user = "sa", 
  password = ""
)
```

`javax.sql.DataSource` can also be specified.
However, in that case, you must also specify dialect.

```kotlin
val dataSource: DataSource = ...
val db: JdbcDatabase = JdbcDatabase(
  dataSource = dataSource, 
  dialect = H2JdbcDialect()
)
```

See also [Dialects]({{< relref "dialect.md" >}}).

### When using R2DBC {#instantiation-for-r2dbc}

To create a Database instance from a URL, write as follows:

```kotlin
val db: R2dbcDatabase = R2dbcDatabase("r2dbc:h2:mem:///example;DB_CLOSE_DELAY=-1")
```

To create a Database instance from `io.r2dbc.spi.ConnectionFactoryOptions` write as follows:

```kotlin
val options = ConnectionFactoryOptions.builder()
  .option(ConnectionFactoryOptions.DRIVER, "h2")
  .option(ConnectionFactoryOptions.PROTOCOL, "mem")
  .option(ConnectionFactoryOptions.DATABASE, "example")
  .option(Option.valueOf("DB_CLOSE_DELAY"), "-1")
  .build()
val db: R2dbcDatabase = R2dbcDatabase(options)
```

The `options` must contain a value whose key is `ConnectionFactoryOptions.DRIVER`.

You can also specify io.r2dbc.spi.ConnectionFactory.
However, in that case, you must also specify dialect.

```kotlin
val connectionFactory: ConnectionFactory = ...
val db: R2dbcDatabase = R2dbcDatabase(
  connectionFactory = connectionFactory, 
  dialect = H2R2dbcDialect()
)
```

See also [Dialect]({{< relref "Dialect" >}}).

## Usage {#usage}

### Transaction Control {#transaction-control}

Transactions are controlled by the `withTransaction` function of the Database instance. 
The transactional logic is passed to the `withTransaction` function as a lambda expression.

```kotlin
db.withTransaction {
    ...
}
```

See [Transaction]({{< relref "transaction.md" >}}) for details.

### Query Execution {#query-execution}

Queries are executed by calling the `runQuery` function of the Database instance.

```kotlin
val a = Meta.address
val query: Query<List<Address>> = QueryDsl.from(a)
val result: List<Address> = db.runQuery(query)
```

When the Database instance is `R2dbcDatabase` and the query type is `org.komapper.core.dsl.query.FlowQuery`, 
the `flowQuery` function can be executed.

```kotlin
val a = Meta.address
val query: FlowQuery<Address> = QueryDsl.from(a)
val flow: Flow<Address> = db.flowQuery(query)
```

Database access is made only when the `flow` instance is collected.

See [Queries]({{< relref "Query" >}}) for information on building queries.

### Low-level API execution {#low-level-api-execution}

If the [Query DSL]({{< relref "Query/QueryDsl" >}}) API does not meet your requirements,
the Low-level APIs are available.

To use the JDBC API directly, call `db.config.session.useConnection()` to get `java.sql.Connection`.

```kotlin
val db: JdbcDatabase = ...
db.config.session.useConnection { con: java.sql.Connection ->
    val sql = "select employee_name from employee where employee_id = ?"
    con.prepareStatement(sql).use { ps ->
        ps.setInt(1,10)
        ps.executeQuery().use { rs ->
            if (rs.next()) {
                println(rs.getString(1))
            }
        }
    }
}
```

Similarly, to use R2DBC API directly, call `db.config.session.useConnection()` to get `io.r2dbc.spi.Connection`.

```kotlin
val db: R2dbcDatabase = ...
db.config.session.useConnection { con: io.r2dbc.spi.Connection ->
    val sql = "select employee_name from employee where employee_id = ?"
    val statement = con.createStatement(sql)
    statement.bind(0, 10)
    statement.execute().collect { result ->
        result.map { row -> row.get(0, String::class.java) }.collect {
            println(it)
        }
    }
}
```
