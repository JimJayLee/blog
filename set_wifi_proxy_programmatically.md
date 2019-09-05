### Set WIFI Proxy Automatically

For some kind of jobs, we may deal with switching proxy. Usually, proxy configuration is done manually in WIFI settings.  

![](image_url)

### Problem with Automation

It is annoying that you have to make modifications every time changing proxy. 

We tries decompiling the Settings app and failure comes to an end. We encountered servals problem. 

1. Decompile system application takes extra effort. Settings is a system app, so its APK is not bundled with DEX file but ODEX, let alone there comes with VDEX above Android O. That is time-consuming task that you have to convert ODEX to SMALI and then covert it back to DEX. Eventually, convert the DEX into JAR. Furthermore, during the conversion, it may cause any files damaged.

2. Settings application is customized by OEM. What it means is if we want or need to switch to other phones, we have to do the decompilation again. Besides, OEM-customize apps are often obfuscated. Reading and understanding the code is a terrible job. 


Therefore, we come up with some ways to do it automatically.

#### Simulate manipulations by ADB Shell(*NEED ROOT*)

We can send commands to ADB Shell embedded in Android system to manipulate phone. For navigations, we can use _uiautomator_ to dump view info for analyzing what screen we are at. 

##### For more details

1. Execute ADB commands, note that device has to be rooted:

```java
public static void execCommand(String cmd){
        try {
            Process process = Runtime.getRuntime().exec("su");
            OutputStream outputStream =  process.getOutputStream();
            DataOutputStream dataOutputStream = new DataOutputStream(outputStream);
            dataOutputStream.writeBytes(cmd);
            dataOutputStream.flush();
            dataOutputStream.close();
        } catch (IOException e) {

            e.printStackTrace();
        }
    } 


2. Necessary commands:

For getting current view info in the screen:
>adb shell uiautomator dump [path: default is '/sdcard/ui.xml']

Note that commands are executed in Shell environment, 'adb shell' is emitted:
```java
public static void goToHome(){
        AdbTool.execCommand("input keyevent 3");//home
    }

    public static void enter(){
        AdbTool.execCommand("input keyevent 66");
    }

    public static void back(){
        AdbTool.execCommand("input keyevent 4");
    }

    public static void inputString(String text){
        AdbTool.execCommand("input text " + text);
    }

    public static void tap(Point p){
        AdbTool.execCommand("input tap " + p.getX() + SPACE + p.getY());
    }

    public static void swipe(Point from, Point to){
        AdbTool.execCommand("input swipe " + from.getX() + SPACE + from.getY() + SPACE + to.getX() + SPACE + to.getY());
    }

    public static void switchApp(){
        AdbTool.execCommand("input keyevent 187");
    }

    public static void wake(){
        AdbTool.execCommand("input keyevent  82 ");
    }

```

##### How to proceed?

For instance, we have to navigate into WIFI configuration screen. First, after entering into Settings app, we are at main menu screen. Second, dump views of the screen and get the position of WIFI menu button. Once we get the position, send a tap command that tap at the correct button. Lastly, we can move on for setting WIFI proxy. 


#### Set Proxy Programmatically

Maybe we should think why we can not make another "Settings" that directly deal with WIFI configuration? Sure, we can until we figure out how it works. As we mention above, decompilation of system app is a hard work. There are a thing that we missed. Android is open source , and built-in system apps as well. Now, the idea seems feasible. Let's get it start.

##### Fetch Settings Source Code

Given that Android source code is a huge project, what we should grab is only Settings app. Go to [Git repositories on android](https://android.googlesource.com), find _platform/packages/apps/Settings_ and clone it into local storage.

*In China, [this post](https://github.com/foxleezh/AOSP/issues/1) may help*

##### Analysis

For a more reading friendly experience, Source Insight for code reading is highly recommended. Not even it can load larger project, but also searching key word is faster.

1. Pick a key word that navigate to WIFI configuration. In this case, "Proxy Auto-Config" may be fine.

in res\values\arrays.xml

``` xml
<!-- Wi-Fi proxy settings. -->
    <!-- Note that adding/removing/moving the items will need wifi settings code change. -->
    <string-array name="wifi_proxy_settings">
        <!-- No HTTP proxy is used for the current wifi network [CHAR LIMIT=25] -->
        <item>None</item>
        <!-- Manual HTTP proxy settings are used for the current wifi network [CHAR LIMIT=25] -->
        <item>Manual</item>
        <!-- Proxy Auto-Config URL that is used for the current wifi network [CHAR LIMIT=25] -->
        <item>Proxy Auto-Config</item>
    </string-array>
```

2. When we locate the keyword, it's in a string array labeled '_wifi_proxy_settings' which is the resource identifier. Next step is look up the usage of that ID.

3. That ID should lead us to WifiConfiController file. 

in com\android\settings\wifi\WifiConfigController.java

```java
/* These values come from "wifi_proxy_settings" resource array */
    public static final int PROXY_NONE = 0;
    public static final int PROXY_STATIC = 1;
    public static final int PROXY_PAC = 2;

private boolean ipAndProxyFieldsAreValid() {
        mIpAssignment =
                (mIpSettingsSpinner != null
                    && mIpSettingsSpinner.getSelectedItemPosition() == STATIC_IP)
                ? IpAssignment.STATIC
                : IpAssignment.DHCP;

        if (mIpAssignment == IpAssignment.STATIC) {
            mStaticIpConfiguration = new StaticIpConfiguration();
            int result = validateIpConfigFields(mStaticIpConfiguration);
            if (result != 0) {
                return false;
            }
        }

        final int selectedPosition = mProxySettingsSpinner.getSelectedItemPosition();
        mProxySettings = ProxySettings.NONE;
        mHttpProxy = null;
        if (selectedPosition == PROXY_STATIC && mProxyHostView != null) {
            mProxySettings = ProxySettings.STATIC;
            String host = mProxyHostView.getText().toString();
            String portStr = mProxyPortView.getText().toString();
            String exclusionList = mProxyExclusionListView.getText().toString();
            int port = 0;
            int result = 0;
            try {
                port = Integer.parseInt(portStr);
                result = ProxySelector.validate(host, portStr, exclusionList);
            } catch (NumberFormatException e) {
                result = R.string.proxy_error_invalid_port;
            }
            if (result == 0) {
                mHttpProxy = new ProxyInfo(host, port, exclusionList);
            } else {
                return false;
            }
        } else if (selectedPosition == PROXY_PAC && mProxyPacView != null) {
            mProxySettings = ProxySettings.PAC;
            CharSequence uriSequence = mProxyPacView.getText();
            if (TextUtils.isEmpty(uriSequence)) {
                return false;
            }
            Uri uri = Uri.parse(uriSequence.toString());
            if (uri == null) {
                return false;
            }
            mHttpProxy = new ProxyInfo(uri);
        }
        return true;
    }

```

From code snippet above, we know manual proxy setting is flagged as PROXY_STATIC. After validate the input values like hosts and port, store these values in _mHttpProxy_ which is  ProxyInfo instance. Let's move on to what calls this method. Only _isSubmittable_ method does and it seems a function that controls the submit button. That is to say, proxy modifications does not take effect until submit button is being clicked.


```java
boolean isSubmittable() {
        boolean enabled = false;
        boolean passwordInvalid = false;
        if (mPasswordView != null
                && ((mAccessPointSecurity == AccessPoint.SECURITY_WEP
                        && mPasswordView.length() == 0)
                    || (mAccessPointSecurity == AccessPoint.SECURITY_PSK
                           && !isValidPsk(mPasswordView.getText().toString())))) {
            passwordInvalid = true;
        }
        if ((mSsidView != null && mSsidView.length() == 0)
                // If Accesspoint is not saved, apply passwordInvalid check
                || ((mAccessPoint == null || !mAccessPoint.isSaved()) && passwordInvalid
                // If AccessPoint is saved (modifying network) and password is changed, apply
                // Invalid password check
                || mAccessPoint != null && mAccessPoint.isSaved() && passwordInvalid
                    && mPasswordView.length() > 0)) {
            enabled = false;
        } else {
            enabled = ipAndProxyFieldsAreValid();
        }
      // codes deleted 
        return enabled;
    }
```

Who takes control of the submit button? We have to find out what the submit action does. Next, I found _SavedAccessPointsWifiSettings_ which implemented the submit interface. That is exactly what we want!

in com\android\settings\wifi\SavedAccessPointsWifiSettings.java

```java

private WifiManager mWifiManager;

@Override
    public void onSubmit(WifiDialog dialog) {
        mWifiManager.save(dialog.getController().getConfig(), mSaveListener);
    }

```

Fetch a WifiManager instance and get configuration from WiFiConfigController. More importantly, we have known that the proxy info is the _mHttpProxy_ instance. That makes sense. Let's look up _getConfig_ for more details. 


```java
public WifiConfiguration getConfig() {
        if (mMode == WifiConfigUiBase.MODE_VIEW) {
            return null;
        }

        WifiConfiguration config = new WifiConfiguration();

        if (mAccessPoint == null) {
            config.SSID = AccessPoint.convertToQuotedString(
                    mSsidView.getText().toString());
            // If the user adds a network manually, assume that it is hidden.
            config.hiddenSSID = mHiddenSettingsSpinner.getSelectedItemPosition() == HIDDEN_NETWORK;
        } else if (!mAccessPoint.isSaved()) {
            config.SSID = AccessPoint.convertToQuotedString(
                    mAccessPoint.getSsidStr());
        } else {
            config.networkId = mAccessPoint.getConfig().networkId;
            config.hiddenSSID = mAccessPoint.getConfig().hiddenSSID;
        }

        config.shared = mSharedCheckBox.isChecked();

        // codes deleted

        config.setIpConfiguration(
                new IpConfiguration(mIpAssignment, mProxySettings,
                                    mStaticIpConfiguration, mHttpProxy));
        if (mMeteredSettingsSpinner != null) {
            config.meteredOverride = mMeteredSettingsSpinner.getSelectedItemPosition();
        }

        return config;
    }
```

##### How to proceed?

From all analysis of code above, we know _ProxyInfo_ stores the proxy information and _WifiManager.save_ makes the proxy work. Therefore, we can make up a ProxyInfo that contains information we set. And invoke _save_ method to take effect.

Before we start, have a glance at methods we need.

```java
public void save(WifiConfiguration config, ActionListener listener) {
        if (config == null) throw new IllegalArgumentException("config cannot be null");
        getChannel().sendMessage(SAVE_NETWORK, 0, putListener(listener), config);
    }
```

_save_ method takes two parameters. A WifiConfiguration instance and an ActionListener instance which can be null. Note that this method is flagged *Hide* which means can not be  called directly, but it does not matter with reflection. Fortunately, we can fetch available WifiConfigurations by call _getConfiguredNetworks_ with WifiManager. Note that _getConfiguredNetworks_ return a list of WifiConfiguration, so we have filter out the one connected by device. Finally, pass new proxy info as a parameter in _setHttpProxy_ calling. 

##### Final code

```java
try {
                Class<?>[] classes = new Class[]{String.class, int.class, String.class};
                ProxyInfo proxyInfo = (ProxyInfo) XposedHelpers.newInstance(ProxyInfo.class, classes, "192.168.88.51", 12111, "");
                WifiManager wifiManager = (WifiManager) activity.getApplicationContext().getSystemService(Context.WIFI_SERVICE);

                List<WifiConfiguration> wifiConfigurationList = wifiManager.getConfiguredNetworks();

                for (WifiConfiguration wifiConfiguration:
                     wifiConfigurationList) {
                    if(getCurrentWifiSSID(activity.getApplicationContext()).equalsIgnoreCase(wifiConfiguration.SSID)){
                        wifiConfiguration.setHttpProxy(proxyInfo);
                        XposedHelpers.callMethod(wifiManager, "save", wifiConfiguration, null);
                    }
                }

                log("Hook wifi", "success");
            } catch (Exception e){
                e.printStackTrace();
                log("Hook wifi", "FAILED " + e.getLocalizedMessage());
            }
```

For convenience, I use Xpose for reflections. It is same as Java reflection. 

```java
private String getCurrentWifiSSID(Context context){
        String ssid = "";

        ConnectivityManager connectivityManager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        if (connectivityManager.getActiveNetworkInfo().isConnected()){
            WifiManager wifiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
            WifiInfo wifiInfo = wifiManager.getConnectionInfo();
            if (wifiInfo != null && !TextUtils.isEmpty(wifiInfo.getSSID())){
                ssid = wifiInfo.getSSID();
            }
        }
        return ssid;
    }
```

Helper method for getting current connected Wifi info.



###### Permission Problem

By running the code, error involves permission comes up. We are modifying Wifi Configuration which is not allowed in a normal application even with _android.Manifest.permission.OVERRIDE_WIFI_CONFIG_ declared. That is to say, the app has to be system-level.

```java
java.lang.SecurityException: App not allowed to read or update stored WiFi Ap config (uid = 1000)
    at android.os.Parcel.readException(Parcel.java:2004)
    at android.os.Parcel.readException(Parcel.java:1950)
    at android.net.wifi.IWifiManager$Stub$Proxy.getWifiApConfiguration(IWifiManager.java:1769)
    at android.net.wifi.WifiManager.getWifiApConfiguration(WifiManager.java:2221)
    at com.android.settings.TetherSettings.initWifiTethering(TetherSettings.java:205)
    at com.android.settings.TetherSettings.onCreate(TetherSettings.java:171)
```

There are two way to solve this problem:

1. Make app be a system app.
2. Let a system app run that pieces of code.

In the end, I hook the Setting app and let it make the modification. It works perfect if you got Xposed installed.