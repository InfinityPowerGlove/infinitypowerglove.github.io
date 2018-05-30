---
layout: post
title:  "Angular Compiler Arrow Function Bug"
author: "interrobrian"
twitter: "interrobrian"
date:   2018-05-10 14:20:00 -0600
categories: angular
---
I came across an odd bug with the Angular Compiler recently. It occurs when it's targeting es5 and encounters a single statement arrow function with a comment on a line between the `=>` and the statement. This has been tested with both Angular 5 and Angular 6. **To clarify, you will only get this bug when using the Angular Compiler, which typically means when compiling an Angular package library (such as with the new `ng g library` command).** Consider the following block of code:

{% highlight typescript %}
const oneThroughFive = [1, 2, 3, 4, 5]
const lessThanThree =
  oneThroughFive.filter(n =>
    // Filter numbers less than 3
    n < 3);
{% endhighlight %}

Compiling and running this with the TypeScript Compiler (`tsc`) results in `lessThanThree` correctly being `[1, 2]`. However, compiling and running this with the Angular Compiler (`ngc`) results in `lessThanThree` being `[]`.

The reason for this is that the Angular Compiler is producing incorrect JavaScript. Here is the output from the TypeScript Compiler:

{% highlight javascript %}
var oneThroughFive = [1, 2, 3, 4, 5];
var lessThanThree = oneThroughFive.filter(function (n) {
    // Filter numbers less than 3
    return n < 3;
});
{% endhighlight %}

However, here is the output from the Angular Compiler:

{% highlight javascript %}
var oneThroughFive = [1, 2, 3, 4, 5];
var lessThanThree = oneThroughFive.filter(function (n) {
    // Filter numbers less than 3
    return 
    // Filter numbers less than 3
    n < 3;
});
{% endhighlight %}

Notice that the Angular Compiler has not only duplicated the comment, but _it's put the duplicated comment inbetween the `return` statement and the predicate_. This creates a significant problem because JavaScript's automatic semicolon insertion will cause the arrow function to [return nothing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/return#Description), and thus the filter function filters everything.

To work around this, simply set the compiler to remove comments by adding this to your `tsconfig.json`:

```
{
  "compilerOptions": {
    ...
    "removeComments": true
  }
}
```
Alternatively, you can place your comment anywhere other than on the line between the `=>` and the statement. Even the same line as the `=>` will work:

{% highlight typescript %}
const oneThroughFive = [1, 2, 3, 4, 5]
const lessThanThree =
  oneThroughFive.filter(n => // Filter numbers less than 3
    n < 3);
{% endhighlight %}

I've opened up a [GitHub issue](https://github.com/angular/angular/issues/23829) on the Angular repository, so hopefully this gets addressed in the future!

**UPDATE** May 11, 2018: [trotyl](https://github.com/trotyl) on GitHub [correctly identified](https://github.com/angular/angular/issues/23829#issuecomment-388263907) the root cause of the problem as a 
[tsickle](https://github.com/angular/tsickle) issue. Therefore, I have created a [GitHub issue](https://github.com/angular/tsickle/issues/802) on tsickle for this as well.

**UPDATE 2** May 15, 2018: It turns out, the root cause of this problem was an [issue](https://github.com/Microsoft/TypeScript/issues/24096) with TypeScript transformations. It has now been fixed on both [tsickle](https://github.com/angular/tsickle/pull/804) and [TypeScript](https://github.com/Microsoft/TypeScript/pull/24135), so it should be resolved in the next Angular release!