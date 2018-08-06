---
layout: post
title:  "Creating Stubs for Ngx Translate"
author: "Jeff-Stapleton"
twitter: "jeffdstapleton"
date:   2018-07-06 11:20:00 -0600
tags: [angular, ngx-translate]
---
I had an issue the other day with ngx translate in my tests. I feel like there isn't a ton of documentation or examples out there so I figured I'd do a quick post explaining the issue and solution. 

I primarily used ngx translate directly in the html. translating static strings to their proper culture. Which usually looks something like the example below:

```html
{% raw %}<div class="banner-title">{{ addressTitleTranslationKey | translate }}</div>{% endraw %}
```

Pretty straight forward and it works like a charm in real life as well as in tests.

Yesterday, however, I wanted to create a hint string that could be one of a few things. I created a typescript function that the html calls, returning one of two possible strings.

```html
{% raw %}<div class="input-hint">{{ getInputHint() }}</div>{% endraw %}
```

```ts
  private getInputHint() {
    let inputHint = ""
    if (this.inputBad) {
      inputHint = this.translationPipe.transform(this.inputInvalidMessageKey)
    } else {
      inputHint = this.translationPipe.transform(this.inputHintKey)
    }
    return inputHint
  }
```

The `TranslatePipe` is how you translate in typescript. Simply call `.transform(this.translationKey)` providing it with the key and it will return the string in the proper culture. 

This all works perfectly in real life but when tests come into play, things get a little tricky.

Because we are calling `getInputHint()` directly in the html this function gets call for every test related to this component, wether you want to test translations or not. So we need to ensure that our stubs for `TranslateService` and `TranslatePipe` are correctly setup.

First, let's look at `TranslatePipe`

```ts
@Injectable()
@Pipe({ name: "translate" })
export class TranslatePipeStub implements PipeTransform {
  public transform(key: string, ...args: any[]): any { return key }
}
```

This stub is pretty straight forward. Since we are just using `transform`, it is the only method we include here -- and since we don't actually care about the translation we simply just return the key. I found it interesting though that despite simply returning the key, ngx still makes a request to `TranslateService.get(key: string)`.

So let's create a stub for `TranslateService` too

```ts
@Injectable()
export class TranslationServiceStub {
  public get(key: any): any { return Observable.of(key) }
}
```

*FYI if you are using a linter it is going complain about `get` being a reserved word, just ignore it*

With both the `TranslatePipe` and `TranslateService` successfully stubbed out, it would appear we are good to go. Unfortunately that isn't the case. This was the part that left me baffled for the better part of my afternoon. If you run your tests with the stubs looking like the above examples, you will get the following error

```text
Failed: Cannot read property 'subscribe' of undefined
```

If you are like me, you might assume that something is wrong with the `get` method, that the observable it is returning is somehow undefined. It took some digging around in compiled code to find that the `TranslateServiceStub` requires three event emitters to function correctly. 

*For your convenience I am just going to share my whole `TranslateServiceStub`, in contains stuff that is outside the scope of this example but is quite commonly used*

```ts
@Injectable()
export class TranslationServiceStub {
  public onLangChange = new EventEmitter<any>()
  public onTranslationChange = new EventEmitter<any>()
  public onDefaultLangChange = new EventEmitter<any>()
  public addLangs(langs: string[]) { return }
  public getLangs() { return ["en-us"] }
  public getBrowserLang() { return "" }
  public getBrowserCultureLang() { return "" }
  public use(lang: string) { return null }
  // tslint:disable-next-line:no-reserved-keywords
  public get(key: any): any { return Observable.of(key) }
}
```

With your stubs now correctly implemented the last thing, which is totally optional, I would do is create a module for you translate stubs. Since they are most likely gonna be used quite frequently, It is nice to only have to import the module.

```ts
@NgModule({
  declarations: [
    TranslatePipeStub,
  ],
  exports: [
    TranslatePipeStub,
  ],
  providers: [
    { provide: TranslateService, useClass: TranslationServiceStub },
  ],
})
 export class TranslateStubsModule { }
```

I hope this proves useful to at least some of you. Good luck and happy coding!