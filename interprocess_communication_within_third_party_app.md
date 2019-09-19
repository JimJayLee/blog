 ## Interprocess Communication
There are a few ways for IPC, such as IBinder, Socket, Broadcast, etc. In my situation, I want to make calls from a remote process that is not belong to my own Application. Let's say, I want to make calls in Settings' process from Google Play Store's process.

### Problem
IBinder, AIDL, or something involes remote services, are all necessary to deal with AndroidManifest. A service that received signals from other processes has to be running in the target process. Howver, services need being declared in the AndroidMainfest.xml first. That seems not working that we modifiy the AndroidManifest file, add something and repackage the App. Nether does Broadcast. 

### Hook What?
In the last _[post](https://github.com/JimJayLee/blog/blob/master/set_wifi_proxy_programmatically.md)_, we encoutered a problem involved system-level permission and ended up hooking a system App to run specific codes.Be more detailed, hook the activity lifecycle of MainActivity in Settings and put the to-do codes in _resume_ or any method that would be executed at least once. Start the MainActivity from Google play store's process. However, every time we want to modify proxy settings, Settings pops up in the foreground, which causes an interruption in our job. 


### Passing Data with Intent

Moreover, look into the signature of _changeProxy_:
```java
 public void changeProxy(String ip, int port){
        try {
            Class<?>[] classes = new Class[]{String.class, int.class, String.class};
            ProxyInfo proxyInfo = (ProxyInfo) XposedHelpers.newInstance(ProxyInfo.class, classes, ip, port, "");
            WifiManager wifiManager = (WifiManager) application.getApplicationContext().getSystemService(Context.WIFI_SERVICE);
            List<WifiConfiguration> wifiConfigurationList = wifiManager.getConfiguredNetworks();

            for (WifiConfiguration wifiConfiguration:
                    wifiConfigurationList) {
                if (getCurrentWifiSSID(application.getApplicationContext()).equalsIgnoreCase(wifiConfiguration.SSID)){
                    wifiConfiguration.setHttpProxy(proxyInfo);
                    XposedHelpers.callMethod(wifiManager, "save", wifiConfiguration, null);
                }
            }
            log("change proxy success");
        } catch (Exception e){
            e.printStackTrace();
            log( "CHANGE PROXY FAILED " + e.getLocalizedMessage());
        }
    }
```

It takes two strings as paramter that should be passed by the caller. Starting activity from outside is not helping at all. Communications is done through *Intent* between activities. Intent can convey data like strings, integers or parcelable object. How is it received? That comes up with _getIntent_:

```java
/** Return the intent that started this activity. */
    public Intent getIntent() {
        return mIntent;
    }
```

That means _Intent_ can be transported cross processes. As a result, _mIntent_ is the data carrier. 

### Services Hooking
As it is mentioned above, activity comes to face by starting activity for proxy modifications. It is obvious that data is conveyed by _Intent_. So we have look for what else can transport _Intent_ except _Activity_
. 

>There are four main Android app components: activities , services , content providers , and broadcast receivers . Whenever you create or use any of them, you must include elements in the project manifest.

As known to all, four components all carry a _Intent_. Except that activity is visible, service and broadcast sound like better choices for interactions. A services that declared exported is allowed to start from outside of application. Either is broadcast receiver. But broadcast receiver can be used with a permision to limit the external entites that intend to invoke it, which is a lot more compilcated. For convenience, service is a efficent way for our goal. 

Let's recap why choose service:
1. Not visible or interrupted.
2. Can transport _Intent_ as data carrier.
3. Simpilicity in hooking.

### Code

In the first place, we have to look up the Settings's AndroidManifest for what services are available. The hooked service gotta be no impact on app running. Few side effects need taken into consideration. Example is using Settings belong to XiaoMi ROM.

```xml
<service
            android:name="com.android.settings.search.RemoteSearchService"
            android:exported="true"
            android:process=":remote" />
```

_RemoteSearchService_, running in other process, declared able to be called from outside, seems a nice hook point. In Settings' process, hook _onstart_ method because this method has the key paramter, _Intent_, where we get the sent data. 

```java
private void hookService(){
        XposedHelpers.findAndHookMethod(Service.class, "onStart", Intent.class, int.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Service service = (Service) param.thisObject;
                log("Service start " + service.getClass());
                if (service.getClass().toString().contains("RemoteSearchService")){
                    Intent intent = (Intent) param.args[0];
                    String data = intent.getStringExtra(Constant.KEY_CHANGE_PROXY);
                    log("service received data: " + data);
                    if (checkMsg(data)){
                        // There should be ip and port
                        String ip = intent.getStringExtra(Constant.KEY_IP);
                        int port = intent.getIntExtra(Constant.KEY_PORT, 0);

                        changeProxy(ip, port);
                    }
                }
            }
        });
    }
```

There might be confused for you why hook the _Service_ class but not _RemoteSearchService_. It's for simplicity. The app is obfuscated and there might be many methods removed or renamed like _onCreate_,onStart_ and so on. But codes belong to Android OS aren't applied to obfuscation, ensuring the success of hooking.

Back to the codes, fetch data from the first param which is intent. There comes the code of starting service.

```java

// Prevent proxy from being modified too frequently
private static boolean sServiceStarted;

public static void startSettings_Service(final Context context){
        if (sServiceStarted){
            Log.e("MainWork", "Service started, pls start it lately");
            return;
        }

        sServiceStarted = true;


        final Intent intent = new Intent();
        intent.setComponent(new ComponentName(SettingConstant.PACKAGENAME, SettingConstant.HOOKED_SERVICE_PATH));
        intent.putExtra(Constant.KEY_CHANGE_PROXY, Constant.VALUE_CHANGE_PROXY)
                .putExtra(Constant.KEY_IP, "192.168.88.5")
                .putExtra(Constant.KEY_PORT, 8118);

        context.startService(intent);

        new Handler(Looper.getMainLooper()).postDelayed(new Runnable() {
            @Override
            public void run() {
                context.stopService(intent);
                sServiceStarted = false;
            }
        }, 5000);
    }
```

Pay attention that service is started by setting component and there is a need to stop service lately in case services consume too much memory. 

### Conclusion

That might a way for IPC, especially in the third party processes. Hook a service in Google play store and make it handle the process of the job in our own app, which means we have the full contoll of the progress. 
