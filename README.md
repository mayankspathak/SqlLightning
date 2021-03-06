SqlLightning
============

SUMMARY
-------

SqlLightning is a small, lightweight SQL data access layer designed to
supplement existing ORM technology. I wrote this tiny library for use in my own
professional and personal projects with the goal of building something that's
fast, easy to use, and isn't overly complicated or hard to maintain. Simply
implement a database access layer using stored procedures, instantiate a
database context, and fire away. The source code is extensible, well documented,
and easy to read. Skip to the examples to see a sample of what you can do.

WHAT IS IT?
-----------

SqlLightning exists to supplement existing object relational mapping systems
like nHibernate or Entity Framework. It is NOT designed to be a replacement for
such technologies, although I suspect one could write smaller projects entirely
using only SqlLightning if the need arose. SqlLightning allows the developer to
design and implement a data access layer forcing best practices set forth by
Microsoft. All calls to the database are required to be:

1. Implemented as a stored procedure
2. Parameterized
3. Use DBSafe types.

All calls to the database in question are wrapped in a transaction and rolled
back in the event of an exception. The base object is also disposable, and
should be wrapped in a ```using``` block where necessary.

Currently SqlLightning only supports a SQL server context--mainly because I
haven't had the need to interface with other RDBMS systems. However,
extensibility is clearly possible--check out the source for more information on
that.

WHY USE IT?
-----------

Object relational mapping systems are required to be generic and flexible to
support the taxing task of mapping a relational model to an object-oriented one.
Often times some issues fall out of extremely flexible designs which include
performance. SqlLightning enables developers to quickly and securely go
directly to the database when the need arises.

A glaring example of this is the way the current iteration of Microsoft's
Entity Framework handles insertion on large numbers of objects. Currently,
the query generator will create one SQL statement for every object added
or updated in a given list--which can result in thousands, or tens of thousands
of calls to a database in large, data-oriented applications. To be frank, EF
chokes in this case. An easy way to fix this is to go directly to the database
with a SqlBulkCopy--which will execute many times faster on very large
datasets.

Also, there's plenty of times when a simple more code oriented approach is
desired. The modelers for EF are nice--but I've always found myself much
happier as close to the metal as I can be. SqlLightning facilitates that in a
very small, performant package.

EXAMPLES
--------

Create a SqlDbContext from an application connection string:

```c#
var connStr = ConfigurationManager.ConnectionStrings["myConnStr"].ToSTring();
var ctx = SqlDbContext.Create(connStr);
```

Create a SqlDbContext from an existing Linq2Sql data context:

```c#
var connectionString = linq2SqlContext.Connection.ConnectionString;
var ctx = SqlDbContext.Create(connStr);
```

Run a stored procedure:

```c#
var connStr = ConfigurationManager.ConnectionStrings["myConnStr"].ToString();
using (var ctx = SqlDbContext.Create(connStr))
{
    ctx.ExecuteNonQuery(
        "sp_DeleteProducts",
        StoredProcParam.InParam("@userName", DbType.String, "jduv"));
}
```

Run a stored procedure with some arbitrary actions:

```c#
using (var ctx = SqlDbContext.Create(connStr))
{
    // Gets all products and returns them.
    var products = ctx.ExecuteTransaction(
        new SqlCommand()
        {
            CommandType = CommandType.StoredProcedure,
            CommandText = "sp_GetProducts"
        },
        (cmd) =>
        {
            var list = new List<Product>();
            using (var reader = cmd.ExecuteReader())
            {
                if (reader.HasRows)
                {
                    while (reader.Read())
                    {
                        list.Add(new Product()
                        {
                            ID = reader.GetInt32(0),
                            Name = reader.GetString(1)
                        });
                    }
                }
            }

            return list;
        }
    );
}
```

Insert into a table using SqlBulkCopy (assumes use of MSFT's EntityDataReader class):

```c#
var products = from p in goodProducts
                 select new
                 {
                     Name = p.Name
                 };

sqlDbContext.ExecuteSqlBulkCopy(
	tableName: "Product",
	dataReader: products.AsDataReader(), // EDR
	hasIdentity: true);
```

Run a stored procedure with a return value:

```c#
var rowVersion = ctx.ExecuteWithReturnValue<byte[]>(
    "sp_GetRowVersion",
    StoredProcParam.InParam("@id", DbType.Int32, 1),
    StoredProcParam.ReturnValue("return", DbType.Binary));
```

Run a stored procedure with an out value:

```c#
var bit = StoredProcParam.OutParam("@someOutValue", DbType.Boolean);
using(var ctx = SqlDbContext.Create(connStr))
{
    ctx.ExecuteNonQuery(
        "sp_performValidation",
        StoredProcParam.InParam("@id", DbType.Int32, 1),
        bit);
}

var bitValue = bit.Value; // grab the value after dispose
```