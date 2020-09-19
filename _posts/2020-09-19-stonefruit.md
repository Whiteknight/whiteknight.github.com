---
layout: post
categories: [projects]
title: StoneFruit
---

I was running into a similar problem over and over again, across different projects and different employers: I had a need to host lots of little scripts and commandlets related to various project tasks: maintenance, testing, deployment, analysis, etc. Migrate this, seed that. Synchronize two data stores. Push files. Kick-off jobs. Send messages to the bus, or consume messages that had already been sent. There are all sorts of little jobs we wanted to script up, include in the build so everything stays up-to-date, and run on the commandline. I saw different commandline utilities like `git`, which did what I wanted: The `git` command by itself doesn't do much, but when you supply a *verb* to it like `git add` or `git clone` real work gets done. The `git` application, in a sense, is little more than a namespace and dispatch engine for the individual verbs. Each of these verbs is essentially a separate mini-program, taking different commandline arguments and producing different output. This is what I wanted, to host a lot of little mini-commandlets, each too small to be a program by themselves, but too large to just write fresh each time I needed to use it.

This, in essence, is **StoneFruit**. It's a CLI framework for hosting commandlets, including argument parsing and dispatch. It's modular, pluggable, and integrates deeply with the DI container of your choice (actually I have only written adaptors for a couple DI containers, but they're easy to make and more are in development).

## History

I have written several drafts of this idea, several times, across several employers. The need always starts the same: I have a collection of repetitive tasks related to a project I'm working on, and I want to automate these tasks. But there are lots of them, so writing an application-per-task is prohibitive. Instead, I try to bring them together into an application and dispatch to the correct script based on commandline arguments. The problem with doing something like this on employer-time is that I can't ever really focus on doing it the right way. The infrastructure always turns out to be thin and rushed because there isn't a way to allocate proper time to it. 

In January 2020, being unhappy with my most recent implementation and wanting to start a rewrite, I finally decided to take the idea home. After putting together a basic [parsing library](http://whiteknight.github.io/2020/02/24/parserobjects.html) to help, I began a ground-up rewrite of the StoneFruit idea with better implementations of everything, more features, and actual unit tests. Since January I have made some major changes to the library and the core architecture of it, and finally I'm feeling confident enough to release a 1.0.0 version to [Nuget](https://www.nuget.org/packages/StoneFruit).

(It's purely coincidental that during this time Microsoft released a project called DragonFruit which also deals with commandline argument parsing. I guess there's an implicit association between the commandline and fruit that I wasn't previously aware of)

## Quick Example

I have used JIRA at work for a while, and I wanted to automate some basic tasks using the helpful (but woefully incomplete) [Atlassian.SDK](https://www.nuget.org/packages/Atlassian.SDK/) library. So, I put together a StoneFruit-based application to wrap some of the features of this library and expose them on the commandline. The application, which I'm calling "JiraCLI" and I might release publically if I can make it completely agnostic to my specific workflow and environment, exposes a couple common verbs so I don't have to go to the website and navigate around by clicking links and buttons. Here are some examples:

* `jiracli issues` provides a list of all active issues assigned to me, grouped by sprint and ordered by priority
* `jiracli issue ABC-123` shows an overview of the issue including relevant fields and summary description
* `jiracli steal ABC-123` assigns the issue to myself, if it's not assigned to me already
* `jiracli open ABC-123` moves the issue to the Open status, if a transition exists
* `jiracli watch ABC-123` adds myself as a watcher to an issue

I have recently moved to a new employer and started a rewrite of this program. I'm trying to use configuration and metadata from the server to make it more generalizable to other environments and workflows instead of hard-coding everything for my own needs. If I can get it into a more generally-useful state, I'll push the code up to github and write another rambling blog post about it.

## Core Concepts

The [documentation page](https://whiteknight.github.io/StoneFruit/) has lots of good information, quick-starts and details. Here, in brief, is a list of core concepts. 

1. A **Handler** is a bit of code which is holds the logic you want to execute. A handler can be a class implementation of `IHandler` or it can be a method
2. A **Verb** is the textual command you enter at the commandline. StoneFruit reads your verb and dispatches the request to the appropriate handler
3. A **Command** is a combination of verb and all other commandline arguments. 
4. A **Script** is a list of zero or more commands (with templating) which can be used like any other handler.

So with the above JIRA example in mind, the `jiracli open ABC-123` command is actually a script which translates to `jiracli transition status=Open ABC-123`. In addition to user-defined scripts like this, there are built-in scripts which are executed in response to events in the StoneFruit engine, such as startup, shutdown and unhandled exception. By modifying these scripts, you can completely customize the entire experience. You can adjust the startup script to show a custom welcome message, validate credentials, or setup user state. 

There are two modes of operation, **Headless** and **Interactive**. In Headless mode, as I showed with the jiracli example above, you pass a single command on the commandline. StoneFruit will execute that one command and immediately exit. In Interactive mode, you execute the application with no arguments, and it opens an interactive prompt where you can enter commands and maintain state between them.

There's also a concept of **Environments**, which can be roughly thought of as temporary state or configuration. For example, your workplace may have separate environments for Local, Test and Production. You can setup StoneFruit with separate configurations for all of these and switch between them at any time. So if you want to compare a bit of data between Test and Prod in interactive mode, you can do something like this:

    env change test
    get-interesting-data
    env change prod
    get-interesting-data

No reloading the application, no manual fiddling with configuration files. 

## Get Started

You don't need a DI container to get started with StoneFruit, but the whole system is designed to be DI-friendly and a container can help get you moving much faster. Here's a quick example, from the documentation site, for how to get started with the Lamar container:

```csharp
public static void Main(string[] args)
{
    var services = new ServiceRegistry();
    services.SetupEngine(engineBuilder => 
        ...
    );
    var container = new Container(services);
    var engine = container.GetService<Engine>();
    engine.RunWithCommandLineArguments();
}
```

That `.SetupEngine()` extension method is where all the magic happens. You have several configuration options to play with there, including scripting and timeouts and all sorts of pluggable things. Then, once you resolve the `engine` from the container, you can run the application and StoneFruit will handle the rest. All you have to do is put together a few `IHandler` classes in your project, then the DI container will find them automatically and wire everything up for you. Then, because everything is run out of the DI container, you can inject all your application services and `DbContext` or whatever into all the handlers you write. I've spent a lot of time trying to make it quick and easy to get started. This is not an area where you want to spend a lot of time on ceremony.

## Future

The 1.0.0 release is recently out, but I'm already putting together a list of TODO notes for things to add for 1.1.0 and even some changes I already want to make for 2.0.0. For now though, I'm going to let 1.0.0 get some use and gather some feedback, while I switch gears and start working on something else. I'm sure I'll post about what those somethings else are in the next post.





