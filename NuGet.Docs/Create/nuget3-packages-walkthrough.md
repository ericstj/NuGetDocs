# NuGet 3 Package Authoring #


## .NET library ##
### pure portable on dotnet ###
This is the simplest type of package.  You wish to ship a portable class library in package that you intend to be consumed only in projects supporting nuget v3 and project.json.

```
<depenedencies>
  <group targetFramework="dotnet">
    <dependency id="System.Collections" version="4.0.0"/>
    <dependency id="System.Globalization" version="4.0.0"/>
    <dependency id="System.Linq" version="4.0.0"/>
    <dependency id="System.Resources.ResourceManager" version="4.0.0"/>
    <dependency id="System.Runtime" version="4.0.0"/>
    <dependency id="System.Runtime.Extensions" version="4.0.0"/>
    <dependency id="System.Threading" version="4.0.0"/>
  </group>
</depenedencies>

lib/dotnet/System.Banana.dll

```

### supporting nuget v2 and project types that don't yet support project.json ###
NuGet v2 and packages.config in Nuget v3 will promote any indirect references to direct references.  This creates a problem for the example we have so far since the dependencies of the library cannot always be updated on platforms like .NET desktop or Windows Phone where the packages just represent the surface area in the platform.

To avoid this problem we can add empty dependency groups for all of the platforms where our dependencies are included in box.  This should **not** be done for platforms that rely on packages for implementation (like DNXCore, NETCore version >= 5.0, and UAP).  This should also **not** be done if the version of the dependency is higher than the one inbox.
```
<depenedencies>
  <group targetFramework="dotnet">
    <dependency id="System.Collections" version="4.0.0"/>
    <dependency id="System.Globalization" version="4.0.0"/>
    <dependency id="System.Linq" version="4.0.0"/>
    <dependency id="System.Resources.ResourceManager" version="4.0.0"/>
    <dependency id="System.Runtime" version="4.0.0"/>
    <dependency id="System.Runtime.Extensions" version="4.0.0"/>
    <dependency id="System.Threading" version="4.0.0"/>
  </group>
  <!-- all depdencies are inbox in netcore45 / win8 -->
  <group targetFramework="netcore45"/>
  <!-- starting with netcore 50 dependencies come from nuget packages -->
  <group targetFramework="netcore50">
    <dependency id="System.Collections" version="4.0.0"/>
    <dependency id="System.Globalization" version="4.0.0"/>
    <dependency id="System.Linq" version="4.0.0"/>
    <dependency id="System.Resources.ResourceManager" version="4.0.0"/>
    <dependency id="System.Runtime" version="4.0.0"/>
    <dependency id="System.Runtime.Extensions" version="4.0.0"/>
    <dependency id="System.Threading" version="4.0.0"/>
  </group>
  <!-- more inbox frameworks -->
  <group targetFramework="net45"/>
  <group targetFramework="wp8"/>
  <group targetFramework="wpa81"/>
</depenedencies>

lib/dotnet/System.Banana.dll
```


The .NET and NuGet team hope to eliminate the need to do this in future releases.

### native library ###
Some libraries can benefit from using a native component within their package.  This can help to provide a performance optimized native implementation of an algorithm or directly consume some other native SDK (eg: statically link it or compile against its headers).

Add native runtime entries.
```
lib/dotnet/System.Banana.dll
runtimes/win7-x86/native/cherry.dll
runtimes/win7-x64/native/cherry.dll
runtimes/win10-arm/native/cherry.dll
```
Note that we still use `lib` for the managed assembly.  This is behaving as both the compile time and runtime asset for the managed assembly.  The presense of `runtime/{rid}/native` folders does not replace lib.

The .NET assembly is likely now runtime specific since it depends on the runtime specific native DLL, which will at least have a different extension on other runtimes (eg: .dll vs .dylib or .so) so we should also modify the path to the implementation assembly.

TODO: tooling to produce reference assembly.
```
ref/dotnet/System.Banana.dll
runtimes/win7/lib/dotnet/System.Banana.dll
runtimes/win7-x86/native/cherry.dll
runtimes/win7-x64/native/cherry.dll
runtimes/win10-arm/native/cherry.dll
```
Note that we now need a `ref` folder.  This is because the `runtimes/{rid}/lib/dotnet` folder replaces `lib/dotnet` since it is more specific.  Since the `runtimes` folder is only used at runtime we need a seperate definition of the compile time asset.

This will give folks the right error when they try to install the package on a new runtime (eg: Linux) instead of permitting the package to install and failing to find the DLL at runtime.

### architecture specific implementation ###
It may be necessary to include architecture specific managed code in your package.  If this is the case the implementation assembly will need to be placed in architecture specific folders.

These folders are only used for the runtime asset from your package, so we'll need to provide a compile time asset.  For that we use `ref`.  This has the added benefit of making your architecture specific library consumable by AnyCPU projects.

TODO: tooling to produce reference assembly.
```
ref/dotnet/System.Banana.dll
runtimes/win7-x86/lib/dotnet/System.Banana.dll
runtimes/win7-x64/lib/dotnet/System.Banana.dll
runtimes/win10-arm/native/System.Banana.dll
```

### cross compiling for additional platforms ###

## WinMD library  ##
### with managed implementation ###
### with native implementation ###
### with resources ###
### supporting c++ ###
### supporting javascript ###
cannot include managed dll in packages that intend to support JS because JS does not handle dll references.

