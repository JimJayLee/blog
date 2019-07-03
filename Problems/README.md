# AndroidManifest Merging in Eclipse

In Eclipse, importing an Android project as library is a thing we often do. But I encountered a problem with building APK, and found that AndroidManifest stays the same as the  main project's. It's obvious that ADT doesn't merge the AndroidManifest files.

### How to Solve that?

Go to the *project.properties*, which belongs to the project.   
Add a line of code below:
``` xml
manifestmerger.enabled=true
```

*Note that minSdkVersion and targetSdkVersion have to be the same in all of AndroidManifest*