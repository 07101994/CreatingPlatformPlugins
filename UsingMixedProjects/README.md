# Using Mixed Projects

Some libraries have some shared code, but have more members that are 
platform-specific. There may even be complex implmentation details that are
only relevant to a specific platform.

Instead of large methods and files with numerous preprocessor directives, the
implementation details can be moved to the platform projects, and just the 
relevant/shared code in the shared project. Some of the smaller members could 
still use preprocessor directives, but this could also be moved out into 
platform projects.

## The Solution

Using a mixture of shared code and platform code, there will be 3 types of
projects:

 - one **shared project** _(for the code)_
 - one **.NET Standard project** _(for the bait)_
 - several **platform projects** - one per desired platform _(for the switch)_

Since the code is distributed across projects, more effort should be taken to 
ensure that the assemblies have matching APIs. The .NET Standard assembly 
should have the same, or a subset, of the APIs exposed in the platform 
assemblies. This is to avoid a runtime exception if a member is used from a 
shared project, but then missing from the platform assembly.

The important aspect is that only the public .NET Standard APIs need to exist
in the platform assemblies - not too important the other way around.

### Example Solution

An example scenario would be 5 projects:

 - one **shared project**
 - one **.NET Standard project** targeting `netstandard1.0`
 - an **Android project**
 - an **iOS project**
 - a Windows 10 **UWP project**

### Example Code

#### Shared Project

There are several ways to create the shared project (if at all), the first of 
which is to make sure all platform-specific code exists in the platform 
libraries - including the code for the .NET Standard library:

```csharp
// a public class that is visible to any platform
public class Screen
{
    // a public method that is visible to any platform
    public double GetDensity()
    {
        // fall back to the platform libraries
        return ScreenImplementation.GetDensity();
    }
}
```

Another way might be to make sure the .NET Standard library doesn't have any 
code other than what comes in from the shared project. This will prevent the 
.NET Standard library from accidentally receiving APIs that do not exist in 
the platform libraries:

```csharp
// a public class that is visible to any platform
public static class Screen
{
    // a public method that is visible to any platform
    public static double GetDensity()
    {
#if NETSTANDARD1_0
        // code that will run on any other platform
        throw new PlatformNotSupportedException();
#else
        // fall back to the platform libraries
        return ScreenImplementation.GetDensity();
#endif
    }
}
```

#### Platform Projects

The platform projects can now just contain implementation details and not be
too concerned about the API as the result of the shared project being compiled
in.

**Android Project**

```csharp
// an internal type that is not visible to the public
internal static class ScreenImplementation
{
    public static double GetDensity()
    {
        var svc = Application.Context.GetSystemService(Context.WindowService);
        var wm = svc.JavaCast<IWindowManager>();
        var dm = new DisplayMetrics();
        wm.DefaultDisplay.GetMetrics(dm);
        return dm.Density;
    }
}
```

**iOS Project**

```csharp
// an internal type that is not visible to the public
internal static class ScreenImplementation
{
    public static double GetDensity()
    {
        return UIScreen.MainScreen.Scale;
    }
}
```

**UWP Project**

```csharp
// an internal type that is not visible to the public
internal static class ScreenImplementation
{
    public static double GetDensity()
    {
        var di = DisplayInformation.GetForCurrentView();
        return di.RawPixelsPerViewPixel;
    }
}
```

#### Consumer Project

The consumer app or library can then easily make use of the APIs:

```csharp
void DoScreenThings()
{
    double density = Screen.GetDensity();
}
```

### Example NuGet Package

There are many elements that can be used to customize the package, but only a
few are required to create a bait-and-switch package:

```xml
<?xml version="1.0"?>
<package>
  <metadata>
    <!-- the unique package ID -->
    <id>PackageId</id>
    <!-- the current version of the package -->
    <version>1.0.0</version>
    <!-- the authors text for the gallery -->
    <authors>FirstName LastName</authors>
    <!-- the description for the package manager UI -->
    <description>Package description here.</description>
  </metadata>
  <files>
    <!-- the platform-specific assembly for Android -->
    <file src="output/android/Library.dll" target="lib/MonoAndroid" />
    <!-- the platform-specific assembly for iOS -->
    <file src="output/ios/Library.dll" target="lib/Xamarin.iOS" />
    <!-- the platform-specific assembly for UWP -->
    <file src="output/uwp/Library.dll" target="lib/uap10.0" />

    <!-- the bait assembly -->
    <file src="output/netstandard/Library.dll" target="lib/netstandard" />
  </files>
</package>
```
