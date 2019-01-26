# CppWithNuGetPackageReference
Example Cpp project that uses NuGet PackageReference (instead of packages.config)

## Why PackageReference instead of packages.config?

VisualStudio (for C++ vcxproj) only supports packages.config by default.
However MSBuild NuGet integration is done in a generic way, it is just that the VS NuGet integration only supports packages.config.
TODO: add links to official docs for comparison + CentralPackageVersions

## How
Example using https://github.com/AArnott/Nerdbank.GitVersioning 

### Directory.Build.props
  We store MSBuild settings in the solution root folder file Directory.Build.props

### Add NuGet PackageReference

  ```
  <ItemGroup>
    <PackageReference Include="Nerdbank.GitVersioning" Version="2.3.38"/>
  </ItemGroup>
  ```

### Run MSBuild Restore
  `msbuild /t:restore CppWithNuGetPackageReference.sln`

### Add _NuGetTargetFallbackMoniker
  It seems adding a PackageReference is not enough.
  Because during the build we get an error `Your project does not reference ".NETFramework,Version=v4.0" framework.`

  From normal (packages.config) based NuGet package installation, we determine that NuGet is `targeting 'native,Version=v0.0'`.

  And by looking at `C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\MSBuild\Microsoft\NuGet\15.0\Microsoft.NuGet.targets(186,5)`
  we can understand how a possible solution might look:

  ```
  ... 
  <ResolveNuGetPackageAssets AllowFallbackOnTargetSelection="$(DesignTimeBuild)"
                             ContinueOnError="$(ContinueOnError)"
                             IncludeFrameworkReferences="$(IncludeFrameworkReferencesFromNuGet)"
                             NuGetPackagesDirectory="$(NuGetPackagesDirectory)"
                             RuntimeIdentifier="$(NuGetRuntimeIdentifier)"
                             ProjectLanguage="$(Language)"
                             ProjectLockFile="$(ProjectLockFile)"
                             ContentPreprocessorValues="@(NuGetPreprocessorValue)"
                             ContentPreprocessorOutputDirectory="$(IntermediateOutputPath)\NuGet"
                             TargetMonikers="$(NuGetTargetMoniker);$(_NuGetTargetFallbackMoniker)">
  ...
  ```
 
   It seems there is a `$(_NuGetTargetFallbackMoniker)` that is passed as additional option to the `TargetMonikers` parameter.
   Although there is no official documentation on this, it looks promising since it is seems to be used as a possible workaround for 'UAP' projects.

  ```
  <PropertyGroup>
    <_NuGetTargetFallbackMoniker>$(_NuGetTargetFallbackMoniker);native,Version=v0.0</_NuGetTargetFallbackMoniker>
  </PropertyGroup>
  ```


## Technical Details
TODO: rewrite text

* VS UI "Restore NuGet Packages" does not restore based on PackageReference

  `All packages are already installed and there is nothing to restore.` 

  However, `MSBuild /t:restore` on solution/project does actually restore:
  ```
    D:\GitHub\CppWithNuGetPackageReference>msbuild /t:restore CppWithNuGetPackageReference.sln
    Microsoft (R) Build Engine version 15.9.21+g9802d43bc3 for .NET Framework
    Copyright (C) Microsoft Corporation. All rights reserved.

    Building the projects in this solution one at a time. To enable parallel build, please add the "/m" switch.
    Build started 26-1-2019 21:08:42.
    Project "D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference.sln" on node 1 (Restore target(s)).
    ValidateSolutionConfiguration:
      Building solution configuration "Debug|x64".
    Restore:
      Restoring packages for D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference\CppWithNuGetPackageReference.vcxproj...
      Committing restore...
      Generating MSBuild file D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference\obj\CppWithNuGetPackageReference.vcxproj.nuget.g.props.
      Generating MSBuild file D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference\obj\CppWithNuGetPackageReference.vcxproj.nuget.g.targets.
      Writing lock file to disk. Path: D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference\obj\project.assets.json
      Restore completed in 390,42 ms for D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference\CppWithNuGetPackageReference.vcxproj.

      NuGet Config files used:
          C:\Users\japj\AppData\Roaming\NuGet\NuGet.Config
          C:\Program Files (x86)\NuGet\Config\Microsoft.VisualStudio.Offline.config

      Feeds used:
          https://api.nuget.org/v3/index.json
          C:\Program Files (x86)\Microsoft SDKs\NuGetPackages\
    Done Building Project "D:\GitHub\CppWithNuGetPackageReference\CppWithNuGetPackageReference.sln" (Restore target(s)).


    Build succeeded.
        0 Warning(s)
        0 Error(s)

    Time Elapsed 00:00:01.13
  ```



* VS Build with PackageReference restored package fails with `Your project does not reference ".NETFramework,Version=v4.0" framework.`
  
  ```
    1>------ Rebuild All started: Project: CppWithNuGetPackageReference, Configuration: Debug x64 ------
    1>Build started 26-1-2019 21:10:19.
    1>Target GetBuildVersion:
    1>  Building version 0.0.2.18725 from commit 25492f4145c10ea960ec419ae9243811f20a148e
    1>Target GenerateNativeVersionInfo:
    1>  Copying file from "x64\Debug\\CppWithNuGetPackageReference.Version.rc.new" to "x64\Debug\\CppWithNuGetPackageReference.Version.rc".
    1>Target PrepareForBuild:
    1>  Creating directory "D:\GitHub\CppWithNuGetPackageReference\x64\Debug\".
    1>  Creating directory "x64\Debug\CppWithN.F88D9F1D.tlog\".
    1>Target ResolveNuGetPackageAssets:
    1>  C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\MSBuild\Microsoft\NuGet\15.0\Microsoft.NuGet.targets(186,5): error : Your project does not reference ".NETFramework,Version=v4.0" framework. Add a reference to ".NETFramework,Version=v4.0" in the "TargetFrameworks" property of your project file and then re-run NuGet restore.
    1>Done building target "ResolveNuGetPackageAssets" in project "CppWithNuGetPackageReference.vcxproj" -- FAILED.
    1>
    1>Done building project "CppWithNuGetPackageReference.vcxproj" -- FAILED.
    1>
    1>Build FAILED.
    1>
    1>C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\MSBuild\Microsoft\NuGet\15.0\Microsoft.NuGet.targets(186,5): error : Your project does not reference ".NETFramework,Version=v4.0" framework. Add a reference to ".NETFramework,Version=v4.0" in the "TargetFrameworks" property of your project file and then re-run NuGet restore.
    1>    0 Warning(s)
    1>    1 Error(s)
    1>
    1>Time Elapsed 00:00:00.26
    ========== Rebuild All: 0 succeeded, 1 failed, 0 skipped ==========
  ```

* Installing a NuGet package through the normal UI (with packages.config) results in:
  ```
    Attempting to gather dependency information for package 'Nerdbank.GitVersioning.2.3.38' with respect to project 'ConsoleApplication1', targeting 'native,Version=v0.0'
    Gathering dependency information took 82,39 ms
    Attempting to resolve dependencies for package 'Nerdbank.GitVersioning.2.3.38' with DependencyBehavior 'Lowest'
    Resolving dependency information took 0 ms
    Resolving actions to install package 'Nerdbank.GitVersioning.2.3.38'
    Resolved actions to install package 'Nerdbank.GitVersioning.2.3.38'
    Retrieving package 'Nerdbank.GitVersioning 2.3.38' from 'nuget.org'.
    Adding package 'Nerdbank.GitVersioning.2.3.38' to folder 'D:\Temp\ConsoleApplication1\packages'
    Added package 'Nerdbank.GitVersioning.2.3.38' to folder 'D:\Temp\ConsoleApplication1\packages'
    Added package 'Nerdbank.GitVersioning.2.3.38' to 'packages.config'
    Executing script file 'D:\Temp\ConsoleApplication1\packages\Nerdbank.GitVersioning.2.3.38\tools\Install.ps1'...
    Could not find an AssemblyInfo.cs file at 'D:\Temp\ConsoleApplication1\ConsoleApplication1\Properties\AssemblyInfo.cs'. Skipping version.json generation.
    Successfully installed 'Nerdbank.GitVersioning 2.3.38' to ConsoleApplication1
    Executing nuget actions took 2,83 sec
    Time Elapsed: 00:00:03.1618944
    ========== Finished ==========
  ```

