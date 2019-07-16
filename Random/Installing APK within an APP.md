### Installing APK within an APP

### Requirement

* Permissions
``` xml
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/> 
``` 

_Note that above Android 0, the REQUEST_INSTALL_PACKAGES permission must be granted for the action. Make sure check and request the permission if it's not granted._

* Proveder

1. In AndroidManifest.xml, and replace application ID with yours

```xml

	<provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="${application ID}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
	</provider>
```

2. Create a file named file_paths(as specified above ) in xml file holder(if do not exist, new one)

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <cache-path
        name="cache"
        path=""/>
</paths>

_Note that the file means that you declare the APK file locates at the Cache files._

### Fire the Action

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
								intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
								intent.setDataAndType(Uri.fromFile(file),"application/vnd.android.package-archive");

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
	Uri apkUri = FileProvider.getUriForFile(context, "com.eskyfun.demo.fileprovider", file);							
	intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
	intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
	intent.setDataAndType(apkUri, "application/vnd.android.package-archive");
} else {
	intent.setDataAndType(Uri.fromFile(file),"application/vnd.android.package-archive");
}

context.startActivity(intent);

_Note that make sure all permissions granted._