# SSDT Type Provider
*Posted on Dec 21, 2020*

I'm very happy to announce that my PR to add SSDT support to SQLProvider has been merged and the new release is now available on [NuGet](https://www.nuget.org/packages/SQLProvider/)!  🎉

TLDR; you can now generate table entities with a path to a local SSDT project!

## What is SQLProvider?

SQLProvider is an F# database library that can generate a strongly typed interface to your database at design time with nothing but a simple connection string. It does this by creating a connection to the database at design-time via F# Type Provider feature.

```F#
type DB = SqlDataProvider<Common.DatabaseProviderTypes.MSSQLSERVER,
                             connectionString>
```

Once the types have been generated, you can then create queries with generated entities using the query computation expression.

```F#
let tasksWithCategories =
    query {
        for p in ctx.Dbo.Projects do
        for t in p.``dbo.ProjectTasks by Id`` do
        for c in t.``dbo.ProjectTaskCategories by Id`` do
        where (p.IsActive && not (p.IsDeleted))
        sortBy t.Sort
        select
            {| Id = t.Id
               Task = t.Name
               Category = c.Name
               Project = p.Name |}
    }
```



The main benefit to using a Type Provider for database access is that it allows the compiler to keep you from making errors. If the database schema changes, the F# compiler help you to make the appropriate changes in your code.

Instead of having to create complex ORM mapping schemes, your mapping will instead consist of directly mapping your domain entities or DTOs in and out of the generated entities (and it even comes with helpers for doing that in less lines of code).

One down side to this approach is that a connection to the database is required in order to compile. This can be problematic on build servers that do not have access to your database. It could also make it difficult for a new developer that does not have database access to compile the code.  Fortunately, the SQLProvider has a very excellent feature that allows you to cache the generated types in a local schema file. While this approach is great, it can at times feel slightly cumbersome to toggle the cache option off and on as you make changes to the database. 

Some people work around this by spinning up a temporal database in their build script. This is obviously quite a bit of work, but it eliminates the need to toggle the schema cache feature.

## What is SSDT?

SSDT stands for SQL Server Data Tools. An SSDT ".sqlproj" can be created in [Visual Studio 2019](https://docs.microsoft.com/en-us/sql/ssdt/download-sql-server-data-tools-ssdt?view=sql-server-ver15) or Azure Data Studio via the [SQL Database Projects Extension](https://docs.microsoft.com/en-us/sql/azure-data-studio/extensions/sql-database-project-extension?view=sql-server-ver15). Given a connection string, SSDT will create a local snapshot of a database, copying all of the tables, views, stored procedures and functions into your source code repository as .sql create scripts. It provides powerful synchronization features that allow you to make changes either in the database or in the SSDT script files. Regardless of where you make the changes, the sync works bidirectionally. 

The beauty of this approach is that it allows you keep your schema changes in source control alongside the code. Even better, this allows coordinate code and database schema changes together within each branch.

## SQLProvider + SSDT!

The new SSDT provider can be created by using the new `MSSQLSERVER_SSDT` database vendor type, along with the `SsdtPath` to the local SSDT project, and a `ResolutionPath` to the `Microsoft.SqlServer.Management.SqlParser` assembly (which is used to parse the .sql create scripts).

```F#
type DB = SqlDataProvider<
            Common.DatabaseProviderTypes.MSSQLSERVER_SSDT,
            ResolutionPath = resPath,
            SsdtPath = ssdtPath>

```

[Click here to RTFM](https://fsprojects.github.io/SQLProvider/core/mssqlssdt.html)

The main advantages to using the SSDT provider is that it allows you to generate types without the need for a live database connection. This makes it easier to make schema changes without having to invalidate the cache file. It also eliminates problems on the build server.

### Limitations

There are a few limitations to using the new SSDT provider:

#### Limited support for views

Only very simple views are being parsed for this first release. Views that have any nesting or functions will likely have missing columns. Over time, I hope to improve the parser so that it can handle many of the more common views.

#### No support for Stored Procs and Functions (yet)

The first version of SSDT provider does not yet implement support for stored procs or functions. This is planned for a future release, but I was anxious to get v1 released before Christmas, so concessions were made! 🎄

#### No support for the "individuals" feature

The individuals feature basically serves as a strongly typed look-up for values in a table. This requires a live database connection, and so it is not supported by the SSDT provider.

#### SQL Server Only

SSDT only works with SQL Server. (Although SQLProvider still supports *many* [databases](https://fsprojects.github.io/SQLProvider/index.html) with the classic connection string based providers).

## Conclusion

I want to thank [@Thoriumi](https://twitter.com/Thoriumi) and the SQLProvider team for all their hard work over the years maintaining such a wonderfully useful tool for free! I learned that it is a lot of hard work down in "the basement" to enable that magical "it-just-works!" experience for the end users.  I personally have used this library almost every day for the last year and a half and I love it!

Oh and if you happen to run into any bugs, please direct your torches and pitchforks to the SQLProvider GitHub Issues page. 

(╯°□°）╯︵ ┻━┻
