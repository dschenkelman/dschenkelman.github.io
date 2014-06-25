---
layout: post
title: 'Windows Phone 8.1 NuGet packages with XAML components'
categories:
- NuGet
- Windows Phone
status: publish
comments: true
type: post
published: true
author:
  login: dschenkelman
  email: damian.schenkelman@gmail.com
  display_name: Damian Schenkelman
  first_name: Damian
  last_name: Schenkelman
---

As part of our multiple Auth0 SDKs, we have a [NuGet package](https://www.nuget.org/packages/Auth0.WindowsPhone/) that provides consumers an easy way to display our Login Widget in Windows Phone applications.

![Widget](https://camo.githubusercontent.com/d1e24b58d92efc4eefb94f07af971b5ef3787aca/687474703a2f2f7075752e73682f346e5a314a2e706e67)

With the release of Windows Phone 8.1 we received several requests asking for a new version of the package that supports the newly available [Windows Phone 8.1 type of project](http://blogs.msdn.com/b/dotnet/archive/2014/04/30/get-your-libraries-ready-for-windows-phone-8-1.aspx), so we the first step was to update our [library and sample projects in GitHub](https://github.com/auth0/auth0.windowsphone).

Having done that, I used the [NuGet Package Explorer](http://npe.codeplex.com) to update our package. Initially I added support our only dependency, Json.NET.

<figure>
  <img src="{{ site.url }}/images/posts/20140625/addDependency.png" alt="Dependencies">
</figure>

After that I added the Windows Phone 8.1 assembly to the required folder and tried the package out.

<figure>
  <img src="{{ site.url }}/images/posts/20140625/initialStructure.png" alt="Initial Structure">
</figure>

Unfortunately I got a not very descriptive exception saying `XAML parsing failed` when the `InitializeComponent` method of our `LoginPage.xaml` was invoked. After some research and a short twitter [conversation](https://twitter.com/dschenkelman/status/481459023901118464) with [@timheuer](https://twitter.com/timheuer) I found that I had to also manually include the following files that were part of the **bin** folder:

* *.pri
* *.xr.xml
* *.xbf

I checked the ***.pri** file to find the relative path where the other two files were being looked for and ended up with the following structure.

<figure>
  <img src="{{ site.url }}/images/posts/20140625/additionalComponents.png" alt="Additional Components">
</figure>

One final adjustment I had to make was to manually change the `targetFramework` from **WindowsPhoneAppx8.1** to **WindowsPhoneApp8.1** (without the 'x') in the **.nuspec** file outside of NuGet Package Explorer, as it seems there is a small bug with the naming it provides.

<figure>
  <img src="{{ site.url }}/images/posts/20140625/dependencies.png" alt="Dependencies change">
</figure>




