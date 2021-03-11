# Getting Started with the SAFE Stack and SQLProvider SSDT

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

3) Restore NuGet dependencies for all projects in the solution:
- `dotnet restore SafeTodo.sln`

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

## Creating a Todo Table Schema in Azure Data Studio
At this point we will shift gears and focus on the SQL Server schema. For this task, we will open Azure Data Studio.
- Open Azure Data Studio
- Install the SQL Database Projects extension
- Install the SQL Server Schema Compare extension


## Installing SQLProvider from NuGet
Next we will install SQLProvider to the Server project:
- Open a new terminal
- From the "SafeTodo" root folder: `dotnet paket add SQLProvider -p Server`* 
- Hit `CTRL`+`SHIFT`+`P` to bring up the console
- Find and select `Database Projects: Create Project from Database`
