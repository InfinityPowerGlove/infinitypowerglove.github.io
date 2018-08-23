---
layout: post
title:  "Debugging Cake Build Scripts"
author: "Jeff-Stapleton"
twitter: "jeffdstapleton"
date:   2018-08-23 15:04:00 -0600
tags: [cakebuild, cake, debugging]
---
This is going to be a quick post. I recently had some issues with a build.cake script and wanted to write a quick guide on how to debug them. 

1. Open a powershell window and navigate to the `build.cake` script.
2. From there, run the cake executable relative to where the build script is, for example:
```ps
 .\tools\Cake\Cake.exe --debug
```
3. The process id will be printed in the terminal for convenience.

![](2018-08-17-debugging-cake-build-scripts-id.png)

4. As far as I know it is not possible to set the target in debug mode. So before we go any further we need to make sure the default target is configured correctly.
```csharp
RunTarget("Default");

Task("Default")
    .IsDependentOn("Verify")
    .IsDependentOn("Package")
    .IsDependentOn("Deploy")
```
5. Now open your debugger of choice, in this example I will be using Visual Studio 2017.
6. In Visual Studio open your `build.cake` file.
7. Click "Attach...".

![](2018-08-17-debugging-cake-build-scripts-attach.png)

8. Select the "Cake.exe" process, reference the process id from step 3 if you can't find the process.

![](2018-08-17-debugging-cake-build-scripts-process.png) 

9. Once the debugger is attached you can add breakpoints and step through the script.

** Just a note that intellisense won't work so you will have to use the immediate and local window to inspect variables.

I hope this quick tutorial was useful. Good luck and happy coding!