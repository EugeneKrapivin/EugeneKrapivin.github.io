# Managing packages in dotnet project

It is commonly agreed that in order to manage nuget packages in dotnet projects you should use the nuget package manager, which comes with support in Visual Studio and in the dotnet CLI. It's really easy to use and generally works well, for small projects it really does an awesome job.

So, post done, what's there to talk about?

Well not so fast, [stay awhile and listen...](https://www.youtube.com/watch?v=tAVVy_x3Erg)

## Paket - The alternative in the shadow of a giant?

In the company I've been working for the last 4 years (Gigya) we use a different package manager called [Paket](https://fsprojects.github.io/Paket/).
The reason as to why it was chosen in the long distant past is no longer known for sure. However, we can make some educated guesses.

### Centralized package version management (or how to handle your DLL hell)

In the past DLL hell was a common thing in large dotnet solutions, especially legacy ones which had 100+ projects. The causes of the DLL hell varied from solution to solution, but their underlying cause was really simple: we had no clue which DLLs of dependent libraries were copied into the build folders.

[Newtonsoft.Json](https://www.newtonsoft.com/json) was one culprit, but by far not the only one. Being an awesome json ser/des library that it is, it was used practically everywhere. But, alas, every library depends on a different version. Sometimes, event projects in the same solution depend on different versions of newtonsoft. (this is all pre consolidate feature in NuGet).
So what did most of us do? yeah, [assembly version redirect](https://docs.microsoft.com/en-us/dotnet/framework/configure-apps/redirect-assembly-versions) the hell out of it and pray to the gods of the runtime to not crash.

So why paket? Well, it's simple - paket has a centralized package version management and a hell of a great dependency version resolution abilities, shadowing anything the NuGet package manager could provide at the time.

For deeper details on how to manage packages with paket I suggest you take a peek at their site, it's really easy.
In a nutshell:

* Put a `paket.dependencies` file in the solution root. This file declares the dependencies used throughout the solution and their version constraints.
* Put a `paket.references` file in **each** project. This file declares the package references the specific project needs. This file has no version or source declarations, only names of packages.

... and that's basically it, the paket `paket.exe` can restore packages or install them from a packages lock file, and you are all set.

It took NuGet a couple of years to catch up. First they added the consolidate feature, which always felt "like too little too late" since the alternative was so powerful. Then they brought package lock support, which gave a boost to the ability to create reproducible builds, knowing that the packages you tested on your machine will be the same packages the CI uses to build the deliverable.

But, alas, all that was not enough to convince anyone in Gigya to ditch paket and go back to nuget, paket just worked.

### What felt wrong

Well... Nothing really. All was sort of great.

I mean, the `csproj` files were cumbersome before, but after paket they really became unreadable. Paket did some magic, used assembly version redirect tricks and other stuff. But Mostly it worked kind of great.

Before the SDK style projects you already had a `packages.config` file to configure your packages and the `csproj` file to reference those packages by path. It was cumbersome, so we just traded the `packages.config` file for a `paket.references` file and stayed with a messy (a bit less friendly) `csproj`.

Then came the SDK style project file. Gone was the `packages.config` file, now you used `<PacakgeReference>` inside the much cleaner and tidier `csproj`, no more paths to the dlls no more mess.

With the sdk style projects came also some new features like the support for default `Directory.Build.props` and `Directory.Build.targets` files. We'll come back to them in a minute.

And paket became too much magic. you no longer knew which packages the `csproj` referenced just from looking at it. you had to work with Visual Studio or begin digging through the `obj` folders to find the generated `csproj` files. The integration was a bit gritty.

1. Paket was never a first class citizen in VS, but it was neither a first-class citizen in the dotnet CLI. Most importantly, it seemed that the community around it never took-off, so no tooling, no support from awesome tools like Dependabot or NuKeeper.
2. It feels like in the end it didn't handle the DLL hell. It just postponed the reckoning since we didn't really change habits. Internal libraries still referenced conflicting versions of the same transitive dependencies and I still had no real clue what version did end-up in our packages lock file.
3. Most people really didn't understand how it worked and why. The complex dependency resolution rules were awesome, but only few understood how it all worked. It was especially felt when it didn't work.

So, I decided to research what the world of package management in dotnet came up with in the last 4 years.

## Our story begins...

Having thought about ditching paket for some time, I've decided to research a bit how other projects achieve this feat of managing their packages. So I've began looking at what open source projects do. Very quickly I've realised that many of the projects have their own methods, none of which made much sense to the outsider. Some of the methods I've found:

* use properties in a [central props file] and use those properties down the line in a [`csproj` or `Build.props` file](https://github.com/dotnet/orleans/blob/master/src/Orleans.Runtime/Directory.Build.props)
* I still fail to understand how [asp.net source code](https://github.com/dotnet/aspnetcore) manages their packages, too complex.
* NuKeeper has no centralized package version management at all, versions are "hard coded" in the `csproj` files

It really did seem that most of the projects I found were not using any kind of centralized package version management at all.

My next place to check was the [dotnet/home gitter channel](https://gitter.im/dotnet/home) where I got [some help](https://gitter.im/dotnet/home?at=5f195e5aa4905773bfb70470), explaining a couple of methods I'd like to discuss here, after researching and trying them for some time.

## Does NuGet and MSBuild have a solution?

Well, yes and no.

Yes - we can concoct something with MSBuild abilities to augment the `csproj` using some clever tricks.
No - there is nothing out-of-the-box from NuGet at this stage (there is a beta feature that I'll show in the end).

### `Directory.Build.props` and `Directory.Build.targets`

**Firstly, a short confession**: I'm not an MSBuild expert. The things below are things I learned and understood from trying things out. So, take it with a grain of salt.

If you haven't really seen those files, or did see but didn't give them much thought, here are some resources:

* [Customize your build](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2019)
* [MS Build: Things You Should Know About Project Files](https://www.youtube.com/watch?v=5HEbsyU5E1g)

There are other great sources around, but those are a great place to start from.

In short, those are files which the MSBuild process looks-up by default and applies them to your `csproj` file. The `props` are included in the beginning, the `targets` are included in the end of the process. This fact will be used to do some centralized package version management.

#### Centralized package version management MSBuild style

We will use the fact that `target` files are applied at the end of the `csproj` and create a simple, yet powerful, method to manage our packages centrally.

Packages can be included directly in the `.csproj` file (as seen in this example) or in a `Build.props` file.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net472</TargetFramework>
    <RootNamespace>proj_a</RootNamespace>
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  </PropertyGroup>

  <ItemGroup Label="External">
    <PackageReference Include="Newtonsoft.Json" />
    <PackageReference Include="Newtonsoft.Json.Schema" />
    <PackageReference Include="TaskTupleAwaiter" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\proj-b\proj-b.csproj" />
  </ItemGroup>

</Project>
```

> **Note** that the `<PackageReference>` element lacks the `Version` property.

Now, in the root of the solution we will add the magic sauce, the `targets` file. This file contains the `<PackageReference>` elements with the package version. However, instead of `Include` we use an `Update` property when defining the package name. The reason for this will be explained shortly.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<Project>
  <ItemGroup>
    <PackageReference Update="Newtonsoft.Json" Version="[12.0.3, )" />
    <PackageReference Update="Newtonsoft.Json.Schema" Version="[3.0.13, )" />
    <PackageReference Update="TaskTupleAwaiter" Version="[1.2.0, 2)" />
    <PackageReference Update="UrlBase64" Version="[0.1.2, )" />
    <PackageReference Update="ZooKeeperNetEx" Version="[3.4.12.4, )" />
  </ItemGroup>
</Project>
```

> **Note** that the `<PackageReference>` element doesn't use the `Include` property but rather an `Update` property instead.

Once we do a `dotnet restore` what happens is the MSBuild begins to prepare the project files for build, firstly applying and adding the `Directory.Build.props` (the first it finds) to the beginning, that way you can move all the metadata to it and leave the `csproj` file for things that are project oriented. In the end of the process the target files are applied.

Our little target file is residing in the root of the solution, not the project. Once MSBuild finds it, it begins running it. Each `<PackageReference>` element with an `Update` property will be matched against any existing `<PackageReference>` elements with the corresponding `Include` property, and update the element with the missing property - the version of the package.
Important thing to notice is that `Update` is really an update and not upsert (update-insert), i.e. packages that are not matched, will not be added to the csproj file.

You could find [Source code](https://github.com/EugeneKrapivin/locked-mode-restore-repro/tree/msbuild-cpvm-floating-version) in my github repo. This repo contains 2 more interesting features: package lock (which might be a separate post) and floating package versions (you really need a package lock if you plan on using this). The readme contains more technical explanations and instructions.

### `Directory.Packages.props` - the NuGet official secret (beta) sauce

If you stayed till now, here comes a [secret sauce from NuGet](https://github.com/NuGet/Home/wiki/Centrally-managing-NuGet-package-versions). It really looks like our previous solution, but with a little twist.

The `<PackageReference>` in the `csproj` stays the same and untouched. However, we use a different file to handle the centralized package version management. The `Directory.Packages.props` is a special file to the nuget CPVM feature.

It actually looks a lot like our `Directory.Build.targets` file

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<Project>
  <ItemGroup Label="External">
    <PackageVersion Include="Newtonsoft.Json" Version="[12.0.3, )" />
    <PackageVersion Include="Newtonsoft.Json.Schema" Version="[3.0.13, )" />
    <PackageVersion Include="TaskTupleAwaiter" Version="[1.2.0, 2)" />
    <PackageVersion Include="UrlBase64" Version="[0.1.2, )" />
    <PackageVersion Include="ZooKeeperNetEx" Version="[3.4.12.4, )" />
  </ItemGroup>
</Project>
```

Just notice that the `<PackageReference>` was changed to `<PackageVersion>` and the `Update` to `Include`.
Another little thing you've got to do (since it's a preview feature) is to enable this sort of centralized package version management by adding this short snippet to the `csproj` files, or alternatively to a central `Directory.Build.props` file.

```xml
<PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
</PropertyGroup>
```

A short demo of this feature could be [found here](https://github.com/EugeneKrapivin/locked-mode-restore-repro/tree/nuget-cpvm)

#### Notes

* Visual Studio UI still doesn't support this feature. Things like updates through the UI will just add the `Version` property on the `<PackageReference>` elements in all the `csproj` files that reference this package.
* For some bizarre reason, [dependency floating version](https://docs.microsoft.com/en-us/nuget/concepts/dependency-resolution#floating-versions) feature was disabled.
* Package locks feature **does not work** due to a bug, the reproduction of which could be found in the [main branch of this repo](https://github.com/EugeneKrapivin/locked-mode-restore-repro/tree/main).
* Beta feature - will probably have changes in the future.

## Conclusions and thoughts

In the long run - it's all the same, once the growing pains will be over people will work with either solution.

As a proof of concept, we've migrated all of my team's projects to use nuget package version management and never looked back. There are growing pains, such as making people understand how [dependency resolution and versioning](https://docs.microsoft.com/en-us/nuget/concepts/dependency-resolution) work. But those will pass.

The solutions are a lot cleaner and easier to understand. There is much less "magic" in the process, making it a bit easier to understand what's happening under the hood, even though as a whole it's a bit more verbose than just using paket.  
The best win/gain for us was the ability to use [NuKeeper](https://github.com/NuKeeperDotNet/NuKeeper) to handle auto package updates via merge requests, something we just could not have done with paket.

I hope that through this (not so short as I hoped) first post, you've got a glimpse into how to manage packages in your projects.



