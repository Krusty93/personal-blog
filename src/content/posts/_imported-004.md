+++
date = '2022-01-06T14:18:57Z'
title = '[Imported] Enforcing a consistent code style and quality in your team-wide .NET projects'
description = 'Imported post about how to enforce a consistent code style and quality in your team-wide .NET projects using EditorConfig and StyleCop'
url = 'dotnet-code-style'
tags = [
    "imported-devto",
    "dotnet",
    "dev",
    "visualstudio",
    "tutorial"
]
+++

**This post was originally published on [Dev.to](https://dev.to/krusty93/enforcing-a-consistent-code-quality-and-style-in-your-team-wide-net-projects-4m62).**

Each developer has his/her own coding style, usually inherited from personal preferences, habits and years of experience. Just think of naming conventions, indentation, spaces, curly braces, blank lines, etc.: there are many elements that can be typed in various ways, and none of them are wrong but just different.

Especially in big teams, mixing different styles has some consequences:

- codebase is not coherent due to lot of discrepancies in code style
- hundreds of compiler warnings/suggestions may hide the important ones
- new developers don't know what style they shall use to write code in early days
- PR reviews may become a fight between people with different point of views
- reading the code could be difficult - just think at the variable naming convention that should help the reader to understand the variables scope

Then, the goal is creating a _team style_ rather than a _personal style_. Moreover, keeping the solution free of warnings is definitely _a good move_.

## I understand that enforcing code style rules in a team is important. But how can we achieve that?

Just few years ago, Microsoft added the `.editorconfig` file support to Visual Studio. This kind of file defines a set rules that override the local Visual Studio settings. So you can work on multiple projects owned by different teams with their own rules without having to change every time your local Visual Studio settings.

And above all, **the whole team has the same settings**.

Last but not least, it is possible to define rules about code quality as well. To give you an idea, it is possible to throw a compiler error if an `IDisposable` object is not properly disposed, or when an `async` method is not `await`ed.

_NOTE: Visual Studio Code needs [this extension](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig) to work with `.editorconfig` files._

Starting with `.editorconfig` files is quite simple: create a file named `.editorconfig`. and put it in the solution folder; then add it to the Visual Studio solution by right clicking on the solution file -> Add -> Existing Item:

{{< cloudinary src="v1773853397/3a0b3b6uaa2q4xhyokfx_xfrnhr.png" alt="File system structure" caption="File system structure" >}}

{{< cloudinary src="v1773853434/830ktof2gd9oec7k67in_ujyikb.png" alt="sln structure" caption="sln structure" >}}

Visual Studio displays an UI when you try to open the `.editorconfig` file, but personally I don't like it. I think it is way quicker to manually edit it using VS Code.

As covered later in this post, `.editorconfig` files support inheritance, so the first thing to do is to set the nearly created file as the top most using the `root` property:

```properties
root = true
```

Then, you can set any rule you would like to set. Personally, I start with the indentation rules for any file in the solution, and then I go deeper with more specific rules for each file extension and namespace. For example:

```properties
[*] # Any file uses space rather than tabs
indent_style = space

# Code files
[*.{cs,csx,vb,vbx}] # these files use 4 spaces to indent the code
indent_size = 4
insert_final_newline = true
charset = utf-8-bom

# XML project files
[*.{csproj,vbproj,vcxproj,vcxproj.filters,proj,projitems,shproj}]
indent_size = 2

# XML config files
[*.{props,targets,ruleset,config,nuspec,resx,vsixmanifest,vsct}]
indent_size = 2

# JSON files
[*.json]
indent_size = 2

# Powershell files
[*.ps1]
indent_size = 2

# Shell script files
[*.sh]
end_of_line = lf
indent_size = 2

# Dotnet code style settings:
[*.{cs,vb}]
```

You can set hundreds of rules, of any type. On [Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/categories) there is an (almost) complete list of all the available rules that you can try locally and find the balance that suits best for your needs.
Each rule can have a specific severity: you can treat warnings as errors or demote them to suggestions, or even disable them. Be careful, the most bottom rule has the precedence over the rest.

By now, Visual Studio is reading the `.editorconfig` file and changing the local settings but as you may noticed, if you set some warning-default rule to `error` and try to build the project, you realized that the compiler works even if Visual Studio is showing the rule broke. This happens because you need to setup MSBuild by simply adding these two properties to each of your `.csproj` (or the `Directory.Build.props` parent file):

```xml
<PropertyGroup>
    <TargetFramework>...</TargetFramework>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
</PropertyGroup>
```

_NOTE: By default, Visual Studio analyses open documents only. If you wish to scan the entire solution go in `Tools` -> `Options` -> `Text Editor` -> `C#` (or your favorite language) -> `Advanced` -> Set `Background Analysis Scope` to `Entire Solution`.

## Namespace rules

Sometimes you want to define some rule (or exception) for a specific project or namespace. For example, I want to enforce the suffix `Async` to any asynchronous method in the solution but for unit tests and controllers.
Luckily `.editorconfig` files allow you to achieve a solution quite straightforward. Basically, you have two ways:

- defining exceptions in the root `.editorconfig` file
- use inheritance writing others files

### Define exceptions in the root file

At the bottom of the `.editorconfig`, define the namespace of where you would like to apply the exception. For example:

```properties
[src/**{.UnitTests,/Controllers}/**.cs]
# Async methods should have "Async" suffix
dotnet_naming_rule.async_methods_end_in_async.severity = none
```

### Using inheritance

Inheritance is a good alternative. Do you remember the `root` property we set at the beginning? That means that the given `.editorconfig` is applied to the whole solution. But however it is possible to create others `.editorconfig` files in subfolders that will inherit all the rules from the `root` file. **Rules defined here takes the precedence**. Getting back the previous example, you can navigate to the `Controllers`/`UnitTests` folders and create a new file. Do NOT define the `root=true` property and add the rule:

 ```properties
# Async methods should have "Async" suffix
dotnet_naming_rule.async_methods_end_in_async.severity = none
```

### Shall I manually change any code formatting error?

This is a tricky question with several solutions.

The first solution is to use the built-in Visual Studio feature - `Code clean up`. Personally I don't like it for the following reasons:

- it has a specific shortcut (CTRL+K, CTRL+E) that developers may forget to use every time they edit a file (we work under pressure sometimes!)
- it's annoying
- it's a **local** setting that must be configured and _there is no way to share it with your team_.

For these reasons, I had been using `Productivity Power Tools`, an (historical) extensions developed by Microsoft. It provides a feature to format the entire file when it is saved (through UI or the shortcut CTRL+S). It makes simply impossible to forget to format your file before committing it in source control.

But, unfortunately Microsoft decided to remove this feature from the PPT 2022 version - apparently without any reason, there are _plenty_ of negative feedbacks for this.

Luckily, a developer named `Elders` has published an extension called [Format document on Save](https://github.com/Elders/VSE-FormatDocumentOnSave) and it does simply what its name says: it formats the document when the user save it!


#### I want to sort and remove unnecessary using statement in my project, but I have to do it manually even if I set the rule. Any way to automate it?

Yes, you can do it!
Going in `Tools` -> `Options` -> `Format Document on Save` you can set the commands to be executed when user saves the file.
In order to format the file and then sort and removing using statements, set it as follow:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1qy1zio4o3jvnfocd79j.png)

##### Good, but any team member shall remember to set this feature on?

Nope! This extension is able to read a file named `.formatconfig` that you can check in your source control and share with your team. It doesn't need to be added to the Visual Studio solution, but just to stay in the same level of the solution file. Local user preferences are ignored.

Here's an example of definition:

```properties
root = true

[*.*]
enable = true
enable_in_debug = true
command = Edit.FormatDocument Edit.RemoveAndSort
allowed_extensions = .*
denied_extensions =
```

Anything familiar? 🐱‍💻

## Working example

Last thing to do is leaving you a link to a repository where you can find a fully working solution and especially an `.editorconfig` file with lot of rules defined. You can get the remaining ones on [Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/overview).

[GitHub](https://github.com/Krusty93/CodeStyle)

Enjoy! 🎉

UPDATE: Looking at the [Visual Studio 2022 development roadmap](https://github.com/dotnet/roslyn/issues/40163), the next preview (17.2) includes a synchronization between the `.editorconfig` file and the code clean up profile. This is really exciting!
