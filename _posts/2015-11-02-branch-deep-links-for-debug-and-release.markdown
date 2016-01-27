---
layout: post
title:  Branch deep links for debug and release on the same device
date:   2015-11-02
author: ernest
tags:
- Branch
- Branch.io
- Android
- Deep-links
- Debug
---

We have recently adjusted our app's Gradle build scripts to allow us to have multiple versions (e.g. debug and release) installed on one device. One of the problems we stumbled upon was that our [Branch](https://branch.io/) deep linking didn't work as we wanted: when creating a Branch link from the debug version of the app, the link would open the release version of the app instead of the debug version. The solution we implemented does not seem obvious nor documented, hence this blog post.


## Before we start
This post makes 2 assumptions:

 1. You already have Branch deep links working for both the *Live* and  *Test* environments of Branch. I.e. you can open Live links in the release version of app and Test links in test/debug version of your app. But you don't/can't have both versions installed at the same time.
 1. The different versions of your app have different `applicationId`s. In our case, we use `com.feedpresso.mobile` for the release version and `com.feedpresso.mobile.debug` for the debug version of our app.

## Setting up the app
Branch's deep links are routed to your app using intent-like URIs where the receiving is determined by the URI's scheme. For example, for Feedpresso we used a scheme like `feedpresso://`. For this to work for different versions installed at the same time, we need to use different schemes.

First, we set things up on the receiving end -- the app. As per the [Branch docs](https://dev.branch.io/recipes/quickstart_guide/android/), the scheme your app should listen on is defined in your in *AndroidManifest.xml*:

```xml
<activity ... >
  <intent-filter>
    ...
    <data android:scheme="feedpresso" android:host="open" />
    ...
  </intent-filter>
</activity>
```

To change this per build type, we can use the Gradle's [`manifestPlaceholders`](http://developer.android.com/tools/building/manifest-merge.html). In our *build.gradle*, we define the different schemes like this:

```java
defaultConfig {
        manifestPlaceholders = [ branchUriScheme:"feedpresso" ]
}

buildTypes {
    debug {
        manifestPlaceholders = [ branchUriScheme:"feedpresso-debug" ]
    }
}
```

Now we can replace the aforementioned AndroidManifest entry with the following:

```xml
<data android:scheme="${branchUriScheme}" android:host="open" />
```

And the placeholder will be replaced with the right value at build time.

## Setting up Branch
Now we'll make sure that Branch links will send the right requests.

  1. On your [Branch Dashboard](https://dashboard.branch.io/), make sure you have the appropriate environment selected, e.g. **Test**.
  1. go to **Settings** -> [**Link Settings**](https://dashboard.branch.io/#/settings/link) and scroll to **Android redirects**.
  1. Under **Android URI Scheme**, enter the scheme name you specified earlier. We went for `feedpresso-debug://`.

To prevent Branch from launching Play Store before opening a Branch link for the first time (i.e. the [default behavior](https://dev.branch.io/recipes/add_custom_link_data_and_routing/android/#always-try-to-open-the-app---alwaysdeeplink)), we can specify a **Custom URL** in the same section on the Dashboard.

We set the URL to `http://localhost` and the Android package name to the `packageId` used in our debug version, e.g. `com.feedpresso.mobile.debug`.


That is all the set-up that was required for us. I hope this helps you use Branch for release and debug versions of your app installed on one device.
