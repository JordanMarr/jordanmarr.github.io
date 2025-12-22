---
title: "Create a Cloudflare Worker in .NET?!"
date: 2025-12-22
categories:
  - F#
tags:
  - fsharp
  - fable
  - cloudflare
---

*This post is part of the [F# Advent 2025](https://sergeytihon.com/2025/11/30/f-weekly-48-2025-fsadvent-starts-dec-1st/) event hosted by Sergey Tihon. Thanks, Sergey!*

Cloudflare Workers is an incredibly compelling serverless platform: fast cold starts, generous free tier (100K requests/month), no credit card required, and a frictionless deployment experience. There's just one problem—it's a JavaScript platform with no .NET SDK.

...or is there?

Thanks to the [Fable](https://fable.io/) compiler and [CloudflareFS](https://github.com/speakeztech/CloudflareFS/) by Houston Haynes at SpeakEZ Technologies, we can write Cloudflare Workers in F# and have them compile to JavaScript. Let me show you how.

## The Problem

Every Wednesday is trash pickup day in my neighborhood. But recycling? That's every *other* Wednesday. And yard waste is on Tuesdays. Oh, and holidays can shift any of these around.

So every week I'd find myself asking: "Is it recycle week?" Then I'd pull out my phone, navigate to the city's schedule, and check. Every. Single. Week.

A few years ago, I solved this by creating an F# Alexa Skill. It would parse my voice request, call the city's API to get pickup dates for the week, and respond with an answer. If any pickups were shifted due to holidays, it would append that info too. It worked great—except for AWS Lambda's notoriously slow cold start times. For something I only call once a week, it was *always* cold.

Recently, I switched from Alexa to Google Assistant. I really didn't want to go through the hassle again: creating a Google developer account, learning a new SDK, navigating all that red tape. I wanted something simpler.

## Requirements

My requirements were straightforward:

- Easy to implement
- Free hosting
- *Truly* free—no credit card required "just in case"
- Fast cold start times (critical for once-a-week usage)

After looking around, **Cloudflare Workers** was the only solution that checked *every* box. And as a bonus, I discovered that the iPhone Shortcuts app can call an API and read the response aloud—triggered via Siri. No vendor-specific SDK required. Just a simple HTTP endpoint.

## The Secret Sauce

So how do we write a Cloudflare Worker in .NET?

**[Fable](https://fable.io/)** is an F# to JavaScript compiler. It lets you write F# code and compile it to clean, idiomatic JavaScript that runs anywhere JS runs—including Cloudflare Workers.

**[CloudflareFS](https://github.com/speakeztech/CloudflareFS/)** by Houston Haynes (SpeakEZ Technologies) provides Fable bindings for the Cloudflare Workers API. It gives you typed access to requests, responses, environment variables, and more.

You'll also want these Fable-compatible NuGet packages (shout-out to [Maxime Mangel](https://github.com/MangelMaxime) for his incredible contributions to the Fable ecosystem):

- **Fable.Promise** – JS Promise interop via the `promise { }` computation expression
- **Fable.Fetch** – Fetch API bindings for HTTP calls
- **Thoth.Json** – Type-safe JSON encoding/decoding

You'll also need to install the Fable compiler as a dotnet tool:

```bash
dotnet tool install -g fable
```

## The Code

Let's walk through the key files.

### wrangler.toml

This is Cloudflare's configuration file. It defines your worker's name, entry point, and environment variables:

```toml
name = "recycle-worker"
main = "dist/Main.js"
compatibility_date = "2024-01-01"

[vars]
PLACE_ID = "THE APP DEFAULTS TO MY PERSONAL PLACE ID WHICH GOES HERE!"
```

The `main` field points to the compiled JavaScript output. Environment variables under `[vars]` are accessible in your code.

### package.json

The build and deployment workflow:

```json
{
  "name": "recycle-worker",
  "version": "1.0.0",
  "description": "Cloudflare Worker for recycle schedule",
  "main": "dist/Main.js",
  "scripts": {
    "build": "dotnet fable . --outDir dist",
    "dev": "npm run build && wrangler dev",
    "deploy": "npm run build && wrangler deploy",
    "watch": "dotnet fable watch . --outDir dist"
  },
  "devDependencies": {
    "wrangler": "^4.42.0"
  }
}
```

The magic happens in the scripts:
- `npm run build` – Compiles F# to JavaScript via Fable
- `npm run dev` – Build and run locally
- `npm run deploy` – Build and deploy to Cloudflare

### Main.fs

Here's the heart of the worker:

```fsharp
module CloudflareWorker.Main

open System
open Fable.Core
open CloudFlare.Worker.Context
open CloudFlare.Worker.Context.Helpers
open CloudFlare.Worker.Context.Helpers.ResponseBuilder

let private parsePathSegments (path: string) =
    path.Split('/', StringSplitOptions.RemoveEmptyEntries)

let private getRecycleAnswer placeId =
    promise {
        let currentDate = DateTime.Now
        let weekStart, weekEnd = Utils.getWeekContaining currentDate
        let! events = CityApi.fetchEvents placeId (weekStart, weekEnd)
        let answer = RecycleLogic.buildAnswer currentDate events
        return text answer 200
    }

let fetch (req: Request) (env: Env) (ctx: ExecutionContext) =
    promise {
        let method = Request.getMethod req
        let path = Request.getPath req
        let segments = parsePathSegments path

        match method, segments with
        // Default: use PLACE_ID from environment
        | "GET", [||]
        | "GET", [| "recycle" |] ->
            let placeId = env |> Utils.requireEnvKey "PLACE_ID"
            return! getRecycleAnswer placeId

        // Allow specifying a different PLACE_ID in the URL
        | "GET", [| "recycle"; placeId |] ->
            return! getRecycleAnswer placeId

        | "GET", [| "health" |] ->
            return json {| status = "ok"; timestamp = DateTime.Now.ToString("o") |} 200

        | _ ->
            return json {| error = "Not Found"; path = path |} 404
    }

// Cloudflare Workers require a default export with a `fetch` handler
[<ExportDefault>]
let handler = {| fetch = fetch |} :> obj
```

A few things to note:

- **The `fetch` function** is the entry point. It receives the `Request`, `Env` (environment variables), and `ExecutionContext`.
- **`promise { }`** is Fable's computation expression for working with JavaScript Promises.
- **CloudflareFS helpers** like `Request.getMethod`, `Request.getPath`, and `ResponseBuilder.text`/`json` give you typed access to the Cloudflare API.
- **`[<ExportDefault>]`** is required—Cloudflare Workers expect a default export with a `fetch` handler.
- **Pattern matching on routes** makes it trivial to define your API endpoints.

The `CityApi.fetchEvents` function calls an external API to get pickup dates. `RecycleLogic.buildAnswer` is a pure function that takes the events and returns a formatted string response. No Cloudflare bindings, no IO—just plain F#.

## Deployment

Ready to deploy? Just run:

```bash
npm run deploy
```

Here's what it looks like:

```
> npm run build && wrangler deploy

> dotnet fable . --outDir dist

Fable 4.28.0: F# to JavaScript compiler

Parsing CloudflareWorker.fsproj...
Project and references (32 source files) parsed in 260ms

Started Fable compilation...
Compiled 32/32: dist\fable_modules\Thoth.Json.10.2.0\Decode.fs
Fable compilation finished in 4362ms

 ⛅️ wrangler 4.53.0
─────────────────────────────────────────────
Total Upload: 85.03 KiB / gzip: 18.01 KiB
Your Worker has access to the following bindings:
Binding                                           Resource
env.PLACE_ID ("xxxxxxxx-xxxx-xxxx-xxxx-...")      Environment Variable

Uploaded recycle-worker (3.31 sec)
Deployed recycle-worker triggers (1.29 sec)
  https://recycle-worker.your-subdomain.workers.dev
Current Version ID: 1b1e4191-b5b1-4d8f-bcac-7136ba439f65
```

That's it. The first time you run it, Wrangler will prompt you to sign in to Cloudflare. It will create your subdomain, deploy your code, and return your live URL.

No dashboard clicking. No manual configuration. One command.

## Wrapping Up

If you're a .NET developer who's been eyeing Cloudflare Workers but thought it was JavaScript-only territory—think again. With F#, Fable, and CloudflareFS, you get:

- The expressiveness of F#
- Type-safe access to the Cloudflare API
- Lightning-fast cold starts
- A generous free tier
- Dead-simple deployment

And hey, if you're a C# developer who clicked on this post because it said ".NET"... welcome to F#. You might like it here.

## Links

- [CloudflareFS on GitHub](https://github.com/speakeztech/CloudflareFS/)
- [Fable Compiler](https://fable.io/)
- [Cloudflare Workers](https://workers.cloudflare.com/)
- [F# Advent 2025](https://sergeytihon.com/2025/11/30/f-weekly-48-2025-fsadvent-starts-dec-1st/)
