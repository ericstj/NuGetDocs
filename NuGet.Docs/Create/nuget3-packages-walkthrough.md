# NuGet 3 Package Authoring #

## 1 .NET library ##
### 1.1 Portable library targeting `dotnet` ###
This is the simplest type of package.  You wish to ship a portable class library in package that you intend to be consumed only in projects supporting nuget v3 and project.json.  The `dotnet` target moniker represents all platforms who's .NET surface area can be represented entirely by package dependencies.  For many projects using this target moniker will eliminate the need to ever list specific platforms in a package.

The below sample illustrates the dependencies of a project.json based PCL targeting .NET Framework 4.6, Windows Universal 10.0, and ASP.NET Core 5.0.  The dependency section can be generated using the third party [NuSpec.ReferenceGenerator](https://www.nuget.org/packages/NuSpec.ReferenceGenerator/) package.
```
<dependencies>
  <group targetFramework="dotnet">
    <dependency id="System.IO" version="4.0.10"/>
    <dependency id="System.Runtime" version="4.0.20"/>
  </group>
</dependencies>

lib/dotnet/System.Banana.dll
```
### 1.2 Support packages.config with `dotnet` ###
Now lets suppose that we want to target older platforms that don't yet support project.json.  The below example changes the PCL to target .NET Framework 4.5, Windows 8, Windows Phone 8.1 and Windows Phone Silverlight 8.

Nuget using packages.config will promote any indirect references to direct references.  Nuget will also offer upgrade of any packages directly referenced.  This creates a problem for the example we have so far since the dependencies of the library cannot always be updated on platforms like .NET desktop or Windows Phone where the packages just represent the surface area inbox in the platform.

To avoid this problem we can add empty dependency groups for all of the platforms where our dependencies are included in box.  This should **not** be done for platforms that rely on packages for implementation (Windows Universal 10.0 and ASP.NET Core 5.0).  This should only be done for platforms which you've specifically targeted in the project.

In order to also support installation into packages.config based portable projects NuGet requires `portable-` folder and dependency group.  The dependency group is needed for the same reason as above, the file is needed because NuGet does not treat any `portable-` targets as compatible with `dotnet`.
```
<dependencies>
  <group targetFramework="dotnet">
    <dependency id="System.IO" version="4.0.0"/>
    <dependency id="System.Runtime" version="4.0.0"/>
  </group>
  <!-- all depdencies are inbox in netcore45 / win8 -->
  <group targetFramework="netcore45"/>
  <!-- netcore50 dependencies from nuget packages -->
  <group targetFramework="netcore50">
    <dependency id="System.IO" version="4.0.0"/>
    <dependency id="System.Runtime" version="4.0.0"/>
  </group>
  <!-- more inbox frameworks -->
  <group targetFramework="net45"/>
  <group targetFramework="wp8"/>
  <group targetFramework="wpa81"/>
  <!-- required for pcl projects that don't use project.json with dotnet -->
  <group targetFramework="portable-netcore45+net45+wp8+wpa8"/>
</dependencies>

lib/portable-netcore45+net45+wp8+wpa8/System.Banana.dll
lib/dotnet/System.Banana.dll
```

The .NET and NuGet team hope to eliminate the need to create these additional dependency groups in future releases.

### 1.3 Native library ###
Some libraries can benefit from using a native component within their package.  This can help to provide a performance optimized native implementation of an algorithm or directly consume some other native SDK (eg: statically link it or compile against its headers).

To represent the native component we will use 'runtime/{rid}/native' folders.  The '{rid}' used during package restore is determined by the runtimes section of the project.json.  The '{rid}' used at package resolve time is determined by the NuGet client and should match one of the {rid}s in the project.json.  In the case of the MSBuild client in Visual Studio, this is based on the project's properties like Platform and TargetFramework.
```
lib/dotnet/System.Banana.dll
runtimes/win-x86/native/cherry.dll
runtimes/win-x64/native/cherry.dll
runtimes/win8-arm/native/cherry.dll
```
Note that we still use `lib` for the managed assembly.  This is behaving as both the compile time and runtime asset for the managed assembly.  The presense of `runtime/{rid}/native` folders does not replace lib.

The .NET assembly is likely now runtime specific since it depends on the runtime specific native DLL, which will at least have a different extension on other runtimes (eg: .dll vs .dylib or .so) so we should also modify the path to the implementation assembly.

TODO: tooling to produce reference assembly.
```
ref/dotnet/System.Banana.dll
runtimes/win/lib/dotnet/System.Banana.dll
runtimes/win-x86/native/cherry.dll
runtimes/win-x64/native/cherry.dll
runtimes/win8-arm/native/cherry.dll
```
Note that we now need a `ref` folder.  This is because the `runtimes/{rid}/lib/dotnet` folder replaces `lib/dotnet` since it is more specific.  Since the `runtimes` folder is only used at runtime we need a seperate definition of the compile time asset.

This will give folks the right error when they try to install the package on a new runtime (eg: Linux) instead of permitting the package to install and failing to find the DLL at runtime.

To help produce a reference assembly you can use the Microsoft.DotNet.BuildTools.GenAPI package.  TODO: sample.

### 1.4 Architecture specific .NET library ###
It may be necessary to include architecture specific managed code in your package.  If this is the case the implementation assembly will need to be placed in architecture specific folders.

These folders are only used for the runtime asset from your package, so we'll need to provide a compile time asset.  For that we use `ref`.  This has the added benefit of making your architecture specific library consumable by AnyCPU projects.

To help produce a reference assembly you can use the Microsoft.DotNet.BuildTools.GenAPI package.  TODO: sample.
```
ref/dotnet/System.Banana.dll
runtimes/win-x86/lib/dotnet/System.Banana.dll
runtimes/win-x64/lib/dotnet/System.Banana.dll
runtimes/win8-arm/lib/dotnet/System.Banana.dll
runtimes/win-x86/native/cherry.dll
runtimes/win-x64/native/cherry.dll
runtimes/win8-arm/native/cherry.dll
```

### 1.5 Support packages.config with `runtimes` ###
Both of the previous examples used the NuGet v3 project.json feature of `runtimes` to provide OS/architecture specific assets.  To do the same for packages.config based projects we'll need to approximate this feature using msbuild targets.  We'll add the targets to the 'build/dotnet' folder, matching the target moniker used by our assets and give it the same name as our package.
```
build/dotnet/System.Banana.targets
ref/dotnet/System.Banana.dll
runtimes/win-x86/lib/dotnet/System.Banana.dll
runtimes/win-x64/lib/dotnet/System.Banana.dll
runtimes/win8-arm/lib/dotnet/System.Banana.dll
runtimes/win-x86/native/cherry.dll
runtimes/win-x64/native/cherry.dll
runtimes/win8-arm/native/cherry.dll
```

```
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <!-- If the project.json file is missing assume a packages.config
       based project and add the NuGet v3 assets.  -->
  <ItemGroup Condition="!Exists('project.json') AND !Exists('$(MSBuildProjectName).project.json')">
    <Reference Include="$(MSBuildThisFileDirectory)..\..\ref\dotnet\*.dll">
      <Private>false</Private>
    </Reference>
    <ReferenceCopyLocalPaths Include="$(MSBuildThisFileDirectory)..\..\runtimes\win*-$(Platform)\lib\dotnet\*.*" />
    <ReferenceCopyLocalPaths Include="$(MSBuildThisFileDirectory)..\..\runtimes\win*-$(Platform)\native\*.*" />
  </ItemGroup>
</Project>

```
Note that this assumes only one `{rid}` per architecture.  If a package uses multiple `{rid}`s per architcture these targets will be more involved than just `win*`.

### 1.6 Cross compiling for additional platforms ###
Expanding on the previous example, suppose that a single build of a library cannot provide the full functionality on all platforms.  For example: you'd like to use the System.IO.MemoryStream.TryGetBuffer.  This API is available in System.IO version 4.0.10 which is supported on .NET Framework 4.6, Windows Universal 10.0, and ASP.NET Core 5.0.  The API can return the underlying byte array used by the MemoryStream without copying it.  On older platforms the API is not available but a similar API ToArray() can be used which does a copy of the byte array.

The package will contain two different PCLs.
Build A targeting .NET Framework 4.5, Windows 8, Windows Phone 8.1 and Windows Phone Silverlight 8, using MemoryStream.ToArray();
Build B targeting .NET Framework 4.6, Windows Universal 10.0, and ASP.NET Core 5.0, using MemoryStream.TryGetBuffer().

```
<dependencies>
  <group targetFramework="dotnet">
    <dependency id="System.IO" version="4.0.10"/>
    <dependency id="System.Runtime" version="4.0.0"/>
  </group>
  <!-- all depdencies are inbox in netcore45 / win8 -->
  <group targetFramework="netcore45"/>
  <!-- netcore50 / uap10 dependencies from nuget packages -->
  <group targetFramework="netcore50">
    <dependency id="System.IO" version="4.0.10"/>
    <dependency id="System.Runtime" version="4.0.0"/>
  </group>
  <!-- more inbox frameworks -->
  <group targetFramework="net45"/>
  <group targetFramework="wp8"/>
  <group targetFramework="wpa81"/>
  <group targetFramework="portable-netcore45+net45+wp8+wpa8"/>
</dependencies>

lib/portable-netcore45+net45+wp8+wpa8/System.Banana.dll (build A)
lib/netcore45/System.Banana.dll (build A, required since dotnet would take precedence over portable-*)
lib/netcore50/System.Banana.dll (build B, required since netcore45 would take precedence over dotnet)
lib/net45/System.Banana.dll (build A, required since dotnet would take precedence over portable-*)
lib/net46/System.Banana.dll (build B, required since net45 would take precedence over dotnet)
lib/wp8/System.Banana.dll (build A, required since dotnet would take precedence over portable-*)
lib/wpa8/System.Banana.dll (build A, required since dotnet would take precedence over portable-*)
lib/dotnet/System.Banana.dll (build B)
```

## 2 WinMD library  ##
### 2.1 .NET implementation ###
### 2.2 Native implementation ###
### 2.3 With resources ###
### 2.4 Supporting c++ ###
### 2.5 Supporting javascript ###
note: cannot include managed dll in packages that intend to support JS because JS does not handle dll references.

