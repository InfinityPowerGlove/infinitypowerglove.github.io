---
layout: post
title:  "Accessing the Android Activity in NativeScript"
author: "interrobrian"
twitter: "interrobrian"
date:   2018-05-30 16:15:00 -0600
tags: [angular, nativescript]
---
Recently, I had an issue where I needed to access the Android activity/window for a NativeScript app. I needed to temporarily change the soft input mode so that the keyboard would display properly while a WebView was visible. It ended up being pretty simple, but figuring out all the pieces was a little tricky:

{% highlight typescript %}
import { android } from "tns-core-modules/application"
import { isAndroid } from "tns-core-modules/platform"

// softInputMode is this enum: https://developer.android.com/reference/android/view/WindowManager.LayoutParams
export function setSoftInputMode(softInputMode: number) {
  // Make sure to check the platform before doing this
  if (isAndroid) {
    // Android.foregroundActivity is the current activity, specified in your AndroidManifest.xml
    const window = android.foregroundActivity.getWindow()
    // Types for this stuff all come up as any - check the Android API docs or debug to discover properties
    window.setSoftInputMode(softInputMode)
  }
}
{% endhighlight %}

And that's all there is to it!