# MSBuild / Dotnet Core Example

## How to setup and consume inherited build targets

### Overview

This project is an example of how to setup an MSBuild project with a Nuget package output (`.nupkg`) that correctly includes shared build project files (`.props` and `.targets`) so that they can be consumed without further action from dotnet / Visual Studio projects that reference the package.

It has  a [partner project](https://github.com/JollyWizard/ConsumeInheritableBuildTarget.csproj) that is an example on how to consume it. 

### Essential Concerns

The `dotnet` tool includes `msbuild` and `nuget` functionality in it's subcommands.

`msbuild` package definitions come as `.*proj` (XML/Project) files, while `nuget` package definitions come as `.nuspec` (JSON) files.

Conventions are established to package inheritable msbuild `.props` and `.target` files inside the `.nupkg` archives.  

The history of these conventions is complicated with features being added and dropped over time, of course with only minimal advisory documentation. 

`nuget` and `msbuild` had diverged workflows, until recent unification under the `dotnet` tool. Features are common across all three tools depending on versioning. 

`nuget` requires a `.nuspec` file, but much of the information  in a `.nuspec` file (author, title, namespace, etc.) is redundant for a `.*proj` solution. 

Also redundant are the `<file>` definitions of `.nuspec`, considering resources are already declared, remapped, and cascading with `.*proj` file `<ItemGroup>` definitions.

To remove redundancy, `dotnet` generates the `.nuspec` file as an intermediate (in `obj\*`). The `.nuspec` file tells the `.nupkg` generator exactly what to place in the package and where. If it's not declared and mapped, it's not included.

`<ItemGroup>` definitions are converted into `.nuspec` mappings during the `msbuild` `pack` target. To enable inheritable targets, the item definitions need to properly map into the `msbuild`/`nupkg` convention for `build` resources.

[Special attributes](https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#including-content-in-a-package) `Pack` and `PackagePath` for each item definition control the `msbuild` `pack` behavior.

This project demonstrates a corect mapping, assuming all other conventions are followed:

```
  <ItemGroup>
    <None Include="build\**" Pack="true" PackagePath="build\" />
  </ItemGroup>
 ```

 The convention required:
 
 * A placeholder file in the `lib\$(TargetFramework)` directory.
 * Targets must be properly named and placed inside `build\$(TargetFramework)\`:

 ```
 lib\
    $(TargetFramework)\
      _._
 
 build\
    $(TargetFramework)\
      $(AssemblyName).props
      $(AssemblyName).targets
 ```

 The placeholder file under $(TargetFramework) may or may not be necessary, but is included here until proven otherwise.

 Once You meet all these requirements, your project is ready to build, package, and publish.

 ### This example in action

 As you can see from this project, if you `dotnet publish` the output to a package, the `.nupkg` archive will include `build\net6.0\...`, mirroring the parent project structure (pictured).

 <image src='media\VS-Project-Structure.PNG'/>

 Befire it can be included as a dependency it needs to be published to a repository. This example includes a `PublishProfiles` entry called `.nuget-local.pubxml` that you can use in VS studio to publish to `..\.nuget-local\`. Configure this as a source in VS Studio / nuget, and then create a child package (or use the example).

 When this project is added as a dependency via:
 
 ```
  <ItemGroup>
   <PackageReference Include="InheritableBuildTarget" Version="1.0.0" />
  </ItemGroup>
 ```

 The inheriting project will automatically have the build scripts applied and they can be seen in VS as dependency resources.

 <image src='media\VS-ResourceAsDependency.PNG'/>

 ### Troubleshooting

 Changing these settings for projects in motion can lead to mysterious and persistent errors. Be aware of the following areas:

 * nuget packages are only unpacked on first use, if you republish a version to a local repository, it will not be updated by consuming applications. You must locate the local nuget unpacked repository and delete the contents before doing a `dotnet restore`.

 * VS Projects hide `bin` and `obj` by default, and do not clean them thoroughly. You should be able to delete both these folders at any time.

 * `dotnet pack` `dotnet publish` may produce inconsistent artifacts with each other or the Visual Studio `pack`/`publish` context menu commands. 
 
    * During the creation of this package. `dotnet` refused to produce artifacts that included the `build` folder, but `VS` context menu continued to work. During this writing the problem went away.

  * There are at lease two ways to trigger a package on every build.
    Trying diffferent approaches may help invalidate cached resources.
    * `<GeneratePackageOnBuild>true</GeneratePackageOnBuild>`
    * `<Project Sdk="Microsoft.NET.Sdk" DefaultTargets="Build;Pack">`

### Epilogue: Technical History and Need for Example

While lot of documentation / questions / blogs exists on how to inherit build targets from nuget packages, the conventions and practices for `nuget`, `msbuild`, and inherited files have changed and evolved during the move from Dotnet Framework to Dotnet Core, without full clarification in official documentation.

Common documentation examples rely on `.nuspec` files that create redundancy and confusion by replacing current convention with override resources. 

The explicit secondary configuration file was antiquated with the merger of the `dotnet`, `msbuild`, and `nuget` tools, now standard in Visual Studio, but updated tutorial documentation wasn't provided to supplement the change.

Forum questioneers and bloggers have addressed the topic in several ways, but these resources are often overly specific, pegged in time, full of red herrings, and hard to decipher as a generalized solution.  

Other resources may be available, but hard to find because of the generic language involved in msbuild naming schemes.

Given the deep utility of inheriting build tasks, it is prudent to have a publically available, clearly labeled example, that can be compiled and used without secondary technical concerns. 