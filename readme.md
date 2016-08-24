# Quick Start
Xamarin.Forms.Carousel repo contains an alpha Xamarin Forms build environment. Similar to other .NET foundation repos (e.g. coreclr) everything starts in a shell. From there you'll need to build (or at least restore nuget packages) before opening the solution. The next generation build environment is being developed in the xfproj branch. For now, the following commands should get you off the ground:

## Opening Solution

1. launch shell via `\env\env.lnk`
2. type `restore` to restore nuget files
3. cd `src\build` and type 'build' to build task assembly
3. open `\src\Xamarin.Forms.CarouselView.sln`

## Building like CI does

1. open `cmd.exe` \env\env.lnk
2. type `build` to build both debug and release package
3. nuget package will be at `\bin\bld\release\carouselView\nuget\`
4. the build will be archived at `\drp\number\10001\`

# Xamarin.Forms Build System
The Xamarin.Forms build environment addresses challenges encountered while developing CarouselView with the immediate goal of simplifying design, build, test, and packaging of Xamarin.Forms libraries. 

In practice, this means having the ability to take a machine with a freshly installed OS and in a single command (possibly powershell now that it's cross platform), install Visual Stuido, install Xamarin Studio, install 3ed party tools (git, nuget, etc), download the source, clean, restore, build, package, publish, deploy, and test on all platforms. Basically, CI "out-of-the-box". This works focuses on the clean, restore, build, package, and publish steps.

## Highlights
This section enumerates selected achievements of this effort specific to Xamarin.Forms library creation. General build enhancements are covered in a later section.

Consumption of Xamarin.Forms libraries is simplified. The number of binaries a Xamarin.Forms app needs to reference to use a library is reduced from 3 (portable, platform, and shim [to support the linker]) to 1. This is achieved by compiling the portable and shim logic into the platform library. This allows a `RenderWithAttribute` applied to the Xamarin.Forms element to directly reference the platform renderer ([see here][2]). This obviates the need for the shim library and dodges a large class of potential linker issues. A compiler error is still generated during library construction if the platform logic references internal portable logic (note that until VSIP integration happens, Intellisense will not complain about such references). Under the hood, this is achieved by kicking off additional compilations of the project. The code can check for the `COMPOSITE` compilation symbol to know what type of compilation is occurring.  

Creation of libraries is simplified. The number of C# projects required for building a library which supports all Xamarin.Forms platforms is reduced from 13 (portable, Android, iOS [classic & unified], and Windows [tablet, phone, uap] + shims) to 1. This is achieved by "merging" the 13 project files into a single project file with each merged project file becoming its own platform. So, for example, the Android CarouselView library can be built like this:

    src\carouselView\lib> msbuild /p:platform=monodroid

To build the iOS classic version substitute `monodroid` with `monotouch`. To build all platforms at once use the group platform `mobile`. 

Creation of apps is simplified. The number of C# projects required to build an app (as is necessary for library testing) is similarly reduced from 6 (Android, iOS [classic & unified], and Windows [tablet, phone, uap]) to 1 and also has a corresponding set of meta-platforms.

## Platform Tree
The full "platform tree" for library, app, and test projects are shown below (compiler defines are given in brackets).

````
▌ all
├──▌ pack (references mobile)
└──▌ mobile
   ├──▌portable
   │  └──▌ dotnet (library)
   ├──▌android [ANDROID]
   │  ├──▌ monodroid (library)
   │  ├──▌ monodroid.app (app)
   │  └──▌ android.aut (automation)
   ├──▌ios [IOS]
   │  └──▌ ios.unified [IOS_UNIFIED]
   │  │  ├──▌ xamarin.ios (library)
   │  │  ├──▌ xamarin.ios.phone (app)
   │  │  └──▌ xamarin.ios.sim (app)
   │  ├──▌ ios.classic [IOS_CLASSIC]
   │  │  ├──▌ monotouch (library)
   │  │  ├──▌ monotouch.phone (app)
   │  │  └──▌ monotouch.sim (app)
   │  └──▌ ios.aut (automation)
   └──▌windows [WINDOWS]
      ├──▌ windows.universal [WINDOWS_UWP]
      │  ├──▌ uap (library)
      │  └──▌ uap.32 (app)
      ├──▌ windows.phone [WINDOWS_PHONE_APP]
      │  ├──▌ wpa (library)
      │  └──▌ wpa.32 (app)
      └──▌ windows.tablet [WINDOWS_APP]
         ├──▌ win (library)
         ├──▌ win.32 (app)
         ├──▌ win.64 (app)
         └──▌ wpa.arm (app)
````

## Solution Configuration and Platforms
Creating a new Xamarin.Forms creates platforms in the solution file (`AnyCPU`, `ARM`, `x64`, `x86`, `IPhone`, `IPhoneSimulator`) which do not map directly to what we think of as _mobile_ platforms. The unified project system creates a set of meta-platforms that do map directly to mobile platforms and these are exposed in Visual Studio.

| Solution | Library | App | Test | 
| --- | --- | --- | --- |
| Android | monotouch | monotouch.app | android.aut |
| iOS | monodroid | monodroid.phone | android.aut |
| iOS (sim) | monodroid | monodroid.sim | ios.aut |
| iOS (phone) | monodroid | monodroid.phone | ios.aut |
| iOS Unified (sim) | xamarin.ios | xamarin.ios.sim | ios.aut |
| iOS Unified (phone) | xamarin.ios | xamarin.ios.phone | ios.aut |
| Win Phone | wpa | wpa.32 | |
| Win Universal | uap | uap.32 | |
| Win Tablet (x32) | win | win.32 | |
| Win Tablet (x64) | win | win.64 | |
| Win Tablet (arm) | win | win.arm | |


## Unified Xamarin.Forms Project
The unified Xamarin.Forms library and app projects contain a folder for each platform the contents of which will only be compiled (and have correct intellisense) when the corresponding platform is specified. 

Debugging support is gently hacked in with the addition of CarouselView.App.[Platform] C# projects. These projects could, should, and hopefully will, be merged into the unified app project via the creation of a VSIP plugin. 

As viewed from the Visual Studio Solution Explorer the unified project for CarouselView is the following:

````
▌ CarouselView
├──▌ CarouselView.csproj
│  ├──▌ Properties
│  │  └─ AssemblyVersion.cs - Contains AssemblyVersion attribute
│  │     └─ AssemblyVersion.t.cs - Template expaded with msbuild variables; build fails if expansion doesn't match
│  ├──▌ Portable
│  │  └─ AssemblyInfo.cs - Typical AssemblyInfo.cs minus AssemblyVersion attribute
│  ├──▌ Android [ANDROID]
│  ├──▌ iOS [IOS]
│  └──▌ Windows [WINDOWS]
│
├──▌ CarouselView.App.csproj
│  ├──▌ Properties
│  │  └─ AssemblyVersion.cs
│  │     └─ AssemblyVersion.t.cs
│  ├──▌ Portable
│  │  └─ AssemblyInfo.cs
│  ├──▌ Android [ANDROID]
│  ├──▌ iOS [IOS]
│  ├──▌ Windows [WINDOWS]
│  │  ├──▌ Phone [WINDOWS_PHONE_APP]
│  │  ├──▌ Tablet [WINDOWS_APP]
│  │  └──▌ Universal [WINDOWS_UWP]
│  ├──▌ Resources (Xamarin Android project insists, for the moment, it's resource directory live at the project root)
│  ├─ Info.plist (Xamarin iOS project insists, for the moment, they live at the project root)
│  └─ Entitlements.plist (Xamarin iOS project insists, for the moment, they live at the project root)
│
├──▌ debugger - make one of the following projects current to enable debugging of CarouselView.App
│  ├──▌ CarouselView.App.Android
│  ├──▌ CarouselView.App.iOS
│  ├──▌ CarouselView.App.Windows.UAP
│  ├──▌ CarouselView.App.Windows.Phone
│  └──▌ CarouselView.App.Windows.Tablet
│
└──▌ CarouselView.Test
````

# General Build System

## Shell
```
build - build both release and debug
rbuild - build release
dbuild - build debug
clean - clean both release and debug
rclean - clean release
dclean - clean debug
```

## Directories
The following directories are well known to the build system and shell. Navigate to most well-known directories by typing its name into the shell. Type `r` to navigate to the root directory.

### Enlistment
The following well-known directories are under source control. Build artifacts are placed outside of these directories which allows for a vastly simplified [.gitignore](.gitignore) file.
````
▌ root of enlistment
├──▌ src – files that contribute to build output
├──▌ ext - files that extend msbuild
├──▌ env - files that initialize the shell; misc .bat files
└──▌ doc - files that document the system
````

### Artifacts
The following well-known directories are generated by the build and are not under source control.
````
▌ root of enlistment
├──▌ bld - output of build
│  ├──▌ bin - resultant binaries of build
│  └──▌ obj - temporary build files
├──▌ dls - downloads; cache of files needed before going offline
│  ├──▌ packages - nuget packages
│  └──▌ tools - misc tools; nuget.exe
└──▌ drp - archive of builds kicked off from root
````

## Sub-Directories
The following sub-directories of well-known directories have structure specific to their function worth documenting. Generally, sub-directories of well-known directories do not have shell aliases. 

### bld
Build output is kept separate from source code; Build output generated by a project in sub-directory of `src` is placed in the same sub-directory in `bld\bin` and `bld\obj`. For example, build output from building `src\carouselView\lib\CarouselView.csproj` is placed in `bld\bin\carouselView\lib` and `bld\obj\carouselView\lib`. 

If projects are built using the shim (see [shim](#shim)) then log files of all verbosities (summary, normal, detailed, and diag) and severities (warning, and error) for all sub-builds (e.g. Shim.Build, Core.NugetRestore, Core.Build) are also created.

If the build is identifiable (see [drp](drp)) then metadata identifying the build (number, revision, branch, url) will be stored in `BuildInfo.*` files under `bin\bld` and `BuildInfo.cs` under `bld\obj\$(Configuration)`.
````
▌ bld
└──▌ bin
│  ├─ BuildInfo.Build.Number - computed by adding one to the last sub-directory of drp\number\
│  ├─ BuildInfo.Enlistment.revision - computed by consulting source control (git)
│  ├─ BuildInfo.Enlistment.status - capture the state of the enlistment; publication fails if dirty
│  ├─ Shim.Build.*.log - log of shim launching other build processes
│  ├─ Core.NugetRestore.*.log - log of nuget restore which happens before launching core build
│  └──▌ debug or release
│     ├─ Core.Build.*.log - compiler errors will appear in here
│     └──▌ [relative directory from src]
└──▌ obj
   └──▌ debug or release
      └──▌ [relative directory from src]
         └─ BuildInfo.cs - generated by shim, replaces universally included src\BuildInfo.cs during build
````

### drp
Builds are archived so they may be distributed via file servers. Among other things, having an archive of builds simplifies bisecting a regression and debugging intermittent build failures. For a build to be archived it must have been assigned an identity and that identity should be burned into each assembly published. Build identities are assigned by the shim (see [shim](#shim)) to builds of clean enlistments kicked off from the root.

The following three identities are assigned to a build:

| Assigned By | Identity | Example |
| --- | --- | --- | --- |
| File system | Number | Next folder in `drp\number` (see [BuildInfo.cs](src/BuildInfo.cs)) |
| Source Control | Revision, Branch, Url | Git hash, branch, url (see [BuildInfo.cs](src/BuildInfo.cs)) |
| Person | Version | `BuildVersion` declared in [src\.props](src/.props) (see [AssemblyVersion.cs][3]) |

After build completion, identifiable builds will have their `bin\bld` directory copied to `drp\number\$(BuildNumber)` and a simlink to that directory is created at `drp\revision\$(EnlistmentRevision)` and created (or overridden) at `drp\version\$(BuildVersion)`. 
````
▌ drp
├──▌ number - archived builds by build number
├──▌ revision - archived builds by source control revision  (symlinks into drp\number)
└──▌ version - archived builds by version (symlinks into drp\number)
````

## Projects
Project files have been modified to enable build features that simplify maintaining a CI infrastructure at the expense of tooling. The main tooling break are design time features of Visual Studio which modify project files because, without VSIP integration, Visual Studio in unaware of the new conventions. For example, adding a new file Android specific file via Visual Studio to [`CarouselView.csproj`][2] will require moving the `<Compile Include="NewAndroidFile.cs">` element to live under `<ItemGroup Condition=" '$(MobilePlatform)' == '$(AndroidMobilePlatformId)'" >`. 

Manual edits of project files are more easily made after installing [EditProj][1]. For more extensive edits, unload all projects under `src` and open project files from the shared project [`.repo`](.repo.shproj). 

The `.repo` project includes all msbuild files. This allows for global search and replace of msbuild symbols. The shell alias `ts` touches the solution file which has the effect of reloading changes made to msbuild files which are included by project files which are otherwise cashed once at startup and never refreshed.

### General Project Template
Project files all conform to the following general template sections of which are described in subsequent sections.

```xml
<Project>
  <PropertyGroup>
    <!--Properties that identify the type of the project-->
  </PropertyGroup>
  
  <!--Load properties that are a function of the type of the project and platform-->
  <Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildProjectDirectory)\, .pre.props))\.pre.props" />

  <!--Load common properties for the given type of the project and platform-->
  <Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), .props))\.props" />

  <PropertyGroup Condition="'$(SomeTypeProperty)'=='SomeTypeId'">
    <!--Set properties specific to this project and\or platform; the goal is to have as few such properties as possible-->
  </PropertyGroup>
  <ItemGroup Condition="'$(SomeTypeProperty)'=='SomeTypeId'">
    <!--Project Files-->
    <!--Project References-->
  </ItemGroup>
  
  <Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), .targets))\.targets" />
</Project>
```

### Common Properties (.props)
Typically, solutions with multiple projects suffer from duplication of project settings. For example, to enable warnings as errors typically requires modifying each project. To prevent duplication and allow settings to be centrally administered common project settings are extracted to `.props` files in parent directories. For example, `WarningLevel` and `TreatWarningsAsErrors` have been extracted to [`src\.props`](src/.props) and so are included by all projects in any sub-directory of `src`. 

Those `.props` files "closest" to the project override those files further away. For example, the [`.props`](.props) file in root directory is included before any other simultaneously making its definitions available to those files and allowing them to override those settings. For example, the `.props` files processed when loading [`CarouselView.csproj`][2] are, [`\.props`](.props), then [`src\.props`](src/.props), then [`src\carouselView\.props`](src/carouselview/.props), then finally the project itself. Note that they are included in the reverse order but, because the first thing they each do is import their parent `.props` file they are logically processed in the reverse of the include order. Don't think too hard about it; It works as you'd expect.

Some settings gathered into `.props` files are common to all projects (e.g. `WarningLevel` and `TreatWarningsAsErrors`) however most are common to a specific _type_ of platform or project. For example, [`src/.props'](src/.props) (where most of settings extracted from various types of projects end up) contains the following section describing debug settings:
````
  <!--debug-->
  <PropertyGroup Condition="'$(Configuration)'=='debug'">
    <DebugSymbols>true</DebugSymbols>
    <DebugType>full</DebugType>
    <Optimize>false</Optimize>
    <DefineConstants>$(DefineConstants)TRACE;DEBUG;</DefineConstants>
  </PropertyGroup>
````
In general, such sections take the following form:
````
  <!--common setting for type-->
  <PropertyGroup Condition="'$(Type)'=='TypeId'">
    <TypeSetting>setting</TypeSetting>
  </PropertyGroup>
````
Before a `.props` file can determine the type of project loaded, and so what settings are appropriate, properties describing the type of the project must be loaded. How type information about a project is populated is the subject of the next section.

### Project and Platform Types (.pre.props)
The first property defined by all projects is `<MetaProject>` which in conjunction with the `Configuration`, `Platform`, and `MetaPlatform` global properties (passed via the command line or through the `msbuild` task), identifies the _type_ of the project. With the type established, a hierarchy of `.pre.props` files (which are loaded before `.props` files) are able to populate properties which fully describe all aspects of the type of the project being loaded and which are documented below.

#### MetaProject
A `MetaProject` is an amalgam of projects. For example, all the Xamarin.Forms library projects (e.g. Android, iOS, and Windows) combine to form the `xf.lib` `MetaProject`, Xamarin.Forms app projects form `xf.app`, and Calabash Android and iOS UI automation projects form `xf.aut`. One of the boolean properties `IsMobileLibraryProject`, `IsMobileAppProject`, or `IsMobileTestProject` is set to `true` depending on the type of meta-project being loaded.

#### MetaPlatform
`MetaPlatform` is the heart of the type system; The `MetaPlatform` abstraction allows grafting a new platform lexicon over of the existing desktop lexicon. For example, in the case of Xamarin.Forms, `MetaPlatform` makes it possible to talk about `mobile`, `android`, or `win.arm` platforms instead of `AnyCPU`, `x32`, or `x64` platforms. 

Each `MetaProject` supports a set of `MetaPlatforms`. For example, here are the relationships for Xamarin.Forms:

| MetaPlatform | MetaProject |
| --- | --- |
| xf.aut | android.aut, ios.aut |
| xf.lib | dotnet, monodroid, monotouch, xamarin.ios, win, uap, wpa |
| xf.app | monodroid.app, monotouch.phone, monotouch.sim, xamarin.ios.sim, xamarin.ios.phone, win.32, win.64, win.arm, uap.32, wpa.32 |

A `MetaPlatform` will have a `MetaPlatformType` of either `group`, `meta`, or `leaf`. Depending on the `MetaPlatformType`, either `IsGroupPlatform`, `IsMetaPlatform` or `IsLeafPlatform` will be set to true.
- A `group` `MetaPlatform` is a collection of one or more `group` or `meta` `MetaPlatforms`. Groups can be named (e.g. `Mobile` = { `portable`, `android`, `ios`, `windows` }) or specified ad-hoc on the command line (e.g. `/p:MetaPlatform=android;windows`). The children of the group are stored in `ChildMetaPlatforms`. 
- A `meta` `MetaPlatform` is a platform in new lexicon which is being grafted over a desktop platform (e.g. `monodroid` over `AnyCpu` or `monotouch.app.sim` over `IPhoneSimulator`). It has one `leaf` `MetaPlatform` child which is stored in `LeafPlatform`.
- A `leaf` `MetaPlatform` is a desktop platform (e.g. `AnyCPU`) augmented with a `MetaPlatform` (e.g. `android`) and `MetaProject` (e.g. `android.lib`) and other properties describing the project type (e.g. `MobilePlatform`==`Android`). These agumented properties are use in `.csproj` and `.props` files (e.g.  [`CarouselView.csproj`][2] and [`src/.props`](src/.props)) to set the properties (e.g. `AndroidSupportedAbis`) of one of the projects composing the unified project (e.g. Xamarin.Android). Once the properties of the composed project are set its msbuild targets are invoked to finally preform the build.

Here is a summary of the relationships between `MetaProjects`, `MetaPlatforms`, and `MetaPlatformTypes` for Xamarin.Forms. These relationships are declared in [ext\xf\xf.pre.props](ext/xf/xf.pre.props).

![Platform Image](doc/Platforms.gif)

## Building

### Shim
The [shim](ext\shim\shim.proj) is a project which searches for and then builds .csproj files after, among other things, configuring logging. 

### MetaPlatform
To build a `MetaPlatform` specify it at the command like just like a `Platform`. For example, build the `android` Xamarin.Forms `MetaPlatform` like this:

    src\carouselView\lib> msbuild /p:MetaPlatform=android
    
In order to support Visual Studio, a `MetaPlatform` can also be passed as a `Platform` like this:

    src\carouselView\lib> msbuild /p:Platform=android

and by issuing `/t:dryRun` commands from the shell. For example, here is the [output](doc/dryRun.md) produced by the following command:

    src\carouselView\lib> msbuild /v:m /p:platform=all /t:dryRun

For even more information about project reference resolution pass `/p:verbosity=high` along with any target.


[1]: https://visualstudiogallery.msdn.microsoft.com/b346d9de-8722-4b0e-b50e-9ae9add9fca8
[2]: src/carouselView/lib/CarouselView.csproj
[3]: src/carouselView/lib/Properties/AssemblyVersion.cs
