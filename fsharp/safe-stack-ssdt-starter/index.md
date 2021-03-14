# SAFE Stack v3 with SQLProvider SSDT Tutorial
Posted on March 15, 2021

Looking create an F# powered front end React app for for a SQL Server database and you want to get started the easy way?
Then this blost post is for you! 

In this post we will cover the following:
* Creating an F# and Fable powered web site using the SAFE Stack template
* Creating a SQL Server table schema using the SSDT extension for Visual Studio Code
* Connecting your SAFE Stack application to SQL Server using the new SSDT type provider

To get started, you will need the following:
* .NET 5 CLI
* Visual Studio Code
  * Extension: Ionide
* Azure Data Studio
  * Extension: SQL Database Projects

## Installing the SAFE Stack Template
Recently, Fable 3 (the F# to JavaScript transpiler) was released. It boasted cool new features and faster compile times.
However, if you new, the hardest part is setting up your project with everything you need to get to the development happy path.
If you're an early adopter or an F# trail blazer, then maybe you would enjoy being one of the first to create something with the new Fable 3;
however, most of us just want to run a template that gets us up and running as quickly as possible.  
That is where the SAFE Stack comes in.  
Fortunately for us, the SAFE Stack v3 now supports Fable 3 and is available on NuGet (currently it is in beta).
https://www.nuget.org/packages/SAFE.Template/

Clicking on the link to the latest v3 link should give you the ".NET CLI" install command:
As of now, it is this: `dotnet new --install SAFE.Template::3.0.0-beta001`

![image](https://user-images.githubusercontent.com/1030435/110747226-0af55200-820c-11eb-959b-36091c140497.png)

Assuming you have already installed the .NET 5 CLI, run the command to install the template.

## Creating a SAFE Stack App
1) Run the new "SAFE" template.
- `dotnet new SAFE -o SafeTodo` - `-o' will create a "SafeTodo" subfolder witha  "SafeTodo.sln" solution file
- `cd SafeTodo` - change to the new "SafeTodo" directory (on Windows)

2) Restore the command line tools that we will use to build and run our SAFE Stack application:
- `dotnet tool restore` - this should install "paket", which is like NuGet but with some additional features, and the "fable" transpiler

3) Restore NuGet dependencies:
- `dotnet paket restore`

Now we can open our new app in Visual Studio Code: 
- `code .` - opens Visual Studio Code to the current folder.

Note: If you haven't already installed the Ionide extension in Visual Studio Code, you should see a prompt in the bottom right corner to install the Recommended Extensions (which is Ionide). Do that.

After installing Ionide, you should now see a "SOLUTION EXPLORER" panel on the left that shows a "src" folder with your three SAFE Stack F# projects: 
- Shared.fsproj
- Server.fsproj
- Client.fsproj

## Running the SAFE Stack App
- Open a new Terminal in Visual Studio Code
- `dotnet run` 

That's it! `dotnet run` will run the "Build.fsproj" project which contains FAKE build script tasks that will do the following automatically:
- install NPM dependencies (React, and other js libraries)
- It will start the Server component (the Web API)
- It will start the Client component (the Fable.React front end)

You should be able to view the site in your browser using the port given after the build completes:
http://127.0.0.1:8080/
NOTE: The current SAFE Stack beta template incorrectly displays http://0.0.0.0:8080/ -- disregard this and use http://127.0.0.1:8080/ instead!


## Creating a "SafeTodo" Database with Azure Data Studio

### Connecting to a SQL Server Instance
1) In the "Connections" tab, click the "New Connection" button

![image](https://user-images.githubusercontent.com/1030435/111041187-26777d00-8405-11eb-9260-5d885b0ff640.png)

2) Enter your connection details, leaving the "Database" dropdown set to "<Default>".

![image](https://user-images.githubusercontent.com/1030435/111041401-fbd9f400-8405-11eb-9d38-3444dfdc15f6.png)

### Creating a new "SafeTodo" Database
- Right click your server and choose "New Query"
- Execute this script:
``` SQL
USE master
GO
IF NOT EXISTS (
 SELECT name
 FROM sys.databases
 WHERE name = N'SafeTodo'
)
 CREATE DATABASE [SafeTodo];
GO
IF SERVERPROPERTY('ProductVersion') > '12'
 ALTER DATABASE [SafeTodo] SET QUERY_STORE=ON;
GO

```
- Right click the "Databases" folder and choose "Refresh" to see the new database.

NOTE: Alteratively, you can install the "New Database" extension in Azure Data Studio which gives you a "New Database" option when right clicking the "Databases" folder.

### Create a "Todos" Table
``` SQL
CREATE TABLE [dbo].[Todos]
(
  [Id] UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
  [Description] NVARCHAR(500) NOT NULL,
  [IsDone] BIT NOT NULL
)
```

## Creating an SSDT Project (.sqlproj)
At this point, you should have a SAFE Stack solution and an empty "SafeTodo" SQL Server database.
In this step, we will use Azure Data Studio with the "SQL Database Projects" extension to create a new SSDT (SQL Server Data Tools) "SQL Project" that will live in our SAFE Stack .sln. 

1) Install the "SQL Database Projects" extension.
2) Right click the SafeTodo database and choose "Create Project From Database" (this option is added by the "SQL Database Projects" extension)

![image](https://user-images.githubusercontent.com/1030435/111041910-99cebe00-8408-11eb-9fcf-9271b40984d7.png)

3) Configure a path within your SAFE Stack solution folder and a project name and then click "Create". NOTE: If you choose to create an "ssdt" subfolder as I did, you will need to manually create this subfolder first.

![image](https://user-images.githubusercontent.com/1030435/111042131-cc2ceb00-8409-11eb-809f-d08ef10932ee.png)

4) You should now be able to view your SQL Project by clicking the "Projects" tab in Azure Data Studio.

![image](https://user-images.githubusercontent.com/1030435/111051915-b0830e00-8424-11eb-9f29-e31eb58ed502.png)

5) Finally, right click the SafeTodoDB project and select "Build". This will create a .dacpac file which we will use in the next step.


## Create a TodoRepository Using the new SSDT provider in SQLProvider

### Installing SQLProvider from NuGet
Next we will install SQLProvider and System.Data.SqlClient to the Server project:
- Open a new terminal
- From the "SafeTodo" root folder: `dotnet paket add SQLProvider -p Server`
- From the "SafeTodo" root folder: `dotnet paket add System.Data.SqlClient -p Server`

### Initialize Type Provider
Next we will wire up our type provider to generate database types based on the compiled .dacpac file.

1) In the Server project, create a new file, `Database.fs`. (this should be above `Server.fs`).

``` fsharp
module Database
open FSharp.Data.Sql

[<Literal>]
let ssdtPath = @".\ssdt\SafeTodoDB\bin\Debug\SafeTodoDB.dacpac"

type DB = 
    SqlDataProvider<
        Common.DatabaseProviderTypes.MSSQLSERVER_SSDT, 
        SsdtPath = ssdtPath,
        UseOptionTypes = true
    >

let createContext (connectionString: string) =
    DB.GetDataContext(connectionString)
```

2) Create `TodoRepository.fs` below `Database.fs`.

``` fsharp
module TodoRepository
open FSharp.Data.Sql
open Database
open Shared

/// Get all todos that have not been marked as "done". 
let getTodos (db: DB.dataContext) = 
    query {
        for todo in db.Dbo.Todos do
        where (not todo.IsDone)
        select 
            { Shared.Todo.Id = todo.Id
              Shared.Todo.Description = todo.Description }
    }
    |> List.executeQueryAsync

let addTodo (db: DB.dataContext) (todo: Shared.Todo) =
    async {
        let t = db.Dbo.Todos.Create()
        t.Id <- todo.Id
        t.Description <- todo.Description
        t.IsDone <- false

        do! db.SubmitUpdatesAsync()
    }
```

3) Create `TodoController.fs` below `TodoRepository.fs`.

``` fsharp
module TodoController
open Database
open Shared

let getTodos (db: DB.dataContext) = 
    TodoRepository.getTodos db

let addTodo (db: DB.dataContext) (todo: Todo) = 
    async {
        if Todo.isValid todo.Description then
            do! TodoRepository.addTodo db todo
            return todo
        else 
            return failwith "Invalid todo"
    }
```

4) Finally, replace the stubbed todosApi implementation in `Server.fs` with our type provided implementation.

``` fsharp
module Server

open Fable.Remoting.Server
open Fable.Remoting.Giraffe
open Saturn
open System
open Shared

let todosApi =
    let db = Database.createContext @"Data Source=.\SQLEXPRESS;Initial Catalog=SafeTodo;Integrated Security=SSPI;"
    { getTodos = fun () -> 
        async {
            let! todos = TodoController.getTodos db
            return todos
        }
      addTodo = TodoController.addTodo db }

open Microsoft.AspNetCore.Http

let fableRemotingErrorHandler (ex: Exception) (ri: RouteInfo<HttpContext>) = 
    printfn "ERROR: %s" ex.Message
    Propagate ex.Message
    
let webApp =
    Remoting.createApi()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.fromValue todosApi
    |> Remoting.withErrorHandler fableRemotingErrorHandler
    |> Remoting.buildHttpHandler

let app =
    application {
        url "http://0.0.0.0:8085"
        use_router webApp
        memory_cache
        use_static "public"
        use_gzip
    }

run app

```

