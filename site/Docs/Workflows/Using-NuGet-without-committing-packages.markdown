﻿# Update as of NuGet 2.7

**With NuGet 2.7, a new Automatic Package Restore approach has been introduced. For more information about
that approach as well as a new NuGet.exe Restore Command, see the [Package Restore reference document](/docs/reference/package-restore).**

# Using NuGet without committing packages to source control

The original NuGet workflow has been to commit the Packages folder into source control. The 
reasoning is that it matches what developers typically do when they don't have NuGet: they create a 
`Lib` or `ExternalDependencies` folder, dump binaries into there and commit them to source control 
to allow others to build.

While this has worked fine for some users, we have also heard from many that committing packages 
into source control is not what they want to do. When using a DVCS like Mercurial or Git, committing 
binaries can grow the repository size like crazy over time, making cloning more and more painful. In 
fact, this has been one of the top requests on NuGet our issue tracker.

The good news is that NuGet now offers a workflow which goes a long way to solving this problem, and is
really easy to set up. Here is the way to do it:

## Enabling Package Restore During Build

Beginning with NuGet 2.0, [restoring packages during build requires explicit consent from the
user](http://blog.nuget.org/20120518/package-restore-and-consent.html). This must be done on
each machine that builds the project.

In Visual Studio, enable "Allow NuGet to download missing packages during build". This setting lives
under **Options -> Package Manager -> General**.

![Allow NuGet to download missing packages setting](images/allow-package-restore-configuration.png)

**To enable package restore for build servers without Visual Studio installed, you can also set the
environment variable `EnableNuGetPackageRestore` to "true".**

## Project Setup
Let’s assume that you have a solution that is either already using NuGet, or planning to use it, and that
you want to set up the no-commit workflow.

Right click on the _Solution_ node in Solution Explorer and select _Enable NuGet Package Restore_.

![Enable NuGet Package Restore Context Menu item](images/enable-package-restore.png)

**That's it!** You're all set.

## Details
So what exactly did that do? It added a solution folder named .nuget containing NuGet.exe and a NuGet.targets MsBuild file. More specifically, it downloaded and extracted two NuGet packages: [NuGet.Commandline](http://nuget.org/packages/nuget.commandline) for NuGet.exe and [NuGet.Build](http://nuget.org/packages/nuget.build) for NuGet.targets. It also changed every project in the solution to import the NuGet.targets MsBuild task. 

![New Solution folder with package restore files](images/package-restore-solution.png)

Finally, it added a NuGet.config file with the following XML:

	<configuration>
	  <solution>
	    <add key="disableSourceControlIntegration" value="true" />
	  </solution>
	</configuration>

The disableSourceControlIntegration setting instructs version control systems like TFS to not add the NuGet packages folder to the pending check-ins list.

With this in place, any time a project is compiled, the build task will look at each project's packages.config file and for each package listed, ensure that the corresponding package exists within the packages folder. For any missing package, the build task will download and unpack the package.

In this scenario, NuGet will grab the exact version when restoring a package. It will not perform any upgrades.

<p class="caution"><b>Be sure to check in your _repositories.config_ file.</b> Some version control systems, 
such as Subversion (a.k.a. SVN), may require additional configuration to allow you to selectively ignore an arbitrary set of files in a path, _but_ include a specific file, such as your _repositories.config_ in that path.
<br>
For SVN specifically, you may use SVN:IGNORE to prevent paths with certain patterns from being committed (and making a mess in your pending changes views).
By adding the following pattern as an SVN:IGNORE to your "packages" directory, the repositories.config will be committed, but nuget package directories will be ignored.
<br>
<br>
*[0-9]*
<br>
<br>
Similar ignore patterns are probably available for most source control systems.
<br>
<br>
In rare cases, you'll need to agree with your team to include the folder from the root of your solution, explicitly include the _repositories.config_ file but _not_ check in any packages. Without 
the file you may experience problems loading or building a multi-project solution with package restore enabled.
</p>

## Mono
On mono you can run `xbuild` on the project file or on your solution and it should successfully 
restore packages for any project that has package restore enabled.
