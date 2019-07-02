# Firebase Analytics SDK integration For Eclipse

It is easy to add Firebase SDK in Android Studio, since official doc sugguests. However, there would be a chance that you have to deal with Eclipse. Hope you never have to.

Let go of nonsense words. Let's dive into it.

First of all, Eclipse takes Jar library for SDK integration and Android Studio takes AAR usually with gradle. So we have to know how Android Studio, more specifically, gradle handles the integration of Firebase SDK for us. 

There are two main steps:
1. Add the Google Services plugin and apply it.
``` groovy
// in build.gradle(Project)
buildscript {
  // ...
  dependencies {
    // ...
    // Add the following line:
    classpath 'com.google.gms:google-services:4.2.0'  // Google Services plugin
  }
}

allprojects {
  // ...
  repositories {
    // Check that you have the following line (if not, add it):
    google()  // Google's Maven repository
    // ...
  }
}
// in build.gradle(app)
// Add the following line to the bottom of the file:
apply plugin: 'com.google.gms.google-services'  // Google Play services Gradle plugin
```

2. Add the dependencies.
``` groovy
dependencies {
 // ...
 implementation 'com.google.firebase:firebase-core:16.0.8'

 // Getting a "Could not find" error? Make sure that you've added
 // Google's Maven repository to your root-level build.gradle file
}
```

It is obvious the key is the Play Services plugin which gets the work done. When register app in Firebase console, a config file(play-services.json) which contains app's information will be available for download and put into the project. Next in compiling time, Gradle parses the config file and converts into a value xml file which required by Firebase SDK initialization.

For example the generated value.xml(Referenced from official doc):
``` xml
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <! -- Present in all applications -->
    <string name="google_app_id" translatable="false">1:1035469437089:android:73a4fb8297b2cd4f</string>

    <! -- Present in applications with the appropriate services configured -->
    <string name="gcm_defaultSenderId" translatable="false">1035469437089</string>
    <string name="default_web_client_id" translatable="false">337894902146-e4uksm38sne0bqrj6uvkbo4oiu4hvigl.apps.googleusercontent.com</string>
    <string name="ga_trackingId" translatable="false">UA-65557217-3</string>
    <string name="firebase_database_url" translatable="false">https://example-url.firebaseio.com</string>
    <string name="google_api_key" translatable="false">AIzbSyCILMsOuUKwN3qhtxrPq7FFemDJUAXTyZ8</string>
    <string name="google_crash_reporting_api_key" translatable="false">AIzbSyCILMsOuUKwN3qhtxrPq7FFemDJUAXTyZ8</string>
    <string name="project_id" translatable="false">mydemoapp</string>

</resources>
```

### GET JARS FROM THE DEPENDENCIES
Grasp all needed dependencies from the gradle cache. The dependencies are listed below as follow:
```
com.android.support-v4:support-compat
com.android.support-v4:support-core-utils
com.google.android.gms:play-services-basement
com.google.android.gms:play-services-tasks
com.google.android.gms:play-services-stats
com.google.android.gms:play-services-measurement-base
com.google.firebase:firebase-analytics
com.google.firebase:firebase-analytics-impl
com.google.firebase:firebase-common
com.google.firebase:firebase-core
com.google.firebase:firebase-iid
com.google.firebase:firebase-iid-interop
com.google.firebase:firebase-measurement-connector
com.google.firebase:firebase-measurement-connector-impl
```

All of the dependency files are in AAR format. Uncompress these files respectively and get out of the class.jar file which need imported in Eclipse.

### COPY NECESSARY DATA INTO ANDROIDMAINFEST.XML
The SDK handles the initialization by itself automatically. Because Jar doesn't come with resouce files and AndroidMainfest, we have to copy the necessary data from every AAR file. They are significant to get the Firebase SDK work. 

``` xml

<!-- Neccessary permissions-->
<permission android:name="${applicationId here}.permission.C2D_MESSAGE"
    android:protectionLevel="signature"/>

<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE"/>
<uses-permission android:name="${applicationId here}.permission.C2D_MESSAGE"/>

<!-- Paste the next section between Applcation label -->

<!-- play service basement AndroidManifest.xml -->
<meta-data android:name="com.google.android.gms.version" 
    android:value="12451000" />


<!-- Firebase analytics AndroidManifest.xml -->
  
<receiver
  android:name="com.google.android.gms.measurement.AppMeasurementReceiver"
  android:enabled="true"
  android:exported="false" >
</receiver>
<receiver
    android:name="com.google.android.gms.measurement.AppMeasurementInstallReferrerReceiver"
    android:enabled="true"
    android:exported="true"
    android:permission="android.permission.INSTALL_PACKAGES" >
    <intent-filter>
        <action android:name="com.android.vending.INSTALL_REFERRER" />
    </intent-filter>
</receiver>

<service
    android:name="com.google.android.gms.measurement.AppMeasurementService"
    android:enabled="true"
    android:exported="false" />
<service
    android:name="com.google.android.gms.measurement.AppMeasurementJobService"
    android:enabled="true"
    android:exported="false"
    android:permission="android.permission.BIND_JOB_SERVICE" />
  
<!-- Firebase common AndroidManifest.xml -->

  <provider
    android:name="com.google.firebase.provider.FirebaseInitProvider"
    android:authorities="com.jimjay.demo.firebaseinitprovider"
    android:exported="false"
    android:initOrder="100" />
  
  
<!-- Firebase iid AndroidManifest.xml -->
<receiver
  android:name="com.google.firebase.iid.FirebaseInstanceIdReceiver"
  android:exported="true"
  android:permission="com.google.android.c2dm.permission.SEND" >
  <intent-filter>
      <action android:name="com.google.android.c2dm.intent.RECEIVE" />
      <action android:name="com.google.android.c2dm.intent.REGISTRATION" />

      <category android:name="com.jimjay.demo" />
  </intent-filter>
</receiver>
<receiver
    android:name="com.google.firebase.iid.FirebaseInstanceIdInternalReceiver"
    android:exported="false" />

<service android:name="com.google.firebase.components.ComponentDiscoveryService" >
    <meta-data
        android:name="com.google.firebase.components:com.google.firebase.iid.Registrar"
        android:value="com.google.firebase.components.ComponentRegistrar" />
</service>
<service
    android:name="com.google.firebase.iid.FirebaseInstanceIdService"
    android:exported="true" >
    <intent-filter android:priority="-500" >
        <action android:name="com.google.firebase.INSTANCE_ID_EVENT" />
    </intent-filter>
</service>
```
Note that the value of _com.google.android.gms.version_ is hardcoded, which is fine to do.


### PARSING THE CONFIG FILE MANUALLY
You can convert the file manually according to ![doc](https://firebase.google.com/docs/reference/gradle/#processing_the_json_file) or use !(online converter)[https://dandar3.github.io/android/google-services-json-to-xml.html] which thanks to _dandar3_ on Github. Eventually, append the values in the end of string.xml.

### CHECK IF IT WORKS
I belives you have imported the Jars as libraries in Eclipse. Now, it's all done. Final step, You can run the app and look up the logcat to find if there's a message saying "FirebaseApp initialized successfully". If so, you may see some messages saying events uploaded as well.


### REFERENCES
![Thanks to dandar3's insightful project](https://github.com/dandar3/android-google-firebase-README)
![The Google Services Gradle Plugin-Official Firebase Doc](https://developers.google.com/android/guides/google-services-plugin)
