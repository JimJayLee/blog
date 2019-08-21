### Problem with Code Shrinking in Modularization

##### Modularized Projects

Split a whole project into separated ones. Usually, the dependencies in them are "compileOnly" in case compiling the whole project prompts error of duplicated dependencies. 


##### Problem

Given an obfuscated SDK which is compiled with multi modules in AAR, integrate it into an application along with other dependencies which contain resources files like Facebook-common, CardView, etc. Log in with Facebook and the App crashes with log below.

```
Process: com.eskyfun.sgsmjzgp, PID: 5980
java.lang.AbstractMethodError: abstract method "void com.facebook.FacebookCallback.onError(com.facebook.FacebookException)"
at com.facebook.login.LoginManager.finishLogin(LoginManager.java:705)
at com.facebook.login.LoginManager.onActivityResult(LoginManager.java:245)
at com.facebook.login.LoginManager$1.onActivityResult(LoginManager.java:174)
at com.facebook.internal.CallbackManagerImpl.onActivityResult(CallbackManagerImpl.java:91)
at com.eskyfun.sdk.helper.a.a(FacebookLoginHelper.java:115)
at com.eskyfun.sdk.ui.MainActivity.onActivityResult(MainActivity.java:75)
at android.app.Activity.dispatchActivityResult(Activity.java:7266)
at android.app.ActivityThread.deliverResults(ActivityThread.java:4264)
```
_AbstractMethodError_ means you have not implemented any declared methods yet and you call them. Actually, I did implement them in the _facebook_login_ module.

##### Exploring

However, App in debug mode works fine. Obfuscation may play an important role in causing the error. 
Let's dive into the source code. Decompiling the SDK, I found something wrong.

There is the Facebook login logic showing below:

```java
LoginManager.getInstance().logOut();
LoginManager.getInstance().registerCallback(this.b(), new FacebookCallback<LoginResult>() {
});
```

The callback methods in FacebookCallback instance are erased completely. As a result, the loginManager does not find the callback implementation, and throws _AbstractMethodError_ instead.

How come Proguard removes the codes that are used by FacebookSDK?  

> Code shrinking with R8 is enabled by default when you set the minifyEnabled property to true.

It seems code shrinking is doing its job. How does it work?

> To shrink your app’s code, R8 first determines all entry points into your app’s code based on the combined set of 
> configuration files. These entry points include all classes that the Android platform may use to open your app’s
> Activities or services. Starting from each entry point, R8 inspects your app’s code to build a graph of all methods,
> member variables, and other classes that your app might access at runtime. Code that is not connected to that
> graph is considered unreachable and may be removed from the app.

There should be a way telling Proguard not to remove any code I wanna keep.

> R8 determines entry points through -keep rules in the project’s R8 configuration files. That is, keep rules specify
> classes that R8 should not discard when shrinking your app, and R8 considers those classes as possible entry
> points into your app. The Android Gradle plugin and AAPT2 automatically generate keep rules that are required by
> most app projects for you, such as your app’s activities, views, and services. However, if you need to customize this
> default behavior with additional keep rules, read the section about how to customize which code to keep.

Adding a rule that keeps the callback methods may do the trick. 

```
-keep interface com.facebook.FacebookCallback {*;}
```

Compile codes, build and decompile AAR. Unfortunately, the FacebookCallback's body still remains empty. 

What's wrong? I look up the proguard rules again. Is there any implies that the callback methods are unreachable?

When compiling the SDK, the _facebook-common_ dependency is not implemented. That's to say, the FacebookCallback interface means an unknown pieces of code. Therefore, Proguard removes them.

Let SDK project have reference to _facebook-common_ dependency. Compile and everything works fine. No error!


