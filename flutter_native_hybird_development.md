### Hybird development
Flutter can make a beautiful interface quickly and easily. Therefore I chose Flutter for a internal project to build its layout. Since the project is highly dependent on Android native features such as service, executing ADB instructions and Wifi services, Flutter, integrated in module way,  is acting as a display showing info and handling inputs. As for communications, platform channels are official recommended.

Obviously, the native end handles buissiness logic and Flutter module only takes care of UI display.
Note that this article will only focus on Android platform only and iOS integration tutorials could be found on official Flutter Github page.


### Create Flutter module
Assuming a regular Android project exists in workplace, create a Flutter module with commands in termial:

```bash
cd android_project_path/
flutter create -t module --org package_name module_name
```

#### Host app requirements
Before connecting Flutter module to host project, make sure that host app supports Java 8 by declcaring source compatibility within app's _build.gradle_ file, under the _android{}_ block:

```groovy
android{
    compileOptions{
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }
}
```

#### Host app references Flutter module
Actually, in official wiki, there are two ways to implement Flutter dependency. Those are compiling Flutter module into Android archive(AAR) and directly depending on the module. And we are going on with the second one, since it is painful for debugging to build AAR every time.

Note that Flutter SDK is always needed.

Turn to the host app's _settings.gradle_ and include the Flutter module as a sub project:

```groovy
include ':app'                                     // assumed existing content
setBinding(new Binding([gradle: this]))                                 // new
evaluate(new File(                                                      // new
  settingsDir.parentFile,                                               // new
  'module_name/.android/include_flutter.groovy'                          // new
)) 
```

Introduce an implementation dependency on the Flutter module from host app:

```groovy
dependencies{
    implementation project(':flutter')
}
```

The work of implementing Flutter module is accomplished, nice. Move on to build UI with Flutter.



### Start Flutter activity in host app
Flutter supports showing a Flutter view in native window and showing a whole Flutter screen. Therefore, we are replacing the MainActivity with FlutterActivity(which is in _io.flutter.app_). 

1. Make MainActivity extend FlutterActivity and connect to Flutter:

```java
public class MainActivity extends FlutterActivity{
    // codes

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {

        FlutterMain.startInitialization(this);
        super.onCreate(savedInstanceState);

        GeneratedPluginRegistrant.registerWith(this);

        getFlutterView().setInitialRoute("flutterView");

        // codes
    }
}

2. Make Application extend FlutterApplication, if no custom application in host, create one.

```java
public class App extends FlutterApplication{
    // codes
}
```

3. add necessary declarements in AndroidManifest.xml:

```
<application
        android:name=".App"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme" >


        <activity android:name=".MainActivity"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

</application>
```

Make sure that the features embeded in activity including _configChanges_ and _windowSoftInputMode_ are added, otherwise layout would not be functioning well with oriention changes and keyboard showing.


### Building and running app 

After installing, Flutter layout should be on screen already. Are hot restart or hot reload supported? Of courses, there are some steps might be gone through. 

Connect a device or launch an emulator. Then make Flutter CLI tooling listen for your app to come up:

```bash
cd host_app/flutter_module
flutter attach
```

Run app in debug mode, there would be following outputs in terminal when app is opened. Then you are okay to use _r_ for hot reload and _R_ for hot restart.



### REFERENCES
[Github-Add Flutter to existing apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)
