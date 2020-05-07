---
title: "VS Code and C/C++ Extension (cpptools) Secrets"
date: 2020-05-07
categories:
  - programming
tags:
  - vscode
---
I love VS Code and I use it as my only IDE these days.

The [C/C++ Extension](https://code.visualstudio.com/docs/languages/cpp)
is great, especially its IntelliSense features.
But it can be a bit tricky to set up.  The extension can use up a lot of
disk space and CPU resources.
I've found that it is helpful to know something about how this all works, especially since I am working on a very large project.

## References
* [Extension Home Page](https://code.visualstudio.com/docs/languages/cpp)
* [Extension FAQ](https://code.visualstudio.com/docs/cpp/faq-cpp)

## Configuration
I'm using the `.vscode/c_cpp_properties.json` file instead of trying to do all the configuration with vscode settings because the file allows me to have more than one configuration.
This makes it easier to try things by just making a new configuration and swap back to a working one if things don't go well.
Note also that the currently-selected configuration appears in the bottom status bar.

The only properties I control now from the vscode settings are some that are performance related:
```json
    "C_Cpp.enhancedColorization": "Disabled",
    "C_Cpp.workspaceParsingPriority": "medium",
```

The `enhancedColorization` feature was buggy at one point and I think it slowed things down when I had it on.
TBH, I have not tried it lately.

The `workspaceParsingPriority` setting is an attempt to keep the extension from taking over the machine if it is trying to index a lot of files.
I've recently trimmed this list and find that the default higher setting works OK.

### Engine Selection
The extension gives you the choice of the default engine or a tag parser.
The default engine seems strategic and the documents say it works better, so I am going with that.
This implies that the `browse.path` property is not used.

### compileCommands Property
I'm working with GitHub repositories that have submodules.
Some of these projects generate a `compile_commands.json` file and some do not.
Even if they all did, I'm not sure that there is a way to give more than one of these files to the extension.
(The file contains full command lines for compiling each file in the project and the extension uses this file to find include files and -D define settings, etc.)

I tried putting a `.vscode/c_cpp_properties.json` file in each submodule, but it seems that the extension only pays attention to the one in the super-project.

Instead, I manually provide the `includePath` and `defines` properties in the top-level extension configuration file that covers the main project and its submodules.

### Include Path and Defines Property
For a complex project, it can be pretty hard to determine all the include directories.
One approach is to capture a compile command and collect all the `-I` directives.

If there are `-D` defines that the extension can't know about, such as defines provided in a script or build file of some sort instead of in a header, then I set them in the `defines` property.
If I see areas of code incorrectly greyed out, I can usually fix this by adding a define.

### Other Properties
Since I am cross-compiling with a specific toolchain, I also provide the `compilerArgs` property to specify the target as well as the `compilerPath` property.
The other settings are pretty standard and obvious.

## Taming the Beast
Because of the size of my projects, it can take the extension several minutes to scan all the files and build its database.
Sometimes it is just better to sit back and let it finish.
I can tell when it is building the database by seeing a "disk" or "database" icon in the bottom right of the vscode status bar.
I can also hover over it with the mouse to see progress.

I can also click on this icon to make it pause and resume.

I suppose that there are other steps that can be taken to make things better here.
I could try to reduce the size of the include path so that fewer files are scanned.
But this could lead to unsatisfying IntelliSense operation and I'd have to incrementially add directories back to the path as I discover them missing.

Some have said that putting `**` at the end of the path to indicate recursive search can slow things down.
I think I can confirm, but experimenting with things like this takes time.

As I said, I usually just let it finish while I do something else.

## Important Data Locations
This extension generates a lot of data and if I'm not careful, it can get out of hand and cause mysterious disk space usage.

It seems to generate two major databases:

* The main browse database
* The IntelliSense cache

### Main Browse Database

On Linux, vscode stores workstation information under `~/.config/Code/User/workspaceStorage`.
Under `workspaceStorage` there is a hashed directory name for every workspace I've created.
Unless told otherwise, the extension puts the browse database here.

This is somewhat bad because if you have a workspaces with IntelliSense databases in them, this workspace storage can take up a lot of room if you have a lot of workspaces.
This space is also unclaimed if you delete the files in your workspace, but don't do anything in vscode to explicitly delete the workspace.

I couldn't find a place in vscode to delete a workspace, that is, the workspace storage.
So this is really a more general problem of clearing out unwanted workspace storage.
If it were not for the large browse databases, this would not be a huge problem.

All this means is that I end up with a lot of stale workspaceStorage directories under my HOME directory.
It is also hard to figure out which ones I don't need anymore since the directory names are hashed.
It is possible to tell by poking around inside of these hashed directories.
But it is often easier to just delete old ones by their date.
Most of the information stored here isn't vital and can be regenerated.

I can specify the location of this browse database and I did that once by putting it in my top-level `.vscode` directory so that I could "keep an eye on it".
But I didn't see any particular advantage to doing that, other than just being more aware of its existence.

### IntelliSense Cache
The other very interesting database to be aware of is the IntelliSense cache.
On Linux, the default location is `$HOME/.cache/vscode-cpptools/ipch`.

The interesting thing about this cache is that the extension adds entries to it each time you hover over an item in the editor or follow the symbol to its definition.
So, this cache grows as you use IntelliSense features.
The default size is 5120MB and this space is allocated per-user and not per-workspace.