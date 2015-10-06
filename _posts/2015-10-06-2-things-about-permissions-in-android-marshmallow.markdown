---
layout: post
title:  "2 things you need to know about new Android permissions in Marshmallow as a developer"
date:   2015-10-08 10:12:51
tags:
- feedpresso
- android
- permissions
- marshmallow
---

Android is releasing a new permission model
with [Marshmallow](https://www.android.com/versions/marshmallow-6-0/).
A bit ago, we were searching, how things are going to change for us as developers.
The change is, that now, permissions are going to be requested at runtime
of application instead of being requested during an install.
However, there are two things to know before diving deep.

# targetSdkVersion

As long as you do not set the target SDK version to 23, you are fine - the Google Play
store and Android are going to use the old permission model and there should be
no need of changes on your side. The permissions are going to be requested
during install time as before.

# Normal permissions

You are not going to need to do any changes, if one of the
[Normal Permissions](https://developer.android.com/guide/topics/security/normal-permissions.html)
that you are granted automatically. If you
use only these, there won't be any SecurityException
during runtime regarding them. Also, users cannot revoke them as well.

# Other
For all the other things you are going to need to go
through [Permissions documentation](https://developer.android.com/guide/topics/security/permissions.html)
to know how to handle requests for these permissions properly.
