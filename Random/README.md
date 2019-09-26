# Lifesaving Tips

### AndroidManifest Merging in Eclipse

In Eclipse, importing an Android project as library is a thing we often do. But I encountered a problem with building APK, and found that AndroidManifest stays the same as the  main project's. It's obvious that ADT doesn't merge the AndroidManifest files.

### How to Solve that?

Go to the *project.properties*, which belongs to the project.   
Add a line of code below:
``` xml
manifestmerger.enabled=true
```

*Note that minSdkVersion and targetSdkVersion have to be the same in all of AndroidManifest*


### R.java Not Found Error

It's annoying as heck when encountering this error:

```
java.lang.NoClassDefFoundError: Failed resolution of: Lcom/google/android/gms/common/R$string
```

It happens when you reference some dependencies bundled with resources. Nothing is wrong when the dependency is AAR, but Jar. That's to say, you are using Ecliplse.

### How to Slove that?

In eclipse, import dependency as library. In Android Studio, just go with AAR!


### Flutter-CupertinoAlertDialog Doesn't Move Upwards

```dart
void _showDialog() async{
    await showCupertinoDialog(
      context: _navigatorKey.currentState.overlay.context,
      builder: (context){
        return CupertinoAlertDialog(
          title: const Text('Remove Account?'),
	  content: CupertinoTextField(placeholder:"example")
          actions: <Widget>[
            CupertinoDialogAction(
              child: const Text('Remove'),
              isDestructiveAction: true,
              onPressed: () {
                PlatformUtil.removeAccount();
                Navigator.pop(context);
              },
            ),
            CupertinoDialogAction(
              child: const Text('Cancel'),
              isDefaultAction: true,
              onPressed: () => Navigator.pop(context),
            ),
          ],
        );
      }
    );
```

When there are text fields embedded in a cupertino style dialo calling up keyboard, the dialog is overlapped by keyboard. It is proven unsolved in Flutter official repo.

### How to solove?
Listen for the keyboard popup.

```dart
class SafeDialogWrapper extends StatelessWidget {

  final Widget child;

  SafeDialogWrapper({Key key, this.child}): super(key: key);

  @override
  Widget build(BuildContext context) {
    var mediaQuery = MediaQuery.of(context);
    return AnimatedContainer(
      padding: mediaQuery.viewInsets,
      duration: const Duration(milliseconds: 150),
      child: child,
    );
  }
}
```

Wrap the dialog with _SafeDialogWrapper_.



### CupertinoAlert Divider Color
The color of CupretinoAlertDialog is not obvious or apparent. It is said that this bug was fixed in the lastest version. Fortunately, I happened to fix it by replacing _showCupertinoDialog_ with _showDialog_. That's all done. Can see divider in dialog.
