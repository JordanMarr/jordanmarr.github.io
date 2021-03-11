# Getting Started with the SAFE Stack and SQLProvider SSDT

Looking create an F# powered front end React app for for a SQL Server database and you want to get started the easy way?
Then this blost post is for you! 

In this post we will cover the following:
* Creating an F# and Fable powered web site using the SAFE Stack template
* Creating a SQL Server table schema using the SSDT extension for Visual Studio Code
* Connecting your SAFE Stack application to SQL Server using the new SSDT type provider

To get started, you will need the following:
* .NET 5
* Visual Studio Code
  * Ionide Extension
  * The SSDT Extension

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
Now we just need to run the new "SAFE" template.
`dotnet new SAFE -o SafeTodo`
* `-o SafeTodo` will create a "SafeTodo" subfolder that contains a solution file named "SafeTodo.sln"

