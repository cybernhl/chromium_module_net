# Chromium Module -net -for Android

This is Ref from  [Building Chromium for Android] &  [懒人chromium net android移植指南] 


## System requirements

* A 64-bit Intel/AMD machine running Linux with at least 8GB of RAM. More than 16GB is highly recommended.
* At least 100GB of free disk space.
* You must have Git and Python installed already.
* You can run Liunx at docker

Most development is done on Ubuntu. Other distros may or may not work;
see the [Linux instructions](linux/build_instructions.md) for some suggestions.

Building the Android client on Windows or Mac is not supported and doesn't work.(because LLVM !!!)

## Prepare Environment

1-1. prepare relation tools : 

```shell
apt update
apt-cache policy lsb-release
apt update
apt-get install openjdk-8-jdk vim curl wget python sudo  git base-files bsdutils e2fsprogs fdisk libblkid1 libcom-err2 libext2fs2 libfdisk1 libgcrypt20 libgnutls30 libmount1 libsmartcols1 libss2 libsystemd0 libudev1 libuuid1 mount util-linux  lsb-release
```
if not superuer
```shell
~ % sudo apt update
~ % sudo apt-cache policy lsb-release
~ % sudo apt update
~ % sudo apt-get install openjdk-8-jdk vim curl wget python sudo  git base-files bsdutils e2fsprogs fdisk libblkid1 libcom-err2 libext2fs2 libfdisk1 libgcrypt20 libgnutls30 libmount1 libsmartcols1 libss2 libsystemd0 libudev1 libuuid1 mount util-linux  lsb-release
```
1-2. prepare relation tools (Setting up a build environment using Docker) : 
 Install Docker
 
MAC  -- [Docker Desktop for Mac]

Windows - [Docker Desktop for Windows] or [WSL (Windows Subsystem for Linux)] 
 
get & run Linux 

PS.  host_os_share_folder_name is host-os folder . use docker command mount to folder_name_at_docker 
PS. if run at Docker use { folder_name_at_docker } let you easy get builded files & folders

```shell
~ % docker run -v ${PWD}/host_os_share_folder_name:/folder_name_at_docker -d ubuntu:18.04 bash -c 'while true; do sleep 5; done'
~ % docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
062d6840435a        ubuntu:18.04        "bash -c 'while true…"   4 hours ago         Up 4 hours                              wonderful_nash

~ %  docker exec -it 062d6840435a bash
root@062d6840435a: apt update
root@062d6840435a: apt-cache policy lsb-release
root@062d6840435a: apt update
root@062d6840435a: apt-get install openjdk-8-jdk vim curl wget python sudo  git base-files bsdutils e2fsprogs fdisk libblkid1 libcom-err2 libext2fs2 libfdisk1 libgcrypt20 libgnutls30 libmount1 libsmartcols1 libss2 libsystemd0 libudev1 libuuid1 mount util-linux  lsb-release
```

## Download & Install depot\_tools

Clone the `depot_tools` repository:

```shell
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

Add `depot_tools` to the end of your PATH (you will probably want to put this
in your `~/.bashrc` or `~/.zshrc`). Assuming you cloned `depot_tools`
to `/path/to/depot_tools`:

```shell
export PATH="$PATH:/path/to/depot_tools"
```
if run at Docker you must move to  { folder_name_at_docker } let you easy get builded files & folders

```shell
cd folder_name_at_docker
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:/folder_name_at_docker/depot_tools"
```

## Get Chromium Project code

Create a `chromium` directory for the checkout and change to it (you can call
this whatever you like and put it wherever you like, as
long as the full path has no spaces):

```shell
mkdir ~/chromium && cd ~/chromium
fetch --nohooks android
```

If you don't want the full repo history, you can save a lot of time by
adding the `--no-history` flag to `fetch`.

Expect the command to take 30 minutes on even a fast connection, and many
hours on slower ones.

If you've already installed the build dependencies on the machine (from another
checkout, for example), you can omit the `--nohooks` flag and `fetch`
will automatically execute `gclient runhooks` at the end.

When `fetch` completes, it will have created a hidden `.gclient` file and a
directory called `src` in the working directory. The remaining instructions
assume you have switched to the `src` directory:

```shell
cd src
```

### Converting an existing Linux checkout

If you have an existing Linux checkout, you can add Android support by
appending `target_os = ['android']` to your `.gclient` file (in the
directory above `src`):

```shell
echo "target_os = [ 'android' ]" >> ../.gclient
```

Then run `gclient sync` to pull the new Android dependencies:

```shell
gclient sync -D --jobs 32
```

(This is the only difference between `fetch android` and `fetch chromium`.)

### Install additional build dependencies

Once you have checked out the code, run

```shell
echo ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true | sudo debconf-set-selections
./build/install-build-deps-android.sh
```

to get all of the dependencies you need to build on Linux, *plus* all of the
Android-specific dependencies (you need some of the regular Linux dependencies
because an Android build includes a bunch of the Linux tools and utilities).

### Run the hooks

Once you've run `install-build-deps` at least once, you can now run the
Chromium-specific hooks, which will download additional binaries and other
things you might need:

```shell
gclient runhooks
build/linux/sysroot_scripts/install-sysroot.py --arch=i386
build/linux/sysroot_scripts/install-sysroot.py --arch=amd64
```

*Optional*: You can also [install API
keys](https://www.chromium.org/developers/how-tos/api-keys) if you want your
build to talk to some Google services, but this is not necessary for most
development and testing purposes.

## Setting up the build

*Optional*:  tmpfs for the build output to reduce the amount of disk writes required.

```shell
mount -t tmpfs -o size=20G,nr_inodes=40k,mode=1777 tmpfs /path/to/out
```
Chromium uses [Ninja](https://ninja-build.org) as its main build tool along with
a tool called [GN](https://gn.googlesource.com/gn/+/master/docs/quick_start.md)
to generate `.ninja` files. You can create any number of *build directories*
with different configurations. To create a build directory which builds Chrome
for Android, run `gn args out/Default` and edit the file to contain the
following arguments:

```
target_os = "android"
target_cpu = "arm64"  # See "Figuring out target_cpu" below
```
*Advanced* : Create args.gn

  - is_component_build    must  set ture
```
将这个配置项置为true，会使以components声明的targets被编译为动态链接库，否则它们将会被编译为静态库。这里我们需要将net等模块编译为动态链接库，因而将该配置项置为true。
```

  - is_debug  set false  && enable_incremental_javac set false
```
is_debug被置为false，表示编译非Debug版。在这种情况下，enable_incremental_javac同样要被置为false。否则在执行gn gen out/Default生成 .ninja 文件时会报error
```

  - disable_file_support  set  true  |  disable_ftp_support set true | enable_websockets set false
```
这几个设置主要是为裁剪需要。我们要禁掉chromium net对这几种协议的支持，以减小最终编译出来的so文件的大小。
```

  - use_platform_icu_alternatives  set true
```
这个配置也是为了减小最终的so文件的总大小。ICU相关的几个so文件总和接近2M，通过将use_platform_icu_alternatives置为true，指示不使用Chromium代码库中的ICU。
```

```shell
mkdir -p out/Default

cat <<EOF > out/Default/args.gn
target_os = "android"
target_cpu = "arm64"
is_debug = true
enable_incremental_javac = false
is_java_debug = true
is_component_build = true
is_clang = true
dcheck_always_on = true
symbol_level = 1
v8_use_external_startup_data = true
fieldtrial_testing_like_official_build = true
icu_use_data_file = false
enable_remoting = true
enable_reporting = true
enable_websockets = true
use_sanitizer_coverage=false
android_full_debug=true
is_asan=true
EOF
```
or

```shell
mkdir -p out/Default
touch out/Default/args.gn
vim out/Default/args.gn

target_os = "android"
target_cpu = "arm64"
is_debug = true
enable_incremental_javac = false
is_java_debug = true
is_component_build = true
is_clang = true
dcheck_always_on = true
symbol_level = 1
v8_use_external_startup_data = true
fieldtrial_testing_like_official_build = true
icu_use_data_file = false
enable_remoting = true
enable_reporting = true
enable_websockets = true
use_sanitizer_coverage=false
android_full_debug=true
is_asan=true

```

* You only have to run this once for each new build directory, Ninja will
  update the build files as needed.
* You can replace `Default` with another name, but
  it should be a subdirectory of `out`.
* For other build arguments, including release settings, see [GN build
  configuration](https://www.chromium.org/developers/gn-build-configuration).
  The default will be a debug component build.
* For more info on GN, run `gn help` on the command line or read the
  [quick start guide](https://gn.googlesource.com/gn/+/master/docs/quick_start.md).

Also be aware that some scripts (e.g. `tombstones.py`, `adb_gdb.py`)
require you to set `CHROMIUM_OUTPUT_DIR=out/Default`.

### Figuring out target\_cpu

The value of
[`target_cpu`](https://gn.googlesource.com/gn/+/master/docs/reference.md#target_cpu)
determines what instruction set to use for native code. Given a device (or
emulator), you can determine the correct instruction set with `adb shell getprop
ro.product.cpu.abi`:

| `getprop ro.product.cpu.abi` output | `target_cpu` value |
|-------------------------------------|--------------------|
| `arm64-v8a`                         | `arm64`            |
| `armeabi-v7a`                       | `arm`              |
| `x86`                               | `x86`              |
| `x86_64`                            | `x64`              |

*** promo
`arm` and `x86` may optionally be used instead of `arm64` and `x64` for
non-WebView targets. This is also allowed for Monochrome, but only when not set
as the WebView provider.
***
## GN build configuration (must run to gen ninja files & foldrs)
 
```shell
gn gen out/Default
```
## Clean GN build configuration
 
```shell
gn clean out/Default
```

## Build Chromium module 

Build Chromium module with Ninja using the command:

PS. This project  gen "net" module
```shell
ninja -C out/Default net
```
这个命令会编译net模块，及其依赖的所有模块，包括base，crypto，boringssl，protobuf，url等。看一下我们编译的成果：

```shell
$ ls -alh out/Default/ | grep so
-rwxr-xr-x  1 root root  13M Mar 20 13:50 libbase.cr.so
-rwxr-xr-x  1 root root 904K Mar 20 13:56 libbase_i18n.cr.so
-rwxr-xr-x  1 root root 4.6M Mar 20 13:50 libboringssl.cr.so
-rwxr-xr-x  1 root root 1.9M Mar 20 13:39 libc++.cr.so
-rwxr-xr-x  1 root root 412K Mar 20 13:57 libchrome_zlib.cr.so
-rwxr-xr-x  1 root root 309K Mar 20 13:51 libcrcrypto.cr.so
-rwxr-xr-x  1 root root 6.5M Mar 20 13:56 libicui18n.cr.so
-rwxr-xr-x  1 root root  11M Mar 20 13:56 libicuuc.cr.so
-rwxr-xr-x  1 root root  37M Mar 20 14:40 libnet.cr.so
-rwxr-xr-x  1 root root 1.7M Mar 20 13:57 libprotobuf_lite.cr.so
-rwxr-xr-x  1 root root 614K Mar 20 13:57 liburl.cr.so
```

# 使用Chromium net

## 将Chromium net导入Android应用 

在我们Android 工程(MyApplication)的app模块的jni目录下
为chromium创建文件夹
app/src/main/jni/third_party/chromium/libs和app/src/main/jni/third_party/chromium/include，
分别用于存放我们编译出来的共享库和net等模块导出的头文件及这些头文件include的其它头文件。

这里我们将编译出来的所有so文件拷贝到
app/src/main/jni/third_party/chromium/libs/armeabi
app/src/main/jni/third_party/chromium/libs/armeabi-v7a目录下：

```shell
cp out/Default/*.so ~/MyApplication/app/src/main/jni/third_party/chromium/libs/armeabi/
cp out/Default/*.so ~/MyApplication/app/src/main/jni/third_party/chromium/libs/armeabi-v7a/
 ```
 ## 提取导出头文件 
 
 为了使用net模块提供的API，不可避免地要将net导出的头文件引入我们的项目。要做到这些，需要从chromium工程提取net导出的头文件。不像许多其它的C/C++项目，源代码文件、私有头文件及导出头文件存放的位置被很好地做了区隔，chromium各个模块的头文件和源代码文件都是放在一起的。这给我们提取导出头文件的工作带来了一点麻烦。

好在有gn工具。gn工具提供的desc命令（参考 [GN的使用 - GN工具  ] 一文）的输出有如下这样两段：
 ```shell
$ gn desc out/Default/ net
Target //net:net
Type: shared_library
Toolchain: //build/toolchain/android:arm
......
sources
  //net/base/address_family.cc
  //net/base/address_family.h
......
 
public
  [All headers listed in the sources are public.]
 ```

 如果看不清楚 可以使用 >> 輸出至文字檔

```shell
$ gn desc out/Default/ net >> /folder_name_at_docker/chromium_net_module_relation.txt
 ```
 
 我们可以据此编写脚本提取net模块的头文件。
 
 脚本传入[chromium代码库的src目录路径]，[输出目录的路径]，[模块名]，及[保存头文件的目标目录路径]作为参数，以提取头文件，[保存头文件的目标目录路径]参数缺失时默认使用当前目录，如以下Python腳本：
 
 ```shell
$ python chromium_mod_headers_extracter.py /folder_name_at_docker/chromium/src  ./out/Default net /folder_name_at_docker/chromium/module/net
$ python chromium_mod_headers_extracter.py /folder_name_at_docker/chromium/src  ./out/Default net /folder_name_at_docker/chromium/module/net
$ python chromium_mod_headers_extracter.py /folder_name_at_docker/chromium/src  ./out/Default net /folder_name_at_docker/chromium/module/net
 ```
利用我们的脚本，提取net、base和url这三个模块导出的头文件。
这里一并将该脚本(chromium_mod_headers_extracter.py)的完整内容贴出来供大家参考：

 
 ```shell
#!/usr/bin/env python
 
import os
import shutil
import sys
 
def print_usage_and_exit():
    print sys.argv[0] + " [chromium_src_root]" + "[out_dir]" + " [target_name]" + " [targetroot]"
    exit(1)
 
def copy_file(src_file_path, target_file_path):
    if os.path.exists(target_file_path):
        return
    if not os.path.exists(src_file_path):
        return
    target_dir_path = os.path.dirname(target_file_path)
    if not os.path.exists(target_dir_path):
        os.makedirs(target_dir_path)
 
    shutil.copy(src_file_path, target_dir_path)
    print "shutil.copy " + " [src_file_path]" + "[target_file_path]"
 
def copy_all_files(source_dir, all_files, target_dir):
    for one_file in all_files:
        source_path = source_dir + os.path.sep + one_file
        target_path = target_dir + os.path.sep + one_file
        copy_file(source_path, target_path)
 
if __name__ == "__main__":
    if len(sys.argv) < 4 or len(sys.argv) > 5:
        print_usage_and_exit()
    chromium_src_root = sys.argv[1]
    out_dir = sys.argv[2]
    target_name = sys.argv[3]
    target_root_path = sys.argv[4]

    target_root_path = os.path.abspath(target_root_path)
 
    os.chdir(chromium_src_root)
 
    cmd = "gn desc " + out_dir + " " + target_name
    outputs = os.popen(cmd).readlines()
    source_start = False
    all_headers = []
 
    public_start = False
    public_headers = []
 
    for output_line in outputs:
        output_line = output_line.strip()
        if output_line.startswith("sources"):
            source_start = True
            print "source_start"
            continue
        elif source_start and len(output_line) == 0:
            source_start = False
            continue
        elif source_start and output_line.endswith(".h"):
            output_line = output_line[1:]
            all_headers.append(output_line)
        elif output_line == "public":
            public_start = True
            continue
        elif public_start and len(output_line) == 0:
            public_start = False
            continue
        elif public_start:
            public_headers.append(output_line)
 
    if len(public_headers) == 1:
        public_headers = all_headers
    if len(public_headers) > 1:
        copy_all_files(chromium_src_root, public_headers, target_dir=target_root_path)
 ```
 此外，前面的提取过程会遗漏一些必须的头文件。主要是如下几个：
 
  
 ```shell
base/callback_forward.h
base/message_loop/timer_slack.h
base/files/file.h
net/cert/cert_status_flags_list.h
net/cert/cert_type.h
net/base/privacy_mode.h
net/websockets/websocket_event_interface.h
net/quic/quic_alarm_factory.h 
 ```
 对于这些文件，我们直接从chromium的代码库拷贝到我们的工程中对应的位置即可。

我们还需要引入chromium的build配置头文件build/build_config.h。直接将chromium代码库中的对应文件拷贝过来，放到对应的位置。

将app/src/main/jni/third_party/chromium/include/base/gtest_prod_util.h文件中对testing/gtest/include/gtest/gtest_prod.h的include注释掉，同时修改FRIEND_TEST_ALL_PREFIXES宏的定义为：

  
 ```shell
#if 0
#define FRIEND_TEST_ALL_PREFIXES(test_case_name, test_name) \
  FRIEND_TEST(test_case_name, test_name); \
  FRIEND_TEST(test_case_name, DISABLED_##test_name); \
  FRIEND_TEST(test_case_name, FLAKY_##test_name)
#else
#define FRIEND_TEST_ALL_PREFIXES(test_case_name, test_name)
#endif 
 ```
这样就可以注释掉类定义中专门为gtest插入的代码。 
 
## Chromium net的简单使用
 
 参照chromium/src/net/tools/get_server_time/get_server_time.cc的代码，来编写简单的demo程序。
首先是JNI的Java层代码：
  
 ```shell
public class NetUtils {
    static {
        System.loadLibrary("neteasenet");
    }
    private static native void nativeSendRequest(String url);
 
    public static void sendRequest(String url) {
        nativeSendRequest(url);
    }
}
 ``` 
 然后是JNI的native实现，app/src/main/jni/src/NetJni.cpp：
 
  ```shell
//
// Created by hanpfei0306 on 16-8-4.
//
 
#include <stdio.h>
#include <net/base/network_delegate_impl.h>
 
#include "jni.h"
 
#include "base/at_exit.h"
#include "base/json/json_writer.h"
#include "base/message_loop/message_loop.h"
#include "base/memory/ptr_util.h"
#include "base/run_loop.h"
#include "base/values.h"
#include "net/http/http_response_headers.h"
#include "net/proxy/proxy_config_service_fixed.h"
#include "net/url_request/url_fetcher.h"
#include "net/url_request/url_fetcher_delegate.h"
#include "net/url_request/url_request_context.h"
#include "net/url_request/url_request_context_builder.h"
#include "net/url_request/url_request_context_getter.h"
#include "net/url_request/url_request.h"
 
#include "JNIHelper.h"
 
#define TAG "NetUtils"
 
// Simply quits the current message loop when finished.  Used to make
// URLFetcher synchronous.
class QuitDelegate : public net::URLFetcherDelegate {
public:
    QuitDelegate() {}
 
    ~QuitDelegate() override {}
 
    // net::URLFetcherDelegate implementation.
    void OnURLFetchComplete(const net::URLFetcher* source) override {
        LOGE("OnURLFetchComplete");
        base::MessageLoop::current()->QuitWhenIdle();
        int responseCode = source->GetResponseCode();
 
        const net::URLRequestStatus status = source->GetStatus();
        if (status.status() != net::URLRequestStatus::SUCCESS) {
            LOGW("Request failed with error code: %s", net::ErrorToString(status.error()).c_str());
            return;
        }
 
        const net::HttpResponseHeaders* const headers = source->GetResponseHeaders();
        if (!headers) {
            LOGW("Response does not have any headers");
            return;
        }
        size_t iter = 0;
        std::string header_name;
        std::string date_header;
        while (headers->EnumerateHeaderLines(&iter, &header_name, &date_header)) {
            LOGW("Got %s header: %s\n", header_name.c_str(), date_header.c_str());
        }
 
        std::string responseStr;
        if(!source->GetResponseAsString(&responseStr)) {
            LOGW("Get response as string failed!");
        }
 
        LOGI("Content len = %lld, response code = %d, response = %s",
             source->GetReceivedResponseContentLength(),
             source->GetResponseCode(),
             responseStr.c_str());
    }
 
    void OnURLFetchDownloadProgress(const net::URLFetcher* source,
                                    int64_t current,
                                    int64_t total) override {
        LOGE("OnURLFetchDownloadProgress");
    }
 
    void OnURLFetchUploadProgress(const net::URLFetcher* source,
                                  int64_t current,
                                  int64_t total) override {
        LOGE("OnURLFetchUploadProgress");
    }
 
private:
    DISALLOW_COPY_AND_ASSIGN(QuitDelegate);
};
 
// NetLog::ThreadSafeObserver implementation that simply prints events
// to the logs.
class PrintingLogObserver : public net::NetLog::ThreadSafeObserver {
public:
    PrintingLogObserver() {}
 
    ~PrintingLogObserver() override {
        // This is guaranteed to be safe as this program is single threaded.
        net_log()->DeprecatedRemoveObserver(this);
    }
 
    // NetLog::ThreadSafeObserver implementation:
    void OnAddEntry(const net::NetLog::Entry& entry) override {
        // The log level of the entry is unknown, so just assume it maps
        // to VLOG(1).
        const char* const source_type = net::NetLog::SourceTypeToString(entry.source().type);
        const char* const event_type = net::NetLog::EventTypeToString(entry.type());
        const char* const event_phase = net::NetLog::EventPhaseToString(entry.phase());
        std::unique_ptr<base::Value> params(entry.ParametersToValue());
        std::string params_str;
        if (params.get()) {
            base::JSONWriter::Write(*params, &params_str);
            params_str.insert(0, ": ");
        }
#ifdef DEBUG_ALL
        LOGI("source_type = %s (id = %u): entry_type = %s : event_phase = %s params_str = %s",
             source_type, entry.source().id, event_type, event_phase, params_str.c_str());
#endif
    }
 
private:
    DISALLOW_COPY_AND_ASSIGN(PrintingLogObserver);
};
 
// Builds a URLRequestContext assuming there's only a single loop.
static std::unique_ptr<net::URLRequestContext> BuildURLRequestContext(net::NetLog *net_log) {
    net::URLRequestContextBuilder builder;
    builder.set_net_log(net_log);
//#if defined(OS_LINUX)
    // On Linux, use a fixed ProxyConfigService, since the default one
  // depends on glib.
  //
  // TODO(akalin): Remove this once http://crbug.com/146421 is fixed.
  builder.set_proxy_config_service(
          base::WrapUnique(new net::ProxyConfigServiceFixed(net::ProxyConfig())));
//#endif
    std::unique_ptr<net::URLRequestContext> context(builder.Build());
    context->set_net_log(net_log);
    return context;
}
 
static void NetUtils_nativeSendRequest(JNIEnv* env, jclass, jstring javaUrl) {
    const char* native_url = env->GetStringUTFChars(javaUrl, NULL);
    LOGW("Url: %s", native_url);
    base::AtExitManager exit_manager;
    LOGW("Url: %s", native_url);
 
    GURL url(native_url);
    if (!url.is_valid() || (url.scheme() != "http" && url.scheme() != "https")) {
        LOGW("Not valid url: %s", native_url);
        return;
    }
    LOGW("Url: %s", native_url);
 
    base::MessageLoopForIO main_loop;
 
    QuitDelegate delegate;
    std::unique_ptr<net::URLFetcher> fetcher =
            net::URLFetcher::Create(url, net::URLFetcher::GET, &delegate);
 
    net::NetLog *net_log = nullptr;
#ifdef DEBUG_ALL
    net_log = new net::NetLog;
    PrintingLogObserver printing_log_observer;
    net_log->DeprecatedAddObserver(&printing_log_observer,
                                  net::NetLogCaptureMode::IncludeSocketBytes());
#endif
 
    std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
    fetcher->SetRequestContext(
            // Since there's only a single thread, there's no need to worry
            // about when the URLRequestContext gets created.
            // The URLFetcher will take a reference on the object, and hence
            // implicitly take ownership.
            new net::TrivialURLRequestContextGetter(url_request_context.get(),
                                                    main_loop.task_runner()));
    fetcher->Start();
    // |delegate| quits |main_loop| when the request is done.
    main_loop.Run();
 
    env->ReleaseStringUTFChars(javaUrl, native_url);
}
 
int jniRegisterNativeMethods(JNIEnv* env, const char *classPathName, JNINativeMethod *nativeMethods, jint nMethods) {
    jclass clazz;
    clazz = env->FindClass(classPathName);
    if (clazz == NULL) {
        return JNI_FALSE;
    }
    if (env->RegisterNatives(clazz, nativeMethods, nMethods) < 0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}
 
static JNINativeMethod gNetUtilsMethods[] = {
        NATIVE_METHOD(NetUtils, nativeSendRequest, "(Ljava/lang/String;)V"),
};
 
void register_com_netease_volleydemo_NetUtils(JNIEnv* env) {
    jniRegisterNativeMethods(env, "com/example/hanpfei0306/myapplication/NetUtils",
                             gNetUtilsMethods, NELEM(gNetUtilsMethods));
}
 
// DalvikVM calls this on startup, so we can statically register all our native methods.
jint JNI_OnLoad(JavaVM* vm, void*) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        LOGE("JavaVM::GetEnv() failed");
        abort();
    }
 
    register_com_netease_volleydemo_NetUtils(env);
    return JNI_VERSION_1_6;
} 
 ``` 
 这个文件里，在nativeSendRequest()函数中调用chromium net执行网络请求，获取响应，并打印出响应的headers及content。
 
 ## 配置Gradle
 
 要在Android Studio中使用JNI，还需要对Gralde做一些配置文。这里需要对MyApplication/build.gradle、MyApplication/gradle/wrapper/gradle-wrapper.properties，和MyApplication/app/build.gradle这几个文件做修改。

修改MyApplication/build.gradle文件，最终的内容为：
 
   
 ```shell
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.6.1'
 
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
 
allprojects {
    repositories {
        google()
        jcenter()
    }
}
 
task clean(type: Delete) {
    delete rootProject.buildDir
}
 ```
 
 修改MyApplication/gradle/wrapper/gradle-wrapper.properties在这个文件中配置gradle的版本。，最终的内容为：
 
 ```shell
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-5.6.4-all.zip
```
 修改MyApplication/app/build.gradle文件，最终的内容为：

 ```shell
apply plugin: 'com.android.model.application'
 
model {
    repositories {
        libs(PrebuiltLibraries) {
            chromium_net {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libnet.cr.so")
                }
            }
            chromium_base {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libbase.cr.so")
                }
            }
            chromium_url {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/liburl.cr.so")
                }
            }
        }
    }
 
    android {
        compileSdkVersion 23
        buildToolsVersion "23.0.3"
 
        defaultConfig {
            applicationId "com.example.hanpfei0306.myapplication"
            minSdkVersion.apiLevel 19
            targetSdkVersion.apiLevel 21
            versionCode 1
            versionName "1.0"
        }
 
        ndk {
            moduleName "neteasenet"
            toolchain "clang"
 
            CFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include/'),])
 
            cppFlags.addAll(["-std=gnu++11", ])
            cppFlags.addAll(["-DV8_DEPRECATION_WARNINGS",
                             "-DENABLE_NOTIFICATIONS",
                             "-DENABLE_BROWSER_CDMS",
                             "-DENABLE_PRINTING=1",
                             "-DENABLE_BASIC_PRINTING=1",
                             "-DENABLE_SPELLCHECK=1",
                             "-DUSE_BROWSER_SPELLCHECKER=1",
                             "-DUSE_OPENSSL_CERTS=1",
                             "-DNO_TCMALLOC",
                             "-DUSE_EXTERNAL_POPUP_MENU=1",
                             "-DDISABLE_NACL",
                             "-DENABLE_SUPERVISED_USERS=1",
                             "-DCHROMIUM_BUILD",
                             "-D_FILE_OFFSET_BITS=64",
                             "-DANDROID",
                             "-DHAVE_SYS_UIO_H",
                             "-D__STDC_CONSTANT_MACROS",
                             "-D__STDC_FORMAT_MACROS",
                             "-D_FORTIFY_SOURCE=2",
                             "-DCOMPONENT_BUILD",
                             "-D__GNU_SOURCE=1",
                             "-D_DEBUG",
                             "-DDYNAMIC_ANNOTATIONS_ENABLED=1",
                             "-DWTF_USE_DYNAMIC_ANNOTATIONS=1",
                             "-DDLOPEN_KERBEROS",
                             "-DNET_IMPLEMENTATION",
                             "-DUSE_KERBEROS",
                             "-DENABLE_BUILT_IN_DNS",
                             "-DPOSIX_AVOID_MMAP",
                             "-DENABLE_WEBSOCKETS",
                             "-DGOOGLE_PROTOBUF_NO_RTTI",
                             "-DGOOGLE_PROTOBUF_NO_STATIC_INITIALIZER",
                             "-DHAVE_PTHREAD",
                             "-DPROTOBUF_USE_DLLS",
                             "-DBORINGSSL_SHARED_LIBRARY",
                             "-DU_USING_ICU_NAMESPACE=0",
                             "-DU_ENABLE_DYLOAD=0",
            ])
            cppFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include'), ])
 
            ldLibs.add("android")
            ldLibs.add("log")
            ldLibs.add("z")
            stl "c++_shared"
        }
 
        sources {
            main {
                java {
                    source {
                        srcDir "src/main/java"
                    }
                }
                jni {
                    source {
                        srcDirs = ["src/main/jni",]
                    }
                    dependencies {
                        library 'chromium_base' linkage 'shared'
                        library 'chromium_url' linkage 'shared'
                        library 'chromium_net' linkage 'shared'
                    }
                }
                jniLibs {
                    source {
                        srcDirs =["src/main/jni/third_party/chromium/libs",]
                    }
                }
            }
        }
 
        buildTypes {
            debug {
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
            release {
                minifyEnabled false
                proguardFiles.add(file("proguard-rules.pro"))
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
        }
    }
}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.4.0'
}
 ```
 关键点主要有如下这些：
 
* 为net、base和url这几个模块创建PrebuiltLibraries libs元素，并正确的设置对这些模块的依赖。
* 配置stl为"c++_shared"。
* cppFlags的"-std=gnu++11"选项必不可少。
* buildType下的debug和release，需要给它们ndk的abiFilters添加我们想要支持的ABI，而不是留空，以防止Android Studio为我们编译我们不打算支持的ABI的so，而出现找不到文件的问题。
* CFlags和cppFlags中除了配置头文件搜索路径的那两行之外，其它的内容，主要是从chromium的构建环境中提取的。方法为：

 ```shell
$ gn desc out/Default/ net
Target //net:net
Type: shared_library
Toolchain: //build/toolchain/android:arm
......
cflags
  -fno-strict-aliasing
  --param=ssp-buffer-size=4
  -fstack-protector
  -funwind-tables
  -fPIC
  -pipe
  -ffunction-sections
  -fno-short-enums
  -finline-limit=64
  -march=armv7-a
  -mfloat-abi=softfp
  -mthumb
  -mthumb-interwork
  -mtune=generic-armv7-a
  -fno-tree-sra
  -fno-caller-saves
  -mfpu=neon
  -Wall
  -Werror
  -Wno-psabi
  -Wno-unused-local-typedefs
  -Wno-maybe-uninitialized
  -Wno-missing-field-initializers
  -Wno-unused-parameter
  -Os
  -fomit-frame-pointer
  -fno-ident
  -fdata-sections
  -ffunction-sections
  -g1
  --sysroot=../../../../../../../~/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm
  -fvisibility=hidden
 
cflags_cc
  -fno-threadsafe-statics
  -fvisibility-inlines-hidden
  -std=gnu++11
  -Wno-narrowing
  -fno-rtti
  -isystem../../../../../../../~/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libcxx/include
  -isystem../../../../../../../~/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++abi/libcxxabi/include
  -isystem../../../../../../../~/dev_tools/Android/android-ndk-r12b/sources/android/support/include
  -fno-exceptions
 
......
 
defines
  V8_DEPRECATION_WARNINGS
  ENABLE_NOTIFICATIONS
  ENABLE_BROWSER_CDMS
  ENABLE_PRINTING=1
  ENABLE_BASIC_PRINTING=1
  ENABLE_SPELLCHECK=1
  USE_BROWSER_SPELLCHECKER=1
  USE_OPENSSL_CERTS=1
  NO_TCMALLOC
  USE_EXTERNAL_POPUP_MENU=1
  ENABLE_WEBRTC=1
  DISABLE_NACL
  ENABLE_SUPERVISED_USERS=1
  VIDEO_HOLE=1
  SAFE_BROWSING_DB_REMOTE
  CHROMIUM_BUILD
  ENABLE_MEDIA_ROUTER=1
  ENABLE_WEBVR
  FIELDTRIAL_TESTING_ENABLED
  _FILE_OFFSET_BITS=64
  ANDROID
  HAVE_SYS_UIO_H
  ANDROID_NDK_VERSION=r10e
  __STDC_CONSTANT_MACROS
  __STDC_FORMAT_MACROS
  _FORTIFY_SOURCE=2
  COMPONENT_BUILD
  __GNU_SOURCE=1
  NDEBUG
  NVALGRIND
  DYNAMIC_ANNOTATIONS_ENABLED=0
  DLOPEN_KERBEROS
  NET_IMPLEMENTATION
  USE_KERBEROS
  ENABLE_BUILT_IN_DNS
  POSIX_AVOID_MMAP
  ENABLE_WEBSOCKETS
  GOOGLE_PROTOBUF_NO_RTTI
  GOOGLE_PROTOBUF_NO_STATIC_INITIALIZER
  HAVE_PTHREAD
  PROTOBUF_USE_DLLS
  BORINGSSL_SHARED_LIBRARY
  U_USING_ICU_NAMESPACE=0
  U_ENABLE_DYLOAD=0
  U_NOEXCEPT=
  ICU_UTIL_DATA_IMPL=ICU_UTIL_DATA_FILE
......
libs
  c++_shared
  ~/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a
  c
  atomic
  dl
  m
  log
  unwind
 
lib_dirs
  ~/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a/

 ```


####  Chromium module -- net relation 


 ```shell
root@007407603c9f:/androidchrome/chromium/src# gn desc out/Default/ net >> /androidchrome/chromium_net_module_relation.txt

WARNING at build arg file (use "gn args <out_dir>" to edit):4:28: Build argument has no effect.
enable_incremental_javac = false
                           ^----
The variable "enable_incremental_javac" was set as a build argument
but never appeared in a declare_args() block in any buildfile.

To view all possible args, run "gn args --list <out_dir>"

The build continued as if that argument was unspecified.

Target //net:net
type: shared_library
toolchain: //build/toolchain/android:android_clang_arm

visibility
  *

metadata
  {
    shared_libraries = [
      "//out/Default/libnet.cr.so"
    ]
  }

testonly
  false

check_includes
  true

allow_circular_includes_from
  //net/dns:dns
  //net/dns:dns_client
  //net/dns:host_resolver
  //net/dns:host_resolver_manager
  //net/dns:mdns_client
  //net/dns/public:public
  //net/http:transport_security_state_generated_files
  //net/third_party/quiche:quiche

sources
  //net/base/address_family.cc
  //net/base/address_family.h
  //net/base/address_list.cc
  //net/base/address_list.h
  //net/base/auth.cc
  //net/base/auth.h
  //net/base/completion_once_callback.h
  //net/base/completion_repeating_callback.h
  //net/base/datagram_buffer.cc
  //net/base/datagram_buffer.h
  //net/base/escape.cc
  //net/base/escape.h
  //net/base/features.cc
  //net/base/features.h
  //net/base/hash_value.cc
  //net/base/hash_value.h
  //net/base/host_port_pair.cc
  //net/base/host_port_pair.h
  //net/base/interval.h
  //net/base/interval_set.h
  //net/base/io_buffer.cc
  //net/base/io_buffer.h
  //net/base/ip_address.cc
  //net/base/ip_address.h
  //net/base/ip_endpoint.cc
  //net/base/ip_endpoint.h
  //net/base/load_timing_info.cc
  //net/base/load_timing_info.h
  //net/base/lookup_string_in_fixed_set.cc
  //net/base/lookup_string_in_fixed_set.h
  //net/base/net_error_details.h
  //net/base/net_error_list.h
  //net/base/net_errors.cc
  //net/base/net_errors.h
  //net/base/net_module.cc
  //net/base/net_module.h
  //net/base/net_string_util.h
  //net/base/network_interfaces.cc
  //net/base/network_interfaces.h
  //net/base/parse_number.cc
  //net/base/parse_number.h
  //net/base/port_util.cc
  //net/base/port_util.h
  //net/base/privacy_mode.h
  //net/base/rand_callback.h
  //net/base/registry_controlled_domains/registry_controlled_domain.cc
  //net/base/registry_controlled_domains/registry_controlled_domain.h
  //net/base/sockaddr_storage.cc
  //net/base/sockaddr_storage.h
  //net/base/sys_addrinfo.h
  //net/base/url_util.cc
  //net/base/url_util.h
  //net/cert/asn1_util.cc
  //net/cert/asn1_util.h
  //net/cert/cert_database.cc
  //net/cert/cert_database.h
  //net/cert/cert_status_flags.cc
  //net/cert/cert_status_flags.h
  //net/cert/cert_status_flags_list.h
  //net/cert/cert_verifier.cc
  //net/cert/cert_verifier.h
  //net/cert/cert_verify_result.cc
  //net/cert/cert_verify_result.h
  //net/cert/client_cert_verifier.h
  //net/cert/crl_set.cc
  //net/cert/crl_set.h
  //net/cert/ct_policy_enforcer.cc
  //net/cert/ct_policy_enforcer.h
  //net/cert/ct_policy_status.h
  //net/cert/ct_verifier.h
  //net/cert/ct_verify_result.cc
  //net/cert/ct_verify_result.h
  //net/cert/do_nothing_ct_verifier.cc
  //net/cert/do_nothing_ct_verifier.h
  //net/cert/internal/cert_error_id.cc
  //net/cert/internal/cert_error_id.h
  //net/cert/internal/cert_error_params.cc
  //net/cert/internal/cert_error_params.h
  //net/cert/internal/cert_errors.cc
  //net/cert/internal/cert_errors.h
  //net/cert/internal/cert_issuer_source.h
  //net/cert/internal/cert_issuer_source_aia.cc
  //net/cert/internal/cert_issuer_source_aia.h
  //net/cert/internal/cert_issuer_source_static.cc
  //net/cert/internal/cert_issuer_source_static.h
  //net/cert/internal/certificate_policies.cc
  //net/cert/internal/certificate_policies.h
  //net/cert/internal/common_cert_errors.cc
  //net/cert/internal/common_cert_errors.h
  //net/cert/internal/crl.cc
  //net/cert/internal/crl.h
  //net/cert/internal/extended_key_usage.cc
  //net/cert/internal/extended_key_usage.h
  //net/cert/internal/general_names.cc
  //net/cert/internal/general_names.h
  //net/cert/internal/name_constraints.cc
  //net/cert/internal/name_constraints.h
  //net/cert/internal/ocsp.cc
  //net/cert/internal/ocsp.h
  //net/cert/internal/parse_certificate.cc
  //net/cert/internal/parse_certificate.h
  //net/cert/internal/parse_name.cc
  //net/cert/internal/parse_name.h
  //net/cert/internal/parsed_certificate.cc
  //net/cert/internal/parsed_certificate.h
  //net/cert/internal/path_builder.cc
  //net/cert/internal/path_builder.h
  //net/cert/internal/revocation_checker.cc
  //net/cert/internal/revocation_checker.h
  //net/cert/internal/revocation_util.cc
  //net/cert/internal/revocation_util.h
  //net/cert/internal/signature_algorithm.cc
  //net/cert/internal/signature_algorithm.h
  //net/cert/internal/simple_path_builder_delegate.cc
  //net/cert/internal/simple_path_builder_delegate.h
  //net/cert/internal/trust_store.cc
  //net/cert/internal/trust_store.h
  //net/cert/internal/trust_store_collection.cc
  //net/cert/internal/trust_store_collection.h
  //net/cert/internal/trust_store_in_memory.cc
  //net/cert/internal/trust_store_in_memory.h
  //net/cert/internal/verify_certificate_chain.cc
  //net/cert/internal/verify_certificate_chain.h
  //net/cert/internal/verify_name_match.cc
  //net/cert/internal/verify_name_match.h
  //net/cert/internal/verify_signed_data.cc
  //net/cert/internal/verify_signed_data.h
  //net/cert/ocsp_revocation_status.h
  //net/cert/ocsp_verify_result.cc
  //net/cert/ocsp_verify_result.h
  //net/cert/pem.cc
  //net/cert/pem.h
  //net/cert/sct_status_flags.cc
  //net/cert/sct_status_flags.h
  //net/cert/signed_certificate_timestamp.cc
  //net/cert/signed_certificate_timestamp.h
  //net/cert/signed_certificate_timestamp_and_status.cc
  //net/cert/signed_certificate_timestamp_and_status.h
  //net/cert/signed_tree_head.cc
  //net/cert/signed_tree_head.h
  //net/cert/symantec_certs.cc
  //net/cert/symantec_certs.h
  //net/cert/x509_cert_types.cc
  //net/cert/x509_cert_types.h
  //net/cert/x509_certificate.cc
  //net/cert/x509_certificate.h
  //net/cert/x509_certificate_net_log_param.cc
  //net/cert/x509_certificate_net_log_param.h
  //net/cert/x509_util.cc
  //net/cert/x509_util.h
  //net/der/encode_values.cc
  //net/der/encode_values.h
  //net/der/input.cc
  //net/der/input.h
  //net/der/parse_values.cc
  //net/der/parse_values.h
  //net/der/parser.cc
  //net/der/parser.h
  //net/der/tag.cc
  //net/der/tag.h
  //net/http/hsts_info.h
  //net/http/http_auth_challenge_tokenizer.cc
  //net/http/http_auth_challenge_tokenizer.h
  //net/http/http_auth_scheme.cc
  //net/http/http_auth_scheme.h
  //net/http/http_byte_range.cc
  //net/http/http_byte_range.h
  //net/http/http_log_util.cc
  //net/http/http_log_util.h
  //net/http/http_raw_request_headers.cc
  //net/http/http_raw_request_headers.h
  //net/http/http_request_headers.cc
  //net/http/http_request_headers.h
  //net/http/http_response_headers.cc
  //net/http/http_response_headers.h
  //net/http/http_response_info.cc
  //net/http/http_response_info.h
  //net/http/http_security_headers.cc
  //net/http/http_security_headers.h
  //net/http/http_status_code_list.h
  //net/http/http_util.cc
  //net/http/http_util.h
  //net/http/http_vary_data.cc
  //net/http/http_vary_data.h
  //net/http/structured_headers.cc
  //net/http/structured_headers.h
  //net/http/transport_security_state.h
  //net/http/transport_security_state_source.cc
  //net/http/transport_security_state_source.h
  //net/log/net_log.cc
  //net/log/net_log.h
  //net/log/net_log_capture_mode.cc
  //net/log/net_log_capture_mode.h
  //net/log/net_log_entry.cc
  //net/log/net_log_entry.h
  //net/log/net_log_event_type.h
  //net/log/net_log_event_type_list.h
  //net/log/net_log_source.cc
  //net/log/net_log_source.h
  //net/log/net_log_source_type.h
  //net/log/net_log_source_type_list.h
  //net/log/net_log_values.cc
  //net/log/net_log_values.h
  //net/log/net_log_with_source.cc
  //net/log/net_log_with_source.h
  //net/socket/client_socket_handle.cc
  //net/socket/client_socket_handle.h
  //net/socket/connect_job.cc
  //net/socket/connect_job.h
  //net/socket/connection_attempts.h
  //net/socket/next_proto.cc
  //net/socket/next_proto.h
  //net/socket/socket.cc
  //net/socket/socket.h
  //net/socket/socket_bio_adapter.cc
  //net/socket/socket_bio_adapter.h
  //net/socket/socket_performance_watcher.h
  //net/socket/socket_performance_watcher_factory.h
  //net/socket/ssl_client_socket.cc
  //net/socket/ssl_client_socket.h
  //net/socket/ssl_client_socket_impl.cc
  //net/socket/ssl_client_socket_impl.h
  //net/socket/ssl_socket.h
  //net/socket/stream_socket.cc
  //net/socket/stream_socket.h
  //net/ssl/client_cert_identity.cc
  //net/ssl/client_cert_identity.h
  //net/ssl/openssl_ssl_util.cc
  //net/ssl/openssl_ssl_util.h
  //net/ssl/ssl_cert_request_info.cc
  //net/ssl/ssl_cert_request_info.h
  //net/ssl/ssl_cipher_suite_names.cc
  //net/ssl/ssl_cipher_suite_names.h
  //net/ssl/ssl_client_auth_cache.cc
  //net/ssl/ssl_client_auth_cache.h
  //net/ssl/ssl_client_cert_type.h
  //net/ssl/ssl_client_session_cache.cc
  //net/ssl/ssl_client_session_cache.h
  //net/ssl/ssl_config.cc
  //net/ssl/ssl_config.h
  //net/ssl/ssl_config_service.cc
  //net/ssl/ssl_config_service.h
  //net/ssl/ssl_connection_status_flags.h
  //net/ssl/ssl_handshake_details.h
  //net/ssl/ssl_info.cc
  //net/ssl/ssl_info.h
  //net/ssl/ssl_key_logger.h
  //net/ssl/ssl_legacy_crypto_fallback.h
  //net/ssl/ssl_private_key.cc
  //net/ssl/ssl_private_key.h
  //net/ssl/ssl_server_config.cc
  //net/ssl/ssl_server_config.h
  //net/third_party/uri_template/uri_template.cc
  //net/third_party/uri_template/uri_template.h
  //net/base/net_errors_posix.cc
  //net/base/backoff_entry.cc
  //net/base/backoff_entry.h
  //net/base/backoff_entry_serializer.cc
  //net/base/backoff_entry_serializer.h
  //net/base/cache_metrics.cc
  //net/base/cache_metrics.h
  //net/base/cache_type.h
  //net/base/chunked_upload_data_stream.cc
  //net/base/chunked_upload_data_stream.h
  //net/base/data_url.cc
  //net/base/data_url.h
  //net/base/elements_upload_data_stream.cc
  //net/base/elements_upload_data_stream.h
  //net/base/expiring_cache.h
  //net/base/file_stream.cc
  //net/base/file_stream.h
  //net/base/file_stream_context.cc
  //net/base/file_stream_context.h
  //net/base/filename_util.cc
  //net/base/filename_util.h
  //net/base/filename_util_internal.cc
  //net/base/filename_util_internal.h
  //net/base/hex_utils.cc
  //net/base/hex_utils.h
  //net/base/host_mapping_rules.cc
  //net/base/host_mapping_rules.h
  //net/base/http_user_agent_settings.h
  //net/base/ip_pattern.cc
  //net/base/ip_pattern.h
  //net/base/load_flags.h
  //net/base/load_flags_list.h
  //net/base/load_states.h
  //net/base/load_states_list.h
  //net/base/logging_network_change_observer.cc
  //net/base/logging_network_change_observer.h
  //net/base/mime_sniffer.cc
  //net/base/mime_sniffer.h
  //net/base/mime_util.cc
  //net/base/mime_util.h
  //net/base/net_info_source_list.h
  //net/base/network_activity_monitor.cc
  //net/base/network_activity_monitor.h
  //net/base/network_change_notifier.cc
  //net/base/network_change_notifier.h
  //net/base/network_change_notifier_factory.h
  //net/base/network_delegate.cc
  //net/base/network_delegate.h
  //net/base/network_delegate_impl.cc
  //net/base/network_delegate_impl.h
  //net/base/network_isolation_key.cc
  //net/base/network_isolation_key.h
  //net/base/platform_mime_util.h
  //net/base/prioritized_dispatcher.cc
  //net/base/prioritized_dispatcher.h
  //net/base/prioritized_task_runner.cc
  //net/base/prioritized_task_runner.h
  //net/base/priority_queue.h
  //net/base/proxy_delegate.h
  //net/base/proxy_server.cc
  //net/base/proxy_server.h
  //net/base/request_priority.cc
  //net/base/request_priority.h
  //net/base/scheme_host_port_matcher.cc
  //net/base/scheme_host_port_matcher.h
  //net/base/scheme_host_port_matcher_result.h
  //net/base/scheme_host_port_matcher_rule.cc
  //net/base/scheme_host_port_matcher_rule.h
  //net/base/test_data_stream.cc
  //net/base/test_data_stream.h
  //net/base/upload_bytes_element_reader.cc
  //net/base/upload_bytes_element_reader.h
  //net/base/upload_data_stream.cc
  //net/base/upload_data_stream.h
  //net/base/upload_element_reader.cc
  //net/base/upload_element_reader.h
  //net/base/upload_file_element_reader.cc
  //net/base/upload_file_element_reader.h
  //net/base/upload_progress.h
  //net/cert/caching_cert_verifier.cc
  //net/cert/caching_cert_verifier.h
  //net/cert/cert_net_fetcher.h
  //net/cert/cert_verify_proc.cc
  //net/cert/cert_verify_proc.h
  //net/cert/cert_verify_proc_builtin.cc
  //net/cert/cert_verify_proc_builtin.h
  //net/cert/coalescing_cert_verifier.cc
  //net/cert/coalescing_cert_verifier.h
  //net/cert/ct_log_response_parser.cc
  //net/cert/ct_log_response_parser.h
  //net/cert/ct_log_verifier.cc
  //net/cert/ct_log_verifier.h
  //net/cert/ct_log_verifier_util.cc
  //net/cert/ct_log_verifier_util.h
  //net/cert/ct_objects_extractor.cc
  //net/cert/ct_objects_extractor.h
  //net/cert/ct_sct_to_string.cc
  //net/cert/ct_sct_to_string.h
  //net/cert/ct_serialization.cc
  //net/cert/ct_serialization.h
  //net/cert/ct_signed_certificate_timestamp_log_param.cc
  //net/cert/ct_signed_certificate_timestamp_log_param.h
  //net/cert/ev_root_ca_metadata.cc
  //net/cert/ev_root_ca_metadata.h
  //net/cert/internal/system_trust_store.cc
  //net/cert/internal/system_trust_store.h
  //net/cert/jwk_serializer.cc
  //net/cert/jwk_serializer.h
  //net/cert/known_roots.cc
  //net/cert/known_roots.h
  //net/cert/merkle_audit_proof.cc
  //net/cert/merkle_audit_proof.h
  //net/cert/merkle_consistency_proof.cc
  //net/cert/merkle_consistency_proof.h
  //net/cert/merkle_tree_leaf.cc
  //net/cert/merkle_tree_leaf.h
  //net/cert/multi_log_ct_verifier.cc
  //net/cert/multi_log_ct_verifier.h
  //net/cert/multi_threaded_cert_verifier.cc
  //net/cert/multi_threaded_cert_verifier.h
  //net/cert/root_cert_list_generated.h
  //net/cert/test_root_certs.cc
  //net/cert/test_root_certs.h
  //net/cert_net/cert_net_fetcher_url_request.cc
  //net/cert_net/cert_net_fetcher_url_request.h
  //net/cookies/canonical_cookie.cc
  //net/cookies/canonical_cookie.h
  //net/cookies/cookie_access_delegate.cc
  //net/cookies/cookie_access_delegate.h
  //net/cookies/cookie_change_dispatcher.cc
  //net/cookies/cookie_change_dispatcher.h
  //net/cookies/cookie_constants.cc
  //net/cookies/cookie_constants.h
  //net/cookies/cookie_deletion_info.cc
  //net/cookies/cookie_deletion_info.h
  //net/cookies/cookie_monster.cc
  //net/cookies/cookie_monster.h
  //net/cookies/cookie_monster_change_dispatcher.cc
  //net/cookies/cookie_monster_change_dispatcher.h
  //net/cookies/cookie_monster_netlog_params.cc
  //net/cookies/cookie_monster_netlog_params.h
  //net/cookies/cookie_options.cc
  //net/cookies/cookie_options.h
  //net/cookies/cookie_store.cc
  //net/cookies/cookie_store.h
  //net/cookies/cookie_util.cc
  //net/cookies/cookie_util.h
  //net/cookies/parsed_cookie.cc
  //net/cookies/parsed_cookie.h
  //net/cookies/site_for_cookies.cc
  //net/cookies/site_for_cookies.h
  //net/cookies/static_cookie_policy.cc
  //net/cookies/static_cookie_policy.h
  //net/disk_cache/backend_cleanup_tracker.cc
  //net/disk_cache/backend_cleanup_tracker.h
  //net/disk_cache/blockfile/addr.cc
  //net/disk_cache/blockfile/addr.h
  //net/disk_cache/blockfile/backend_impl.cc
  //net/disk_cache/blockfile/backend_impl.h
  //net/disk_cache/blockfile/bitmap.cc
  //net/disk_cache/blockfile/bitmap.h
  //net/disk_cache/blockfile/block_files.cc
  //net/disk_cache/blockfile/block_files.h
  //net/disk_cache/blockfile/disk_format.cc
  //net/disk_cache/blockfile/disk_format.h
  //net/disk_cache/blockfile/disk_format_base.h
  //net/disk_cache/blockfile/entry_impl.cc
  //net/disk_cache/blockfile/entry_impl.h
  //net/disk_cache/blockfile/errors.h
  //net/disk_cache/blockfile/eviction.cc
  //net/disk_cache/blockfile/eviction.h
  //net/disk_cache/blockfile/experiments.h
  //net/disk_cache/blockfile/file.cc
  //net/disk_cache/blockfile/file.h
  //net/disk_cache/blockfile/file_block.h
  //net/disk_cache/blockfile/file_lock.cc
  //net/disk_cache/blockfile/file_lock.h
  //net/disk_cache/blockfile/histogram_macros.h
  //net/disk_cache/blockfile/in_flight_backend_io.cc
  //net/disk_cache/blockfile/in_flight_backend_io.h
  //net/disk_cache/blockfile/in_flight_io.cc
  //net/disk_cache/blockfile/in_flight_io.h
  //net/disk_cache/blockfile/mapped_file.cc
  //net/disk_cache/blockfile/mapped_file.h
  //net/disk_cache/blockfile/rankings.cc
  //net/disk_cache/blockfile/rankings.h
  //net/disk_cache/blockfile/sparse_control.cc
  //net/disk_cache/blockfile/sparse_control.h
  //net/disk_cache/blockfile/stats.cc
  //net/disk_cache/blockfile/stats.h
  //net/disk_cache/blockfile/storage_block-inl.h
  //net/disk_cache/blockfile/storage_block.h
  //net/disk_cache/blockfile/stress_support.h
  //net/disk_cache/blockfile/trace.cc
  //net/disk_cache/blockfile/trace.h
  //net/disk_cache/cache_util.cc
  //net/disk_cache/cache_util.h
  //net/disk_cache/disk_cache.cc
  //net/disk_cache/disk_cache.h
  //net/disk_cache/memory/mem_backend_impl.cc
  //net/disk_cache/memory/mem_backend_impl.h
  //net/disk_cache/memory/mem_entry_impl.cc
  //net/disk_cache/memory/mem_entry_impl.h
  //net/disk_cache/net_log_parameters.cc
  //net/disk_cache/net_log_parameters.h
  //net/disk_cache/simple/post_doom_waiter.cc
  //net/disk_cache/simple/post_doom_waiter.h
  //net/disk_cache/simple/simple_backend_impl.cc
  //net/disk_cache/simple/simple_backend_impl.h
  //net/disk_cache/simple/simple_backend_version.h
  //net/disk_cache/simple/simple_entry_format.cc
  //net/disk_cache/simple/simple_entry_format.h
  //net/disk_cache/simple/simple_entry_format_history.h
  //net/disk_cache/simple/simple_entry_impl.cc
  //net/disk_cache/simple/simple_entry_impl.h
  //net/disk_cache/simple/simple_entry_operation.cc
  //net/disk_cache/simple/simple_entry_operation.h
  //net/disk_cache/simple/simple_file_tracker.cc
  //net/disk_cache/simple/simple_file_tracker.h
  //net/disk_cache/simple/simple_histogram_macros.h
  //net/disk_cache/simple/simple_index.cc
  //net/disk_cache/simple/simple_index.h
  //net/disk_cache/simple/simple_index_delegate.h
  //net/disk_cache/simple/simple_index_file.cc
  //net/disk_cache/simple/simple_index_file.h
  //net/disk_cache/simple/simple_net_log_parameters.cc
  //net/disk_cache/simple/simple_net_log_parameters.h
  //net/disk_cache/simple/simple_synchronous_entry.cc
  //net/disk_cache/simple/simple_synchronous_entry.h
  //net/disk_cache/simple/simple_util.cc
  //net/disk_cache/simple/simple_util.h
  //net/disk_cache/simple/simple_version_upgrade.cc
  //net/disk_cache/simple/simple_version_upgrade.h
  //net/filter/filter_source_stream.cc
  //net/filter/filter_source_stream.h
  //net/filter/gzip_header.cc
  //net/filter/gzip_header.h
  //net/filter/gzip_source_stream.cc
  //net/filter/gzip_source_stream.h
  //net/filter/source_stream.cc
  //net/filter/source_stream.h
  //net/filter/source_stream_type_list.h
  //net/http/alternative_service.cc
  //net/http/alternative_service.h
  //net/http/bidirectional_stream.cc
  //net/http/bidirectional_stream.h
  //net/http/bidirectional_stream_impl.cc
  //net/http/bidirectional_stream_impl.h
  //net/http/bidirectional_stream_request_info.cc
  //net/http/bidirectional_stream_request_info.h
  //net/http/broken_alternative_services.cc
  //net/http/broken_alternative_services.h
  //net/http/failing_http_transaction_factory.cc
  //net/http/failing_http_transaction_factory.h
  //net/http/http_auth.cc
  //net/http/http_auth.h
  //net/http/http_auth_cache.cc
  //net/http/http_auth_cache.h
  //net/http/http_auth_controller.cc
  //net/http/http_auth_controller.h
  //net/http/http_auth_filter.cc
  //net/http/http_auth_filter.h
  //net/http/http_auth_handler.cc
  //net/http/http_auth_handler.h
  //net/http/http_auth_handler_basic.cc
  //net/http/http_auth_handler_basic.h
  //net/http/http_auth_handler_digest.cc
  //net/http/http_auth_handler_digest.h
  //net/http/http_auth_handler_factory.cc
  //net/http/http_auth_handler_factory.h
  //net/http/http_auth_handler_ntlm.cc
  //net/http/http_auth_handler_ntlm.h
  //net/http/http_auth_mechanism.h
  //net/http/http_auth_multi_round_parse.cc
  //net/http/http_auth_multi_round_parse.h
  //net/http/http_auth_preferences.cc
  //net/http/http_auth_preferences.h
  //net/http/http_basic_state.cc
  //net/http/http_basic_state.h
  //net/http/http_basic_stream.cc
  //net/http/http_basic_stream.h
  //net/http/http_cache.cc
  //net/http/http_cache.h
  //net/http/http_cache_lookup_manager.cc
  //net/http/http_cache_lookup_manager.h
  //net/http/http_cache_transaction.cc
  //net/http/http_cache_transaction.h
  //net/http/http_cache_writers.cc
  //net/http/http_cache_writers.h
  //net/http/http_chunked_decoder.cc
  //net/http/http_chunked_decoder.h
  //net/http/http_content_disposition.cc
  //net/http/http_content_disposition.h
  //net/http/http_network_layer.cc
  //net/http/http_network_layer.h
  //net/http/http_network_session.cc
  //net/http/http_network_session.h
  //net/http/http_network_session_peer.cc
  //net/http/http_network_session_peer.h
  //net/http/http_network_transaction.cc
  //net/http/http_network_transaction.h
  //net/http/http_proxy_client_socket.cc
  //net/http/http_proxy_client_socket.h
  //net/http/http_proxy_connect_job.cc
  //net/http/http_proxy_connect_job.h
  //net/http/http_request_info.cc
  //net/http/http_request_info.h
  //net/http/http_response_body_drainer.cc
  //net/http/http_response_body_drainer.h
  //net/http/http_server_properties.cc
  //net/http/http_server_properties.h
  //net/http/http_server_properties_manager.cc
  //net/http/http_server_properties_manager.h
  //net/http/http_status_code.cc
  //net/http/http_status_code.h
  //net/http/http_stream.h
  //net/http/http_stream_factory.cc
  //net/http/http_stream_factory.h
  //net/http/http_stream_factory_job.cc
  //net/http/http_stream_factory_job.h
  //net/http/http_stream_factory_job_controller.cc
  //net/http/http_stream_factory_job_controller.h
  //net/http/http_stream_parser.cc
  //net/http/http_stream_parser.h
  //net/http/http_stream_request.cc
  //net/http/http_stream_request.h
  //net/http/http_transaction.h
  //net/http/http_transaction_factory.h
  //net/http/http_version.h
  //net/http/partial_data.cc
  //net/http/partial_data.h
  //net/http/proxy_client_socket.cc
  //net/http/proxy_client_socket.h
  //net/http/proxy_fallback.cc
  //net/http/proxy_fallback.h
  //net/http/transport_security_persister.cc
  //net/http/transport_security_persister.h
  //net/http/url_security_manager.cc
  //net/http/url_security_manager.h
  //net/http/webfonts_histogram.cc
  //net/http/webfonts_histogram.h
  //net/http2/platform/impl/http2_bug_tracker_impl.h
  //net/http2/platform/impl/http2_containers_impl.h
  //net/http2/platform/impl/http2_estimate_memory_usage_impl.h
  //net/http2/platform/impl/http2_flag_utils_impl.h
  //net/http2/platform/impl/http2_flags_impl.cc
  //net/http2/platform/impl/http2_flags_impl.h
  //net/http2/platform/impl/http2_logging_impl.h
  //net/http2/platform/impl/http2_macros_impl.h
  //net/http2/platform/impl/http2_string_utils_impl.h
  //net/log/file_net_log_observer.cc
  //net/log/file_net_log_observer.h
  //net/log/net_log_util.cc
  //net/log/net_log_util.h
  //net/log/trace_net_log_observer.cc
  //net/log/trace_net_log_observer.h
  //net/nqe/cached_network_quality.cc
  //net/nqe/cached_network_quality.h
  //net/nqe/effective_connection_type.cc
  //net/nqe/effective_connection_type.h
  //net/nqe/effective_connection_type_observer.h
  //net/nqe/event_creator.cc
  //net/nqe/event_creator.h
  //net/nqe/network_congestion_analyzer.cc
  //net/nqe/network_congestion_analyzer.h
  //net/nqe/network_id.cc
  //net/nqe/network_id.h
  //net/nqe/network_qualities_prefs_manager.cc
  //net/nqe/network_qualities_prefs_manager.h
  //net/nqe/network_quality.cc
  //net/nqe/network_quality.h
 ``` 
 
 ####  Chromium module -- net relation tree
 
  ```shell
root@007407603c9f:/androidchrome/chromium/src# gn desc out/Default/ net deps --tree  >> /androidchrome/chromium_net_module_relation_tree.txt 
WARNING at build arg file (use "gn args <out_dir>" to edit):4:28: Build argument has no effect.
enable_incremental_javac = false
                           ^----
The variable "enable_incremental_javac" was set as a build argument
but never appeared in a declare_args() block in any buildfile.

To view all possible args, run "gn args --list <out_dir>"

The build continued as if that argument was unspecified.

//build/config:shared_library_deps
  //build/config:common_deps
    //build/config/sanitizers:deps
      //build/config/sanitizers:options_sources
    //buildtools/third_party/libc++:libc++
      //buildtools/third_party/libc++abi:libc++abi
//net:net_deps
  //base:base
    //base:anchor_functions_buildflags
      //build:buildflag_header_h
    //base:base_jni_headers
      //base:android_runtime_jni_headers
        //base:debugging_buildflags
          //build:buildflag_header_h
        //base:logging_buildflags
          //build:buildflag_header_h
      //base:debugging_buildflags...
      //base:logging_buildflags...
    //base:base_static
    //base:build_date
    //base:cfi_buildflags
      //build:buildflag_header_h
    //base:clang_profiling_buildflags
      //build:buildflag_header_h
    //base:debugging_buildflags...
    //base:logging_buildflags...
    //base:orderfile_buildflags
      //build:buildflag_header_h
    //base:partition_alloc_buildflags
      //build:buildflag_header_h
    //base:sanitizer_buildflags
      //build:buildflag_header_h
    //base:synchronization_buildflags
      //build:buildflag_header_h
    //base/allocator:allocator
    //base/allocator:buildflags
      //build:buildflag_header_h
    //base/numerics:base_numerics
    //base/third_party/double_conversion:double_conversion
    //base/third_party/dynamic_annotations:dynamic_annotations
    //base/third_party/libevent:libevent
    //build:branding_buildflags
      //build:buildflag_header_h
    //build/config:shared_library_deps...
    //third_party/android_ndk:cpu_features
    //third_party/ashmem:ashmem
    //third_party/boringssl:boringssl
      //build/config:shared_library_deps...
      //third_party/boringssl:boringssl_asm
      //third_party/boringssl/src/third_party/fiat:fiat_license
    //third_party/modp_b64:modp_b64
  //base:i18n
    //base:base...
    //base/third_party/dynamic_annotations:dynamic_annotations
    //build:chromecast_buildflags
      //build:buildflag_header_h
    //build/config:shared_library_deps...
    //third_party/ced:ced
    //third_party/icu:icu
      //third_party/icu:icui18n
        //build/config:shared_library_deps...
        //third_party/icu:icuuc
          //build/config:shared_library_deps...
          //third_party/icu:icudata
            //third_party/icu:make_data_assembly
      //third_party/icu:icuuc...
  //base/third_party/dynamic_annotations:dynamic_annotations
  //base/util/type_safety:type_safety
  //net:constants
    //base:base...
  //net:net_export_header
  //net:net_jni_headers
    //base:debugging_buildflags...
    //base:logging_buildflags...
  //net:net_resources
    //base:base...
    //net:net_resources_grit
      //tools/grit:grit_sources
      //tools/gritsettings:default_resource_ids
        //tools/grit:grit_sources
  //net:preload_decoder
    //base:base...
  //net/base/registry_controlled_domains:registry_controlled_domains
  //third_party/brotli:dec
    //third_party/brotli:common
      //third_party/brotli:headers
    //third_party/brotli:headers
  //third_party/icu:icu...
  //third_party/protobuf:protobuf_lite
    //build/config:shared_library_deps...
  //third_party/zlib:zlib
    //build/config:shared_library_deps...
    //third_party/android_ndk:cpu_features
    //third_party/zlib:zlib_adler32_simd
    //third_party/zlib:zlib_arm_crc32
    //third_party/zlib:zlib_inflate_chunk_simd
    //third_party/zlib:zlib_x86_simd
  //url:buildflags
    //build:buildflag_header_h
//net:net_export_header
//net:net_public_deps
  //crypto:crypto
    //base:base...
    //base/third_party/dynamic_annotations:dynamic_annotations
    //build/config:shared_library_deps...
    //crypto:platform
      //third_party/boringssl:boringssl...
    //third_party/boringssl:boringssl...
  //crypto:platform...
  //net:buildflags
    //build:buildflag_header_h
  //net:net_nqe_proto
    //net:net_export_header
    //net:net_nqe_proto_gen
      //net:net_export_header
      //third_party/protobuf:protoc(//build/toolchain/linux:clang_x64)
        //build/config:executable_deps(//build/toolchain/linux:clang_x64)
          //build/config:common_deps(//build/toolchain/linux:clang_x64)
            //buildtools/third_party/libc++:libc++(//build/toolchain/linux:clang_x64)
              //buildtools/third_party/libc++abi:libc++abi(//build/toolchain/linux:clang_x64)
        //build/win:default_exe_manifest(//build/toolchain/linux:clang_x64)
        //third_party/protobuf:protoc_lib(//build/toolchain/linux:clang_x64)
          //third_party/protobuf:protobuf_full(//build/toolchain/linux:clang_x64)
            //third_party/zlib:zlib(//build/toolchain/linux:clang_x64)
              //build/config:shared_library_deps(//build/toolchain/linux:clang_x64)
                //build/config:common_deps(//build/toolchain/linux:clang_x64)...
              //third_party/zlib:zlib_adler32_simd(//build/toolchain/linux:clang_x64)
              //third_party/zlib:zlib_crc32_simd(//build/toolchain/linux:clang_x64)
              //third_party/zlib:zlib_inflate_chunk_simd(//build/toolchain/linux:clang_x64)
              //third_party/zlib:zlib_x86_simd(//build/toolchain/linux:clang_x64)
    //third_party/protobuf:protobuf_lite...
  //net/third_party/quiche:net_quic_proto
    //net:net_export_header
    //net/third_party/quiche:net_quic_proto_gen
      //net:net_export_header
      //third_party/protobuf:protoc(//build/toolchain/linux:clang_x64)...
    //third_party/protobuf:protobuf_lite...
  //net/third_party/quiche:net_quic_test_tools_proto
    //net:net_export_header
    //net/third_party/quiche:net_quic_test_tools_proto_gen
      //net:net_export_header
      //third_party/protobuf:protoc(//build/toolchain/linux:clang_x64)...
    //third_party/protobuf:protobuf_lite...
  //net/traffic_annotation:traffic_annotation
    //base:base...
  //third_party/boringssl:boringssl...
  //url:url
    //base:base...
    //base/third_party/dynamic_annotations:dynamic_annotations
    //build/config:shared_library_deps...
    //ipc:param_traits
    //third_party/icu:icu...
//net/dns:dns
  //net:net_deps...
  //net:net_public_deps...
  //net/dns:dns_client
    //net:net_deps...
    //net:net_public_deps...
    //net/dns:host_resolver
      //net:net_deps...
      //net:net_public_deps...
      //net/dns/public:public
        //net:net_deps...
        //net:net_public_deps...
    //net/dns/public:public...
  //net/dns:host_resolver...
  //net/dns:host_resolver_manager
    //net:net_deps...
    //net:net_public_deps...
    //net/dns:host_resolver...
    //net/dns/public:public...
  //net/dns:mdns_client
    //net:net_deps...
    //net:net_public_deps...
    //net/dns:dns_client...
    //net/dns:host_resolver...
//net/dns:dns_client...
//net/dns:host_resolver...
//net/dns:host_resolver_manager...
//net/dns:mdns_client...
//net/dns/public:public...
//net/http:transport_security_state_generated_files
  //build:branding_buildflags...
  //net:net_deps...
  //net:net_public_deps...
  //net/dns:dns...
  //net/http:generate_transport_security_state
    //net/tools/transport_security_state_generator:transport_security_state_generator(//build/toolchain/linux:clang_x64)
      //base:base(//build/toolchain/linux:clang_x64)
        //base:anchor_functions_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:base_static(//build/toolchain/linux:clang_x64)
        //base:build_date(//build/toolchain/linux:clang_x64)
        //base:cfi_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:clang_profiling_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:debugging_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:logging_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:orderfile_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:partition_alloc_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:sanitizer_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base:synchronization_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //base/allocator:allocator(//build/toolchain/linux:clang_x64)
          //base/allocator:tcmalloc(//build/toolchain/linux:clang_x64)
            //base/allocator:buildflags(//build/toolchain/linux:clang_x64)
              //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
            //base/third_party/dynamic_annotations:dynamic_annotations(//build/toolchain/linux:clang_x64)
        //base/allocator:buildflags(//build/toolchain/linux:clang_x64)...
        //base/allocator:tcmalloc(//build/toolchain/linux:clang_x64)...
        //base/numerics:base_numerics(//build/toolchain/linux:clang_x64)
        //base/third_party/double_conversion:double_conversion(//build/toolchain/linux:clang_x64)
        //base/third_party/dynamic_annotations:dynamic_annotations(//build/toolchain/linux:clang_x64)
        //base/third_party/libevent:libevent(//build/toolchain/linux:clang_x64)
        //base/third_party/symbolize:symbolize(//build/toolchain/linux:clang_x64)
        //base/third_party/xdg_mime:xdg_mime(//build/toolchain/linux:clang_x64)
        //base/third_party/xdg_user_dirs:xdg_user_dirs(//build/toolchain/linux:clang_x64)
        //build:branding_buildflags(//build/toolchain/linux:clang_x64)
          //build:buildflag_header_h(//build/toolchain/linux:clang_x64)
        //build/config:shared_library_deps(//build/toolchain/linux:clang_x64)...
        //third_party/boringssl:boringssl(//build/toolchain/linux:clang_x64)
          //build/config:shared_library_deps(//build/toolchain/linux:clang_x64)...
          //third_party/boringssl:boringssl_asm(//build/toolchain/linux:clang_x64)
          //third_party/boringssl/src/third_party/fiat:fiat_license(//build/toolchain/linux:clang_x64)
        //third_party/modp_b64:modp_b64(//build/toolchain/linux:clang_x64)
      //build/config:executable_deps(//build/toolchain/linux:clang_x64)...
      //crypto:crypto(//build/toolchain/linux:clang_x64)
        //base:base(//build/toolchain/linux:clang_x64)...
        //base/third_party/dynamic_annotations:dynamic_annotations(//build/toolchain/linux:clang_x64)
        //build/config:shared_library_deps(//build/toolchain/linux:clang_x64)...
        //crypto:platform(//build/toolchain/linux:clang_x64)
          //third_party/boringssl:boringssl(//build/toolchain/linux:clang_x64)...
        //third_party/boringssl:boringssl(//build/toolchain/linux:clang_x64)...
      //net/tools/transport_security_state_generator:transport_security_state_generator_sources(//build/toolchain/linux:clang_x64)
        //base:base(//build/toolchain/linux:clang_x64)...
        //net/tools/huffman_trie:huffman_trie_generator_sources(//build/toolchain/linux:clang_x64)
          //base:base(//build/toolchain/linux:clang_x64)...
        //third_party/boringssl:boringssl(//build/toolchain/linux:clang_x64)...
      //third_party/boringssl:boringssl(//build/toolchain/linux:clang_x64)...
//net/third_party/quiche:quiche
  //net:net_deps...
  //net:net_public_deps...

 ```
 
   [懒人chromium net android移植指南]: <https://blog.csdn.net/tq08g2z/article/details/77311318>
   [Building Chromium for Android]: <https://www.andreasch.com/2019/01/29/build-chromium-android/>
   [Docker Desktop for Mac]: <https://gitlab.silkrode.com.tw/team_mobile/termux-packages/blob/dev_neo/Docker.dmg>

[Docker Desktop for Windows]: <https://gitlab.silkrode.com.tw/team_mobile/termux-packages/blob/dev_neo/Docker Desktop Installer.exe>

[WSL (Windows Subsystem for Linux)  ]: <http://tinyurl.com/y674v74a>

[WSL (Windows Subsystem for Linux)  ]: <http://tinyurl.com/y674v74a>

[GN的使用 - GN工具  ]: <https://www.twblogs.net/a/5c03dbd0bd9eee728c16ab43>

