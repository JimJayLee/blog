
## Rust编译Android NDK总结

环境配置

+ 安装Rust以及_cargo_(Rust的包管理工具)
```
curl https://sh.rustup.rs -sSf | sh
```
+ 编译NDK的独立工具链，在android ndk的路径下找到执行脚本，四种架构都需要执行：
```
/Users/~/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 26 --arch arm64 --install-dir ~/Downloads/NDK/arm64
/Users/~/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 26 --arch arm --install-dir ~/Downloads/NDK/arm
/Users/~/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 26 --arch x86 --install-dir ~/Downloads/NDK/x86
/Users/~/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 26 --arch x86_64 --install-dir ~/Downloads/NDK/x86_64
```
+ 新建__cargo-config.toml__文件，添加以下代码，并把路径换成独立工具链的路径。这是因为需要告诉_cargo_怎么去找到工具链进行编译
	```
	[target.aarch64-linux-android]
	ar = "{PATH}/NDK/arm64/bin/aarch64-linux-android-ar"
	linker = "{PATH}/NDK/arm64/bin/aarch64-linux-android-clang"

	[target.armv7-linux-androideabi]
	ar = "{PATH}/NDK/arm/bin/arm-linux-androideabi-ar"
	linker = "{PATH}/NDK/arm/bin/arm-linux-androideabi-clang"

	[target.i686-linux-android]
	ar = "{PATH}/NDK/x86/bin/i686-linux-android-ar"
	linker = "{PATH}/NDK/x86/bin/i686-linux-android-clang"

	[target.x86_64-linux-android]
	ar = "{PATH}/NDK/x86_64/bin/x86_64-linux-android-ar"
	linker = "{PATH}/NDK/x86_64/bin/x86_64-linux-android-clang"
	```
+ 将__cargo-config.toml__文件复制到特定位置
```
cp cargo-config.toml ~/.cargo/config
```
+ 将我们需要编译的架构加入_rustup_，才能交叉编译
```
rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android
```
+ 最后就可以编写Rust代码了～

Rust与Java交互

+ 与NDK使用C++编写一致，Java调用Rust的Jni接口使用同样的方法命名格式，Java+方法路径名以及使用下划线分割
```rust
pub unsafe extern fn Java_com_abc_network_HttpRequest_post(env: JNIEnv, _: JClass, java_url: JString, java_param: JString) -> jstring
```
+ Rust已经有一个Jni的库，封装好很多 Java的数据类型，比如_JInt_、_JString_等，可以直接接收Java的数据。但是我们仍一些了解Rust的数据类型和Java的数据类型怎么互转？
	```rust
	let to_java_int = 0 as jint;
	let to_java_string: JString = env.new_string(JNIString::from("rust_string")).unwrap();
	let to_java_value = JValue::from(0 as jint); # 反射调用java方法时传递的参数都是JValue
	let jstring_to_jvalue = JValue::from(JObject::from(to_java_string));

	let jvalue_to_jstring = JString::from(jstring_to_jvalue.l().unwrap())
	# 先转成JavaStr，再转&Str，最后String
	let jstring_to_rust_string = String::from(env.get_string(jvalue_to_jstring).unwrap().to_str().unwrap())
	# 第二种方法，先转成指针，再转成&Str
	let p = env.get_string(jvalue_to_jstring).unwrap().as_ptr();
	let p_str = unsafe{ CStr::from_ptr(p) }.to_str().unwrap();
	# JString也可以从指针转过来
	let pointer: *mut c_char = CString::new("Hello ".to_owned()).unwrap().into_raw()
	let rust_string_to_jstring = env.new_string(CString::from_raw(pointer).to_str().unwrap())
	```
+ Rust调用Java代码只能使用反射，跟C++类似
```java
    let atClass = env.find_class("android/app/ActivityThread").unwrap();
    let application = env.call_static_method(
        atClass,
        "currentApplication",
        "()Landroid/app/Application;",
        &[]).unwrap();

    let path = env.call_method(
        application.l().unwrap(),
        "getPackageCodePath",
        "()Ljava/lang/String;",
        &[]
    ).unwrap();
```


编译问题总结

1. 使用_Reqwest_网络库编译发生错误
```
error: failed to run custom build command for `openssl-sys v0.9.58`
```
解析：这是_Reqwest_依赖的一个库，_openssl-sys_。通过Github issue的了解，是因为这个网络库并没有为android平台进行跨平台编译适配。后续通过更换其他网络，仍然存在这个问题，这是因为大部分网络库都依赖是此_openssl-sys_库。但是有人提出可以使用**rust_cross**解决。这个Cross是使用docker进行跨平台编译，完美解决这个Mac系统下无法编译Android库的问题。
2. 将编译工具切换到**corss**，可以顺利编译出来我们需要的so
```
 cross build --target aarch64-linux-android --release
 cross build --target armv7-linux-androideabi --release
 cross build --target x86_64-linux-android --release
```
