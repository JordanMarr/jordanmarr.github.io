# Tips for Unit Testing Fable Apps using .NET
October 5, 2025

While there are front-end testing frameworks for Fable like Fable.Mocha, I generally prefer unit testing my Fable logic using .NET instead.

Elmish gets you most of the way there because it naturally pushes you towards creating an `update` function of "pure" handlers for your messages.

However, there are some pitfalls that will prevent you from being able to test your Elmish loop via .NET. This post covers several tips to help you write Fable Elmish code that can be tested from a .NET project.

## Separate Elmish Model from React View

While I tend to start with the model and view combined into one file for convenience, this will usually cause problems when testing if your view code contains any React specific JS functions that will fail on the .NET side.

This can be resolved by splitting the Elmish model, init and update functions into separate files. That way, your .NET unit tests only need to reference the model file.

Ex: 
* TimeSheetModel.fs - contains Elmish `Model` type, `init` and `update` functions.
* TimeSheetPage.fs - contains front-end view code (Feliz, Fable.React, Fable.Lit, etc.)

## Stub Your Fable.Remoting API

If you are using Fable.Remoting for strongly typed access to your endpoints (highly recommended!), you will likely have an `api` binding in your Fable project to access your Fable.Remoting endpoints. This will cause your .NET unit test code to fail.

The solution is to use the `FABLE_COMPILER` directive to create a stub of your Fable.Remoting server API bindings:

```fsharp
/// Provides the server side api.
let api =
#if FABLE_COMPILER
    Remoting.createApi()
    |> Remoting.withRouteBuilder normalizeRoutes
    |> Remoting.buildProxy<ServerApi>
#else
    // Stubbed defaults for your .NET unit tests
    ServerApi.mkDefault()
#endif
```

This is the way I like to stub my server api record:

```fsharp
module ServerApi =
    let private def<'T> = Unchecked.defaultof<'T>

    /// The default implementation is used for unit testing purposes only.
    let mkDefault () =
        {
            Auth_User = fun _ -> def
            Auth_Admin_Users = fun _ -> def
            Auth_Admin_UserProjectFiltersFormData = fun _ -> def
            Auth_Admin_SaveUserProjectFilters = fun _ -> def
            Auth_Admin_UpdateUser = fun _ -> def
            Auth_Admin_Impersonate = fun _ -> def
            TimeSheet_GetFormData = fun _ -> def
            TimeSheet_GetEntries = fun _ -> def
            TimeSheet_SaveEntries = fun _ -> def
            // ... more endpoints
        }
```

## Avoid Browser-Specific Functions in Elmish Handlers

### The Problem

There is nothing preventing you from making impure calls to JS browser-specific functions in your Elmish handlers. For example, you may call `Toastify.info "Save complete."` at the end of your `Msg.Save` handler. While this is convenient, this binding function will fail when executed in the context of a .NET unit test.

```fsharp
match msg with

| SaveCompleted result ->
    match result with
    | Ok () ->
        Toastify.success "Save completed."
        model, Cmd.none
    | Error err ->
        match err with
        | ForgeNeedsAuthorization ->
            Toastify.error "Must be logged into BIM 360."
            model, Cmd.none
        | GeneralError msg ->
            Toastify.error msg
            model, Cmd.none
```

### The Solution: Custom Elmish Commands

Commonly used impure functions like toast messages can be turned into custom Elmish `Cmd` handlers:

```fsharp
module Cmd =
    open Elmish

    let private onFail ex = failwith "toast failed"
    let info msg = Cmd.OfFunc.attempt Toastify.info msg onFail
    let success msg = Cmd.OfFunc.attempt Toastify.success msg onFail
    let error msg = Cmd.OfFunc.attempt Toastify.error msg onFail
    let warn msg = Cmd.OfFunc.attempt Toastify.warn msg onFail
```

Which can be called like this:

```fsharp
match msg with

| SaveCompleted result ->
    match result with
    | Ok () ->
        model, Cmd.success "Save completed."
    | Error err ->
        match err with
        | ForgeNeedsAuthorization ->
            model, Cmd.error "Must be logged into BIM 360."
        | GeneralError msg ->
            model, Cmd.error msg

// other handlers...
```

This pattern converts the "impure" IO functions into declarative instructions that are deferred for later. In the case of unit testing your Elmish `update` function, these impure instructions will be returned as an array of commands and can be simply ignored, allowing you to focus solely on the contents of the resulting model.

```fsharp
[<Test>]
let ``Test Save Completed Handler`` () =
    // Prepare model
    let model, _ = DrawingLogPage.init ()
    let saveCompletedMsg = Msg.SaveCompleted ()
    // Simulate saving, ignoring the Cmd array.
    let model, _ = DrawingLogPage.update saveCompletedMsg model
    // assert against model...
```

## Using Cmd.ofEffect for One-Off Impure Operations

For less common operations, it may be less efficient to create a custom Cmd. In this case, you can just use the `Cmd.ofEffect` directly.

For example, you may need to clear a File Input using DOM manipulation after importing a file:

```fsharp
| HandleImportExcelResult result ->
    match result with
    | Ok imported ->
        let importSummary = ConduitSchedule.mergeImport model.Conduits imported

        // Append the new rows to the model and sort by ConduitId
        { model with
            // Update properties...
        }, Cmd.ofEffect (fun _ ->
            // (Even better, this can be refactored into its own helper function.)
            let input = Browser.Dom.document.getElementById("file-input") :?> Browser.Types.HTMLInputElement
            input.value <- ""
        )

    | Error errorMsg ->
        model, Cmd.error errorMsg
```

## Using an IO Record for Complex Impure Logic

Finally, you may need to utilize impure functions as part of your handler logic. You can isolate these into a record of impure functions that are passed to your `init` or `update` functions as needed.

A common example is needing to show a JS `confirm` dialog to confirm the user's intent.

### Before: Impure Update Function

```fsharp
let update (msg: Msg) (model: Model) =
    match msg with
    | SetConduitId conduitId ->
        if Browser.Dom.window.confirm "Are you sure you want to overwrite the Id?" then
            { model with
                ConduitId = conduitId
            }, Cmd.none
        else
            model, Cmd.info "Operation canceled."
```

The `confirm` function is obviously browser-side JS code, and so it will fail in your .NET unit tests.

### After: Testable Update Function

A fix is to create an `IO` record type of impure functions at the top of your Elmish file:

```fsharp
/// Impure operations used by this page.
type IO =
    {
        confirm: string -> bool
    }
```

Then refactor your `update` handler to take in `io: IO` as an input parameter:

```fsharp
let update (io: IO) (msg: Msg) (model: Model) =
    match msg with
    | SetConduitId conduitId ->
        if io.confirm "Are you sure you want to continue?" then
            { model with
                ConduitId = conduitId
            }, Cmd.none
        else
            model, Cmd.info "Operation canceled."
```

Your page/view code can then supply the standard IO functions:

```fsharp
/// Impure IO functions that can be mocked for testing.
let io =
    {
        IO.confirm = fun msg -> Browser.Dom.window.confirm msg
    }

[<ReactComponent>]
let Page () =
    let model, dispatch = React.useElmish(init, update io)
```

Your .NET unit test code can easily pass in stubbed `IO` function implementations as required by your tests:

```fsharp
[<Test>]
let ``Test SetConduitId Handler`` () =
    let io = { IO.confirm = fun _ -> true }

    let conduitIdMsg = Msg.SetConduitId Guid.Empty

    // Simulate handler
    let model, _ = DrawingLogPage.update io conduitIdMsg model

    // assert against model...
```

## Conclusion

By following these patterns, you can write Fable applications that are fully testable using .NET unit tests. The key is to keep your Elmish model and update logic pure and isolated from browser-specific code, using techniques like:

- Separating your model from your view
- Stubbing Fable.Remoting APIs with compiler directives
- Converting impure operations into Elmish commands
- Using `Cmd.ofEffect` for one-off impure operations
- Passing impure functions as an IO record for complex scenarios

This approach gives you the best of both worlds: a productive Fable development experience with the speed and reliability of .NET unit testing.
