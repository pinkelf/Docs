## Android RN集成步骤
相比于iOS平台，Android打包要复杂一些。主要原因在于React Native For Android本身依赖了大量第三方的库，比如fresco, okhttp等。因此，对于一些已经有一定规模的工程，想要集成React Native框架，只能手工去一个个检查，对齐并补全相关的依赖了。

### 源码获取
为了防止依赖库的冲突，我们的第一不自然就是获取RN的源码。
- 从react native的github地址选取稳定版本的[release](https://github.com/facebook/react-native/releases)，在release note的底部下载源码。通常来说，最新的版本越稳定，性能表现也越优异。
- 解压后会看到ReactCommon和ReactAndroid文件夹，以及编译相关的gradle文件。其他的文件基本与Android打包没什么关系，可以考虑删除。

### ndk处理
RN的ndk部分，依赖了一些常见的库，需要进行适当修改，以防止冲突。同时，我们还可以通过做一些简单的处理，解决翻墙问题并大幅提高打包速度。
- 查看ReactAndroid/build.gradle，有一堆so相关的task。这些task通常分为download和prepare两步。
- 手动下载这些task里所指定的lib，即手工处理download部分的task。目前包括boost, jsc, double-conversion, folly和glog。有些网址网速很慢甚至根本访问不到的问题就解决了，完全可以自己手工去网上找对应版本的so库。
- 将这些lib放到ReactAndroid/build/downloads，然后去掉build.gradle中的download相关task，只留下prepare的task。备份一下这些so库，也能在不小心clean了以后，迅速开始打包，无需重新下载了。
- 使用RN指定的ndk版本进行打包(ndk的兼容性一直堪忧，请务必使用rn官方说明的ndk版本，不是越新越好的)
- 特别要注意的是，rn默认打包为armeabi-v7a和x86。如果你的app不是armeabi-v7a的，那么可不能直接使用。该怎么处理会在后面介绍到。

### 处理Java依赖
Java部分的代码主要集中在ReactAndroid文件夹内，尤其是所有的gradle脚本。0.41版本的依赖关系如下（最新的版本可能某些库有更新）：

````
compile 'javax.inject:javax.inject:1'
compile 'com.android.support:appcompat-v7:23.0.1'
compile 'com.android.support:recyclerview-v7:23.0.1'
compile 'com.facebook.fresco:fresco:0.11.0'
compile 'com.facebook.fresco:imagepipeline-okhttp3:0.11.0'
compile 'com.facebook.soloader:soloader:0.1.0'
compile 'com.fasterxml.jackson.core:jackson-core:2.2.3'
compile 'com.google.code.findbugs:jsr305:3.0.0'
compile 'com.squareup.okhttp3:okhttp:3.4.1'
compile 'com.squareup.okhttp3:okhttp-urlconnection:3.4.1'
compile 'com.squareup.okhttp3:okhttp-ws:3.4.1'
compile 'com.squareup.okio:okio:1.9.0'
compile 'org.webkit:android-jsc:r174650'
````
这就是要重点关心的地方了，改动量也会大一些。主要关注的点如下：
- 翻墙问题。。。老样子，并不是任何地方的网络在任何时间都是通的。方便的话，还是下载下来，然后从在线依赖改为本地依赖吧。
- 接口不一致问题。。。比如fresco的版本，接口就会不一致。再比如okhttp3,直接把okhttp2的名字都换了。
- 包有变化。。。

### 依赖替换
*为了接入基线后的稳定运行，所有基线使用到的库，ReactAndroid的本地依赖都需使用基线的版本，而非网络上的公共版本。*

1. 对于inject, soloader, jackson-core, jsr, jsc, 下载网络上的公共版本，进行配置。由于soloader和jsc是aar包，因此解压缩，拆包，分别将jar和so放入libs中。

2. 对于support-v7,recyclerview-v7,fresco, imagepipeline, okhttp, okhttp-urlconnection, okio, okhttp-ws，则要替换所有的依赖关系为基线提供的本地依赖。

替换过程中，需考虑的问题包括：
- 基线使用了okhttp3和对应的okio版本，ws和url connection还需自行引用。
- 基线的fresco为自行维护，使用基线版本即可。需注意API版本冲突。
- 基线的supportv4和v7也为基线自行维护。需注意API版本冲突。目前使用的aar中去除了对于appcompat-v7的依赖。


### ReactAndroid源码打包
*使用修改后的已经对接了基线的三方库，再对ReactAndroid源码进行正式打包，从而保证集成后运行的稳定性。*

- 修改ReactAndroid工程的build.gradle，去掉线上依赖部分，改为依赖本地工程。
- 根据modify中的内容，对ReactAndroid源码进行修改。
- 打包ReactAndroid工程。
- 获取ReactAndroid-release.aar。
- 由于ndk的冲突，所以需要把aar解压，把其中的armv7改为arm，然后再度压缩，改回为aar。
- 将获得的aar包放入基线的reactandroid工程中。
