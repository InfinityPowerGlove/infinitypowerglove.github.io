---
layout: post
title: "Enabling the iOS Clear Button in NativeScript"
author: "interrobrian"
twitter: "interrobrian"
date: 2018-09-17 16:15:00 -0600
tags: [angular, nativescript]
---

If you'd like to use the iOS clear button on a NativeScript text field, it's actually quite easy. All you have to do is enable it on the native iOS UITextField:

{% highlight html %}
<TextField
  ios.clearButtonMode="1"
></TextField>
{% endhighlight %}

Values for this property come from [UITextField.ViewMode](https://developer.apple.com/documentation/uikit/uitextfield/viewmode). However, there area couple caveats:
1. Just enabling the clear button will cause your input text to go under the clear button.
2. ngModel will make the clear button disappear.

To fix these, you must adjust the right padding when the field is focused, and avoid ngModel. Here's a simple way to do it in NativeScript + Angular:

{% highlight html %}
<TextField
  ios.clearButtonMode="1"
  [text]="myTextVariable"
  (textChange)="myTextVariable = $event.value"
  (focus)="$event.object.paddingRight = isIOS ? 30 : 10"
  (blur)="$event.object.paddingRight = 10"
></TextField>
{% endhighlight %}

{% highlight typescript %}
import { isIOS } from "tns-core-modules/platform"
@Component({...})
export class MyComponent {
  public isIOS = isIOS
}
{% endhighlight %}

Hopefully those of you using NativeScript Core or Vue find that easy to translate!
