### Interaction with Wechat Game Program JS scripts
This is about how to make communication between JS SDK and Unity C# scripts.

#### Calling JavaScript functions from Unity scripts
First import WeXin runtime SDK and your JS SDK into Unity project, create a file with *.jslib* extension under a Plugins subfolder in Assets folder. The plugin file needs to have a syntax like as follow:

```javascript
mergeInto(LibraryManager.library, {
  WxLogin: function(){

    GameGlobal.YOUR_SDK_MODULAR_NAME.sdkLogin((data)=>{
      
      console.log(data);
      if(data.resultCode == 1){
        console.log("sdkLogin success");
        GameGlobal.Module.SendMessage("LoginUI","WxLogin",JSON.stringify(data));
      } else{
        console.log("sdkLogin-failed");
        GameGlobal.Module.SendMessage("LoginUI","LoginFailure",JSON.stringify(data));
      }
    });
  },

  WxPay: function(pid, pname, sid, sname, rid, rname,amount,currencyCode,level,extra) {
    let data = {
      productId: UTF8ToString(pid),
      productName: UTF8ToString(pname),
      serverId:UTF8ToString(sid),
      serverName:UTF8ToString(sname),
      roleId:UTF8ToString(rid),
      roleName:UTF8ToString(rname),
      amount:UTF8ToString(amount),
      currencyCode:UTF8ToString(currencyCode),
      extra:UTF8ToString(extra),
      level:UTF8ToString(level)
    };
    GameGlobal.JS_SDK_MODULAR_NAME.sdkPay(data,(response)=>{
      console.log("WxPay success");
      response.data = UTF8ToString(pid);
      GameGlobal.Module.SendMessage("PayUI","wxPayCallback",JSON.stringify(response));
    });
  },
});
```
Note that in WX Runtime SDK folder, just import your sdk as any name in game.js file. Then you can invoke JS functions with GameGlobal.SDK_NAME. BTW, the string parameters passed from C# is not compatible with JS, and they have to be converted by UTF8ToString().

Finally call these functions from C# scripts:
```C#
using UnityEngine;
using System.Runtime.InteropServices;

public class NewBehaviourScript : MonoBehaviour {

    [DllImport("__Internal")]
    private static extern void WxLogin();

    [DllImport("__Internal")]
    private static extern void WxPay(string pid, string pname, string sname, string rid,string rname, string amount, string currencyCode, string level, string extra);

    void Start() {
        // Call functions here
    }
}
```

### Calling Unity scripts functions from Javascript
Sending data from JS to Unity is implemented in jslib file above.
```js
GameGlobal.Module.SendMessage("LoginUI","WxLogin",JSON.stringify(data));
```
Where *LoginUI* is the name of an object in your scene; *WxLogin* is the name of a method in the script attached to the object *LoginUI*; finally you can send some values like string, int or leave it empty.
