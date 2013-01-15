---
layout: post
title: "Single-click deployment of WPF application to GitHub Pages"
date: 2013-01-15 02:03
comments: true
author: Floyd Pink
categories: [flickr downloadr, Technical, deployment, MSBuild, NAnt, git, bash, .NET, WPF Application, ClickOnce Deployment, automation, GitHub Pages]
---
## Summary

This post will try to detail:

* how the [NAntContrib Version task](http://nantcontrib.sourceforge.net/release/latest/help/tasks/version.html) is automating the version increment; and then building the [flickr downloadr](/../) WPF app
* how the app is being published (via NAnt and MSBuild), for deployment to the [custom-installation url](/../downloads/latest), which is being hosted at [GitHub Pages](http://pages.github.com/))
* how some shell scripting helps to automate the upload of this published application to the site

## Introduction
The [downloads](/../downloads/latest) page on flickrdownloadr.com has a link to the latest published version of the application listed there along with the current version number. For the first few times, this page was being updated manually. But because it's still very early phases of development, where we are going to be releasing quite a few times and also because [release early, release often](http://en.wikipedia.org/wiki/Release_early,_release_often) is a good philosophy to try and practice, the need to automate this app-deployment-process soon arose. 

Ideally, the publish process detailed here would be run from a CI environment, but that is not happening right now for a couple of reasons:

1. There are no free and hosted CI services _that_ runs the .NET version (v4.5) that flickrdownloadr is built on. Even though [Codebetter CI](http://teamcity.codebetter.com) has kindly agreed to use their service for [this project](http://teamcity.codebetter.com/project.html?projectId=project228&tab=projectOverview) and there were quite a few successful builds earlier, they do not support v4.5 yet.
2. The deploy shell script (that is explained below) relies on the SSH identity to authenticate to GitHub to be already established when it runs. I haven't figured out how else to handle the GitHub authentication (with write access) from a CI environment, without publicly storing the credentials in the repository.

The app is being published using the [ClickOnce deployment](http://msdn.microsoft.com/en-us/library/t71a733d(v=vs.110\).aspx) methodology that is built-in to the Visual Studio IDE. Publishing a WPF application using ClickOnce manually, from within the Visual Studio IDE is a pretty straight-forward process and there are good tutorials to guide through the step-by-step wizard. But publishing from within the IDE is hardly something we could run as part of an automated setup.

<!-- more -->
[NAnt](http://nant.sourceforge.net/) has been working quite well for me, with all the build automation, unit testing and CI needs, I was inclined to trying to use it for this deployment automation as well. And even though there are a few relevant hits involving NAnt, MSBuild and ClickOnce when you google for "ClickOnce NAnt", there are a few gotchas in those articles, that are only just being stated and not really solved, especially with my specific environment and such. These include:

* The auto increment of the version number would happen only when published from within Visual Studio and not from MSBuild
* The deployment web page (with information about the version etc.) is generated only for a manual publish from the IDE
* There is no way to include the current version of the application into the product name (that would be seen by the end user in their Start Menu and/or desktop shortcut.

So I started piecing together the below process, standing on the shoulders of many of those nice articles/tutorials. And the below detailed process is what's been working for me right now for the automation of the ClickOnce deployment into the flickr downloadr website hosted at  GitHub Pages. The `deploy.bat` command is run from within `/build` directory of the `master` branch, which essentially runs NAnt with a build file that has a few targets to:

* increment the app version
* compile the application
* publish it to local directory and
* call the shell script to 
> - <small>clone the `gh-pages` branch from the GitHub repo to which we're publishing</small>
> - <small>commit and push the published ClickOnce application files</small>
> - <small>checkout the `master` branch, where the app itself is</small>
> - <small>commit and push the updated `version number` and `AssemblyInfo.cs` files</small>

After that brief intro, let's get into the details. And some code snippets.

## _As the `code` flies !_
###### Following the deploy process along with flow of the lines of code that does it

The sequence of steps from the single-click deployment process, gets kicked off by the `deploy.bat` file. Here are the contents of that file:

``` bat deploy.bat https://github.com/flickr-downloadr/flickr-downloadr/blob/master/build/deploy.bat on GitHub
go deploy Release
pause
```

Here `go` is another batch file that is there in the same directory:

``` bat go.bat https://github.com/flickr-downloadr/flickr-downloadr/blob/master/build/go.bat on GitHub
@echo off
IF dummy==dummy%2 (
nant\nant-0.92\bin\NAnt.exe -buildfile:FlickrDownloadr.build %1 -D:project.build.type=Debug
) ELSE (
nant\nant-0.92\bin\NAnt.exe -buildfile:FlickrDownloadr.build %1 -D:project.build.type=%2
)
date /t && time /t
pause
```

So, as evident here, this calls NAnt with `FlickrDownloadr.build` as the build file and `deploy` as the target to run, with `Release` as the build configuration.

``` xml deploy https://github.com/flickr-downloadr/flickr-downloadr/blob/5ed83541e657318da0cf5ef25bac53c60a709eea/build/FlickrDownloadr.build#L110 on GitHub
  <target name="deploy" depends="publish">
    <exec program="bash.exe" basedir="C:\Program Files (x86)\Git\bin\" verbose="true">
      <environment>
        <variable name="HOME" value="${environment::get-variable('userprofile')}"/>
        <variable name="ADDPATH" value=".:/usr/local/bin:/mingw/bin:/bin:/bin"/>
        <variable name="BUILDNUMBER" value="${buildnumber.version}"/>
      </environment>
      <arg file="deploy.sh" />
    </exec>
  </target>
```

This target first waits for its dependant target `publish` to execute. Let's see what that does first and we shall come back to this target to see when it would execute the `deploy.sh` bash script.

``` xml publish https://github.com/flickr-downloadr/flickr-downloadr/blob/5ed83541e657318da0cf5ef25bac53c60a709eea/build/FlickrDownloadr.build#L121 on GitHub
  <target name="publish" depends="compilesolution">
    <msbuild project="${source.directory}\${flickrdownloadr.app.project}">
      <arg value="/v:m"/>
      <arg value="/p:IsWebBootstrapper=true"/>
      <arg value="/p:UpdateEnabled=true"/>
      <arg value="/p:UpdateMode=Foreground"/>
      <arg value="/p:UpdateInterval=7"/>
      <arg value="/p:UpdateIntervalUnits=Days"/>
      <arg value="/p:UpdatePeriodically=false"/>
      <arg value="/p:Configuration=${project.build.type}"/>
      <arg value="/p:PublisherName=flickrdownloadr"/>
      <arg value="/p:ProductName=flickr downloadr (beta v${buildnumber.version})"/>
      <arg value="/p:ApplicationVersion=${buildnumber.version}"/>
      <arg value="/p:PublishDir=${bin.dir}\Deploy\"/>
      <arg value="/p:InstallUrl=http://flickrdownloadr.com/downloads/latest/"/>
      <arg value="/p:SupportUrl=http://flickrdownloadr.com/downloads/latest/"/>
      <arg value="/p:ErrorReportUrl=http://flickrdownloadr.com/"/>
      <arg value="/p:BootstrapperEnabled=true"/>
      <arg value="/p:CreateDesktopShortcut=true"/>
      <arg value="/p:CreateWebPageOnPublish=true"/>
      <arg value="/p:WebPage=index.html"/>
      <arg value="/t:publish" />
    </msbuild>
  </target>
```

This target again has another dependant target which is the compilation of the solution. Before we go there though, let's take one more closer look at this `publish` target.

It's simple invocation of the `msbuild` task (from NAntContrib) to invoke the `publish` build configuration (the line with `<arg value="/t:publish" />` towards the end) on the WPF application _project_ (and not the whole VS solution, which is what, as you'd see in a bit, the compilation invokes). A few of the properties being specified in there are moot though, for example, the install web page does not get created even with the really, positively asserting property in there!

The `compilesolution` task is another straight-forward one like this:

``` xml compilesolution https://github.com/flickr-downloadr/flickr-downloadr/blob/5ed83541e657318da0cf5ef25bac53c60a709eea/build/FlickrDownloadr.build#L40 on GitHub
  <target name="compilesolution" depends="cleanBin, createBin, increment-version">
    <echo message="Compiling Solution:" />

    <exec program="msbuild.exe"  basedir="C:\WINDOWS\Microsoft.NET\Framework\v4.0.30319"
			  commandline='"${source.directory}\${flickrdownloadr.solution}" /p:Platform="Any CPU" /p:Configuration=${project.build.type} /t:Rebuild /v:m /m' workingdir="." />
  </target>
```

As mentioned earlier, this time around MSBuild is being invoked on the Visual Studio solution itself. But the most interesting aspect here is the `increment-version` target that is a dependency:

``` xml increment-version https://github.com/flickr-downloadr/flickr-downloadr/blob/5ed83541e657318da0cf5ef25bac53c60a709eea/build/FlickrDownloadr.build#L66 on GitHub
  <target name="increment-version">
    <echo message="Incrementing the version:" />
    <version buildtype="NoIncrement" revisiontype="Increment" startdate="2012-04-02" verbose="true"/>
    <call target="create-common-assemblyinfo" />
  </target>
```

First the NAntContrib `version` task is being run and then the `create-common-assemblyinfo` target is called. 

The version task, if you read it along with [its documentation](http://nantcontrib.sourceforge.net/release/latest/help/tasks/version.html), you would see that only the revision number is being incremented here in flickrdownloadr, though you could use this task to increment the build number if you so choose. The major and minor versions are always supposed to be manually updated, by editing the build.number text file. 

Then we get to the [NAnt asminfo](http://nant.sourceforge.net/release/latest/help/tasks/asminfo.html) task to generate one, shared `CommonAssemblyInfo.cs` file, which is linked to by all the projects in the solution as their `AssemblyInfo.cs` modules.

``` xml create-common-assemblyinfo https://github.com/flickr-downloadr/flickr-downloadr/blob/5ed83541e657318da0cf5ef25bac53c60a709eea/build/FlickrDownloadr.build#L72 on GitHub
  <target name="create-common-assemblyinfo">
    <!-- ensure source/CommonAssemblyInfo.cs is writable if it already exists -->
    <attrib file="${source.directory}/CommonAssemblyInfo.cs" readonly="false" if="${file::exists('${source.directory}/CommonAssemblyInfo.cs')}" />
    <!-- Get Copyright Symbol -->
    <script language="C#" prefix="csharp-functions" >
      <code>
        <![CDATA[
              [Function("get-copyright-symbol")]
              public static string Testfunc(  ) {
                  return "\u00a9";
              }
            ]]>
      </code>
    </script>
    <!-- generate the source file holding the common assembly-level attributes -->
    <asminfo output="${source.directory}/CommonAssemblyInfo.cs" language="CSharp">
      <imports>
        <import namespace="System" />
        <import namespace="System.Reflection" />
        <import namespace="System.Runtime.InteropServices" />
      </imports>
      <attributes>
        <attribute type="ComVisibleAttribute" value="false" />
        <attribute type="AssemblyTitleAttribute" value="flickrDownloadr" />
        <attribute type="AssemblyDescriptionAttribute" value="A desktop application for windows that would help download all (or selected) photos from the user's photostream (in one of the selected sizes) along with the tags, titles and descriptions." />
        <attribute type="AssemblyConfigurationAttribute" value="${project.build.type}" />
        <attribute type="AssemblyCompanyAttribute" value="http://flickrdownloadr.com" />
        <attribute type="AssemblyProductAttribute" value="flickr downloadr" />
        <attribute type="AssemblyCopyrightAttribute" value="Copyright ${csharp-functions::get-copyright-symbol()} 2012-${datetime::get-year(datetime::now())} Haridas Pachuveetil" />
        <attribute type="AssemblyTrademarkAttribute" value="" />
        <attribute type="AssemblyCultureAttribute" value="" />
        <attribute type="AssemblyVersionAttribute" value="${buildnumber.version}" />
        <attribute type="AssemblyFileVersionAttribute" value="${buildnumber.version}" />
        <attribute type="AssemblyInformationalVersionAttribute" value="${buildnumber.major}.${buildnumber.minor}" />
      </attributes>
    </asminfo>
  </target>
```

I noticed (and learned about) this little beauty for the first time in the [NAnt build file](https://github.com/nant/nant/blob/7d81f6f2e4f35711e57de7306773cefc3875cd71/NAnt.build#L92) for the NAnt project itself!

So then, by the time the application is compiled, the version (automatically, only the revision; or if the major/minor/build numbers have been manually updated, them too) would have been incremented and a new `CommonAssemblyInfo.cs` with this value would have been generated, which all the projects are linking to. So all of the assemblies being generated by the compilation would now have the updated version numbers.

Coming back to the `publish` target now, it creates the ClickOnce deployable application files into the directory specified (`<arg value="/p:PublishDir=${bin.dir}\Deploy\"/>`) also with the dynamic version number as the publish version (`<arg value="/p:ApplicationVersion=${buildnumber.version}"/>`). 

Also, this version string would be included in the product name as well (`<arg value="/p:ProductName=flickr downloadr (beta v${buildnumber.version})"/>`), so everytime there is a release, the name of the app itself (only the name, it will still be treated as the same app as the older version by Windows, the ClickOnce updater service etc.) will have the current deployed version number in it; a la **flickr downloadr (beta v0.3.4.26)**. Ain't that nice?

Moving onto the not-so-short bash script we are running as part of the `deploy` target that invoked the `publish`. Before we get into the script itself if you scroll back up to the snippet that invokes it, there are a couple of environment variables being added from NAnt to be made available in the bash. That maneuver took sometime to discover, but I thought it's a nice little way to pass stuff in to the bash script from within the build environment. And most importantly for us, this includes the `BUILDNUMBER` variable as well which has the current app version.

The first 50 lines within the `deploy.sh` script is to ensure that the currently logged in SSH identity is being reused (probably there is a better way to do this, or a few lines could be shaved off from this version itself as there is no way one would expect the deploy script to interactively ask to login afresh, if no one is already logged in. Anyways...). I thought the rest of the bash script is also pretty self explanatory, so I am just linking it all here:

{% gist 4536562 deploy.sh %}

If you notice, the `build.number` file (which is what gets updated by the `version` task and therefore holds the current version of the app), is being copied over to the `gh-pages` branch and being committed everytime. This text file is then dynamically queried via jQuery ajax from the home page and the downloads page, to fetch and display the current version of the deployed application. 

Even though this is a very minor piece in the whole scheme of things, to have the site show the version number of the latest available release (that too, let me do this one more time, with a deployment effort of a single-click), I thought was a really cool thing.

One really nagging thing after everything started (seemingly) working as expected, was that the ClickOnce updater/installer started complaining about some signing or manifest or something (I can't find the exact error now, shall recreate it and update here soon) and would not install/upgrade the app at all. After some googling  found out that it had to do with how the cr-lf differences between Windows and *NIX are being handled by the Git clients and/or GitHub etc. 

To fix this, a `.gitattributes` file was added into the `gh-pages` branch `downloads` directory, following [this helpfile at GitHub](https://help.github.com/articles/dealing-with-line-endings), to the effect of asking git to treat everything within the `/downloads` directory as non-text files:
``` plain .gitattributes https://github.com/flickr-downloadr/flickr-downloadr/blob/gh-pages/downloads/.gitattributes on GitHub
*  -text
```

## That's it!
All of that put together, the latest version could now be deployed to the [flickrdownloadr.com](/../downloads/latest) web site just by running a batch file.

A couple of things to note though:
1. If there are local changes (that are committed or not), it would be a good idea to commit and push them all before deploy. This is especially for avoiding unnecessary conflicts to files like `build.number` or `CommonAssemblyInfo.cs` from within the `deploy.sh`
2. Also running a pull on the local repo before running the deploy would be a good thing.

Hopefully this would help you in setting up your one-click ClickOnce deployment.

Continuous Deployment, for the win !