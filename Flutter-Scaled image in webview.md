# Scale Image to Fit screen in Flutter

### Load Local HTML Content in Webview

The official Webview plugin doesn't come with custom features to Webview, including wide view port, zoom and so on. More importantly, we can't scale the image which is stretched out of screen. 

### Add CSS Style for IMG Label

Add CSS style for IMG at the beginning before the web content.

```HTML
<style>img{display: inline; height: auto; max-width: 100%;}</style>
```

In dart,

```dart
Widget _buildWebContent(ParsedNewObject pno){
    return WebView(
      initialUrl: Uri.dataFromString("<style>img{display: inline; height: auto; max-width: 100%;}</style>"+pno.content, mimeType: 'text/html', encoding: Encoding.getByName('utf-8')).toString(),
      javascriptMode: JavascriptMode.unrestricted,
    );
  }
```

It works like charming.

### REFERENCE
[StackOverflow-Android WebView, Scaling Image to fit the screen](https://stackoverflow.com/a/23656581)