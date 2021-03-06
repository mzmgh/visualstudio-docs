---
title: "Customize your build | Microsoft Docs"
ms.custom: ""
ms.date: "06/14/2017"
ms.technology: msbuild
ms.topic: "conceptual"
helpviewer_keywords: 
  - "MSBuild, transforms"
  - "transforms [MSBuild]"
ms.assetid: d0bceb3b-14fb-455c-805a-63acefa4b3ed
author: mikejo5000
ms.author: mikejo
manager: douge
ms.workload: 
  - "multiple"
---
# Customize your build

MSBuild projects that use the standard build process (importing `Microsoft.Common.props` and `Microsoft.Common.targets`) have several extensibility hooks that can be used to customize your build process.

## Adding arguments to command-line MSBuild invocations for your project

A `Directory.Build.rsp` file in or above your source directory will be applied to command-line builds of your project. For details, see [MSBuild Response Files](../msbuild/msbuild-response-files.md#directorybuildrsp).

## Directory.Build.props and Directory.Build.targets

In versions of MSBuild prior to version 15, if you wanted to provide a new, custom property to projects in your solution, you had to manually add a reference to that property to every project file in the solution. Or, you had to define the property in a *.props* file and then explicitly import the *.props* file in every project in the solution, among other things.

However, now you can add a new property to every project in one step by defining it in a single file called *Directory.Build.props* in the root folder that contains your source. When MSBuild runs, *Microsoft.Common.props* searches your directory structure for the *Directory.Build.props* file (and *Microsoft.Common.targets* looks for *Directory.Build.targets*). If it finds one, it imports the property. *Directory.Build.props* is a user-defined file that provides customizations to projects under a directory.

### Directory.Build.props example

For example, if you wanted to enable all of your projects to access the new Roslyn **/deterministic** feature (which is exposed in the Roslyn `CoreCompile` target by the property `$(Deterministic)`), you could do the following.

1. Create a new file in the root of your repo called *Directory.Build.props*.
2. Add the following XML to the file.

  ```xml
  <Project>
    <PropertyGroup>
      <Deterministic>true</Deterministic>
    </PropertyGroup>
  </Project>
  ```
3. Run MSBuild. Your project’s existing imports of *Microsoft.Common.props* and *Microsoft.Common.targets* find the file and import it.

### Search scope

When searching for a *Directory.Build.props* file, MSBuild walks the directory structure upwards from your project location (`$(MSBuildProjectFullPath)`), stopping after it locates a *Directory.Build.props* file. For example, if your `$(MSBuildProjectFullPath)` was *c:\users\username\code\test\case1*, MSBuild would start searching there and then search the directory structure upward until it located a *Directory.Build.props* file, as in the following directory structure.

```
c:\users\username\code\test\case1
c:\users\username\code\test
c:\users\username\code
c:\users\username
c:\users
c:\
```

The location of the solution file is irrelevant to *Directory.Build.props*.

### Import order

*Directory.Build.props* is imported very early in *Microsoft.Common.props*, so properties defined later are unavailable to it. So, avoid referring to properties that are not yet defined (and will thus evaluate to empty).

*Directory.Build.targets* is imported from *Microsoft.Common.targets* after importing *.targets* files from NuGet packages. So, it can be used to override properties and targets defined in most of the build logic, but at times it may be necessary to do customizations within the project file after the final import.

### Use case: multi-level merging

Suppose you have this standard solution structure:

```
\
  MySolution.sln
  Directory.Build.props     (1)
  \src
    Directory.Build.props   (2-src)
    \Project1
    \Project2
  \test
    Directory.Build.props   (2-test)
    \Project1Tests
    \Project2Tests
```

It might be desirable to have common properties for all projects *(1)*, common properties for *src* projects *(2-src)*, and common properties for *test* projects *(2-test)*.

For MSBuild to correctly merge the "inner" files (*2-src* and *2-test*) with the "outer" file (*1*), you must take into account that once MSBuild finds a *Directory.Build.props* file, it stops further scanning. To continue scanning, and merge into the outer file, place this into both inner files:

`<Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />`

A summary of MSBuild's general approach is as follows:

- For any given project, MSBuild finds the first *Directory.Build.props* upward in the solution structure, merges it with defaults, and stops scanning for more
- If you want multiple levels to be found and merged, then [`<Import...>`](../msbuild/property-functions.md#msbuild-getpathoffileabove) (shown above) the "outer" file from the "inner" file
- If the "outer" file does not itself also import something above it, then scanning stops there
- To control the scanning/merging process, use `$(DirectoryBuildPropsPath)` and `$(ImportDirectoryBuildProps)`

Or more simply: the first *Directory.Build.props* which doesn't import anything, is where MSBuild stops.

## MSBuildProjectExtensionsPath

By default, `Microsoft.Common.props` imports `$(MSBuildProjectExtensionsPath)$(MSBuildProjectFile).*.props` and `Microsoft.Common.targets` imports `$(MSBuildProjectExtensionsPath)$(MSBuildProjectFile).*.targets`. The default value of `MSBuildProjectExtensionsPath` is `$(BaseIntermediateOutputPath)`, `obj/`. This is the mechanism that NuGet uses to refer to build logic delivered with packages, that is, at restore time, it creates `{project}.nuget.g.props` files that refer to the package contents.

This extensibility mechanism can be disabled by setting the property `ImportProjectExtensionProps` to `false` in a `Directory.Build.props` or before importing `Microsoft.Common.props`.

> [!NOTE]
> Disabling MSBuildProjectExtensionsPath imports will prevent build logic delivered in NuGet packages from applying to your project. Some NuGet packages require build logic to perform their function and will be rendered useless when this is disabled.

## .user file

Microsoft.Common.CurrentVersion.targets imports `$(MSBuildProjectFullPath).user` if it exists, so you can create a file next to your project with that additional extension. For long-term changes you plan to check into source control, prefer changing the project itself, so that future maintainers do not have to know about this extension mechanism.

## MSBuildExtensionsPath and MSBuildUserExtensionsPath

> [!WARNING]
> Using these extension mechanisms makes it harder to get repeatable builds across machines. Try to use a configuration that can be checked into your source control system and shared among all developers of your codebase.

By convention, many core build logic files import

```
$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\{TargetFileName}\ImportBefore\*.targets
```

before their contents, and

```
$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\{TargetFileName}\ImportAfter\*.targets
```

afterward. This allows installed SDKs to augment the build logic of common project types.

The same directory structure is searched in `$(MSBuildUserExtensionsPath)`, which is the per-user folder `%LOCALAPPDATA%\Microsoft\MSBuild`. Files placed in that folder will be imported for all builds of the corresponding project type run under that user's credentials. The user extensions can be disabled by setting properties named after the importing file in the pattern `ImportUserLocationsByWildcardBefore{ImportingFileNameWithNoDots}`. For example, setting `ImportUserLocationsByWildcardBeforeMicrosoftCommonProps` to `false` would prevent importing `$(MSBuildUserExtensionsPath)\$(MSBuildToolsVersion)\Imports\Microsoft.Common.props\ImportBefore\*`.

## Customizing the solution build

> [!IMPORTANT]
> Customizing the solution build in this way applies only to command-line builds with `MSBuild.exe`. It **does not** apply to builds inside Visual Studio.

When MSBuild builds a solution file, it first translates it internally into a project file and then builds that. The generated project file imports `before.{solutionname}.sln.targets` before defining any targets and `after.{solutionname}.sln.targets` after importing targets, including targets installed to the `$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\SolutionFile\ImportBefore` and `$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\SolutionFile\ImportAfter` directories.

For example, you could define a new target to write a custom log message after building `MyCustomizedSolution.sln` by creating a file in the same directory named `after.MyCustomizedSolution.sln.targets` that contains

```xml
<Project>
 <Target Name="EmitCustomMessage" AfterTargets="Build">
   <Message Importance="High" Text="The solution has completed the Build target" />
 </Target>
</Project>
```

## See Also

 [MSBuild Concepts](../msbuild/msbuild-concepts.md)
 [MSBuild Reference](../msbuild/msbuild-reference.md)
