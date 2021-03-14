# SAFE Stack v3 with SQLProvider SSDT Tutorial
Posted on March 15, 2021

Looking to create an F# powered React app using the new SAFE Stack v3, SQL Server and the new [SSDT F# Type Provider](https://jordanmarr.github.io/fsharp/ssdt-type-provider/)?
Then you're in luck! This post will provide a step-by-step tutorial to get you up and running.

In this post we will cover the following:
* Creating a Fable 3 powered web site using the latest SAFE Stack template
* Creating a SQL Server database using Azure Data Studio
* Adding a .sqlproj to your SAFE Stack solution using the SQL Database Projects extension in Azure Data Studio
* Creating a data module in your SAFE Stack application using the new SSDT Type Provider in the SQLProvider library.

To get started, you will need the following:
* .NET 5 CLI
* Visual Studio Code
  * Extension: Ionide
* Azure Data Studio
  * Extension: SQL Database Projects

### The Stack
For the last two years, I have maintained a greenfield web app that is essentially a SAFE Stack app; it has been a joyful experience! My daily driver has been Visual Studio Professional which I have developed with since .NET v1.1. 
I chose to write this post around VSCode + Ionide + Azure Data Studio because it is a cross platform stack, and I want to accommodate the many F# devs that are coming over from other backgrounds and other platforms. So this post was a learning experience for me as well!

But I do want to add that VS2019 Community Edition and above has built-in SSDT .sqlproj support via the "SQL Server Data Tools" extension (available as an [option](https://docs.microsoft.com/en-us/sql/ssdt/download-sql-server-data-tools-ssdt?view=sql-server-ver15) in the VS installer).  

## Installing the SAFE Stack Template
Fable 3 was recently released boasting new features, faster compile times, and a cool new way to launch as a dotnet tool. 
While Fable 3 does have its own starter template, it only covers the front end. 
The SAFE Stack takes that and adds:
- A backend web API powered by ASP.NET and Giraffe
- A middle tier Shared project for entity models
- An RPC-style communication via Fable.Remoting
- A working webpack config
- A FAKE powered build project that automates running and deploying your app. 

The new SAFE Stack v3 (currently in beta) supports Fable 3 and is available on NuGet.
https://www.nuget.org/packages/SAFE.Template/

Clicking on the link to the latest v3 link should give you the ".NET CLI" install command:
As of now, it is: `dotnet new --install SAFE.Template::3.0.0-beta001`

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

4) OPTIONAL - Initialize a local git repository (the SAFE Stack already includes a .gitignore file!):
- `git init`

Now we can open our new app in Visual Studio Code: 
- `code .` - opens Visual Studio Code to the current folder.

_NOTE: If you haven't already installed the Ionide extension in Visual Studio Code, you should see a prompt in the bottom right corner to install the Recommended Extensions (which is Ionide). Do that._

Assuming Ionide is installed, you should now see an Ionide tab that displays a "SOLUTION EXPLORER" panel on the left with a "src" folder with your three SAFE Stack F# projects: 
- Shared.fsproj
- Server.fsproj
- Client.fsproj

## Running the SAFE Stack App
- Open a new Terminal in Visual Studio Code
- `dotnet run` 

That's it! Entering `dotnet run` from the SafeTodo root folder will run the "Build.fsproj" project which contains FAKE build script tasks that will do the following automatically:
- Install NPM dependencies (React, and other js libraries) if they are not yet installed
-  Start the Server component (the Giraffe Web API) in watch mode
- Start the Client component (the Fable.React front end) in watch mode

You should be able to view the site in your browser using the port given after the build completes:
http://localhost:8080/
NOTE: The current SAFE Stack beta template displays http://0.0.0.0:8080, but I replaced that with http://localhost:8080.


## Creating a "SafeTodo" Database with Azure Data Studio

### Connecting to a SQL Server Instance
1) In the "Connections" tab, click the "New Connection" button

![image](https://user-images.githubusercontent.com/1030435/111041187-26777d00-8405-11eb-9260-5d885b0ff640.png)

2) Enter your connection details, leaving the "Database" dropdown set to `<Default>`.

![image](https://user-images.githubusercontent.com/1030435/111041401-fbd9f400-8405-11eb-9d38-3444dfdc15f6.png)

### Creating a new "SafeTodo" Database
- Right click your server and choose "New Query"
- Execute this script:

``` sql
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

_NOTE: Alternatively, if you don't want to manually create the new database, you can install the "New Database" extension in Azure Data Studio which gives you a "New Database" option when right clicking the "Databases" folder._

### Create a "Todos" Table

``` sql
CREATE TABLE [dbo].[Todos]
(
  [Id] UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
  [Description] NVARCHAR(500) NOT NULL,
  [IsDone] BIT NOT NULL
)
```

## Creating an SSDT Project (.sqlproj)
At this point, you should have a SAFE Stack solution and a minimal "SafeTodo" SQL Server database with a "Todos" table.
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
let SsdtPath = __SOURCE_DIRECTORY__ + @"/../../ssdt/SafeTodoDB/bin/Debug/SafeTodoDB.dacpac"

// TO RELOAD SCHEMA: 1) uncomment the line below; 2) save; 3) recomment; 4) save again and wait.
//DB.GetDataContext().``Design Time Commands``.ClearDatabaseSchemaCache

type DB = 
    SqlDataProvider<
        Common.DatabaseProviderTypes.MSSQLSERVER_SSDT, 
        SsdtPath = SsdtPath,
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
open Microsoft.AspNetCore.Http

let todosApi =
    let db = Database.createContext @"Data Source=.\SQLEXPRESS;Initial Catalog=SafeTodo;Integrated Security=SSPI;"
    { getTodos = fun () -> TodoController.getTodos db
      addTodo = TodoController.addTodo db }

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

## Run the App!
From the VS Code terminal in the SafeTodo folder, launch the app (server and client):

`dotnet run`

You should now be able to add todos.

![image](https://user-images.githubusercontent.com/1030435/111055048-044f2080-8440-11eb-9efc-ae454ff071c4.png)

## Adding a "Done" Feature
Now that we can add and view todos, we need a way to mark them as complete.
First, we need to modifiy the `Shared.Todo` entity to have an `IsDone` property.
``` fsharp
type Todo =
    { Id : Guid
      Description : string
      IsDone : bool }

module Todo =
    let isValid (description: string) =
        String.IsNullOrWhiteSpace description |> not

    let create (description: string) =
        { Id = Guid.NewGuid()
          Description = description
          IsDone = false }
```

Now we need to fix the `getTodos` function so that it loads the new IsDone field. While we're at it, since we now have an `IsDone` field, let's change it so that it returns all todos; that way, we can render the completed todos differently in the UI.
``` fsharp
/// Get all todos (regardless of IsDone status) 
let getTodos (db: DB.dataContext) = 
    query {
        for todo in db.Dbo.Todos do
        select 
            { Shared.Todo.Id = todo.Id
              Shared.Todo.Description = todo.Description
              Shared.Todo.IsDone = todo.IsDone }
    }
    |> List.executeQueryAsync
```

Next we will add an `updateTodo` function to the TodoRepository:

``` fsharp
let updateTodo (db: DB.dataContext) (todo: Shared.Todo) = 
    async {
        let! existingTodo =
            query { 
                for t in db.Dbo.Todos do
                where (t.Id = todo.Id)
                select t
            }
            |> Seq.tryHeadAsync

        // Fail if this unexpected scenario occurs
        let existingTodo = existingTodo |> Option.defaultWith (fun () -> failwith "Update failed; Todo was deleted!")

        existingTodo.Description <- todo.Description
        existingTodo.IsDone <- todo.IsDone

        do! db.SubmitUpdatesAsync()
        return todo
    }
```

Moving our way up the stack, now we need to add a new `updateTodo` handler on the `Shared.ITodosAp`:
``` fsharp
type ITodosApi =
    { getTodos : unit -> Async<Todo list>
      addTodo : Todo -> Async<Todo>
      updateTodo : Todo -> Async<Todo> }
```

Then we need to add it to the `TodoController`::
``` fsharp
let updateTodo (db: DB.dataContext) (todo: Todo) = 
    async {
        if Todo.isValid todo.Description 
        then return! TodoRepository.updateTodo db todo
        else return failwith "Invalid todo"
    }
```

And we need to fix the now broken `todosApi` implementation in `Server.fs`:
``` fsharp
let todosApi =
    let db = Database.createContext @"Data Source=.\SQLEXPRESS;Initial Catalog=SafeTodo;Integrated Security=SSPI;"
    { getTodos = fun () -> TodoController.getTodos db
      addTodo = TodoController.addTodo db
      updateTodo = TodoController.updateTodo db }
```

NOTE: We are using a shorthand way of setting the `updateTodo` field by partially applying the `TodoController.getTodos` function with the `db` dependency argument; however, you can also write it out this way if you prefer:
``` fsharp
      updateTodo = fun todo -> TodoController.updateTodo db todo
```

Now for the UI!
Let's make it so that clicking a todo will 
- Toggle the IsDone field
- Send it to the new `updateTodo` function on the Fable.Remoting API
- Update the model after the todo has been updated on the server

For this, we will need
- A new Msg handler, `ToggleIsDone`
- Another Msg handler, `UpdatedTodo` (that will be triggered after the async response is received)

``` fsharp
type Msg =
    | GotTodos of Todo list
    | SetInput of string
    | AddTodo
    | AddedTodo of Todo
    | ToggleIsDone of Todo
    | UpdatedTodo of Todo
```

Then we will implement these handlers in the `update` function:

``` fsharp
    | ToggleIsDone todo ->
        let toggled = { todo with IsDone = not todo.IsDone }        
        model, Cmd.OfAsync.perform todosApi.updateTodo toggled UpdatedTodo
```

``` fsharp
    | UpdatedTodo todo -> 
        { model with 
            Todos = 
                model.Todos
                |> List.map (fun t -> 
                    if t.Id = todo.Id then todo else t
                )
        }, Cmd.none
```

Now for rendering the todos, we want to
- Conditionally show todos with `IsDone = true` with a line-through style
- Add a "pointer" cursor style when hovering over todos to indicate that they are clickable
- Add an "onClick" event handler when clicking a todo that will dispatch our new `ToggleIsDone` message

``` fsharp
                for todo in model.Todos do
                    Html.li [                         
                        prop.onClick (fun e -> dispatch (ToggleIsDone todo))
                        prop.text todo.Description 
                        prop.style [
                            if todo.IsDone then style.textDecoration textDecorationLine.lineThrough
                            style.cursor "pointer"
                        ]
                    ]
```

## Wrapping Up
If all things went well, you should now have a working SAFE Stack v3 app with a full data layer using the new SQLProvider SSDT type provider!

![image](https://user-images.githubusercontent.com/1030435/111082097-dec42480-84dc-11eb-8721-939a1ff2930c.png)


The final source code for this tutorial will be posted on my GitHub account as "SafeTodo". Feel free to post in the issues if you have any issues, or hit me up on [Twitter](https://twitter.com/jordan_n_marr)!
