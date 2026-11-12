# Building the static JavaFX SDK

This document describes how to build the static JavaFX SDK (the one used by
[Gluon Substrate](https://github.com/gluonhq/substrate) to create native images) from the command line.

## Supported targets

| Target            | Build host           | Flavours         |
|-------------------|----------------------|------------------|
| `linux-x86_64`    | Linux x86_64         | regular, monocle | 
| `linux-aarch64`   | Linux aarch64        | regular, monocle | 
| `darwin-x86_64`   | macOS x86_64         | regular, monocle | 
| `darwin-aarch64`  | macOS aarch64        | regular, monocle | 
| `windows-x86_64`  | Windows x86_64       | regular, monocle | 
| `ios-arm64`       | macOS                | regular          | 
| `android-aarch64` | Linux x86_64         | regular          | 

There is no cross-compilation for desktop targets: each one is built on a
machine of the same OS and architecture.

## Requirements

### All targets

* `git`, `zip`, and `wget` or `curl`.
* The platform prerequisites for a regular OpenJFX build (compilers, system
  libraries, etc.), as listed in the
  [Building OpenJFX](https://wiki.openjdk.org/display/OpenJFX/Building+OpenJFX) wiki.
* The JDK version set in `jfx.build.jdk.version` in
  [`build.properties`](build.properties) (currently **19.0.2**), with
  `JAVA_HOME` pointing to it. Gradle is provided by the `gradlew` wrapper.

### Per target

| Target            | Additional requirements |
|-------------------|-------------------------|
| Linux             | `objcopy` (binutils). Ubuntu 22. |
| macOS             | Xcode with the command line tools. |
| Windows           | Visual Studio 2022 Build Tools 17.1 (MSVC 14.31.31103) and a POSIX shell (Cygwin) to run the commands below. |
| iOS               | Xcode with the iPhoneOS SDK matching `IOS_VERSION` (14.4). |
| Android           | Android NDK r20b, and FreeType built for `arm64-v8a` (the install dir must contain `lib/libfreetype.so`). |

On Windows, CI sets these environment variables before building (adjust the
path to your Visual Studio installation):

```
VS150COMNTOOLS=C:\progra~2\micros~1\2022\17.1.0\BuildTools\VC\Auxiliary\Build
MSVC_VER=14.31.31103
MSVC_REDIST_VER=14.31.31103
WINDOWS_CRT_VER=143
```

## Build steps

All commands are run from the root of the repository.

### 1. Get the sources

```
git clone https://github.com/gluonhq/jfx-static.git
cd jfx-static
git checkout <tag>        # optional: the tag you want to build
```

### 2. Set up the JDK

Download the JDK for your build host and unpack it. For example on Linux x86_64:

```
JDK_VERSION=$(grep "^jfx.build.jdk.version=" build.properties | cut -d= -f2)
JDK_MAJOR=$(echo $JDK_VERSION | cut -d. -f1)
wget -O jdk.tar.gz https://download.oracle.com/java/$JDK_MAJOR/archive/jdk-${JDK_VERSION}_linux-x64_bin.tar.gz
tar zxf jdk.tar.gz --directory tools
export JAVA_HOME=$(pwd)/$(ls -d tools/jdk-* | head -n1)
```

The archive suffix for the other hosts is `linux-aarch64_bin.tar.gz`,
`macos-x64_bin.tar.gz`, `macos-aarch64_bin.tar.gz` or `windows-x64_bin.zip`.
On macOS, `JAVA_HOME` must end in `/Contents/Home`.

Any other JDK of the same version works as well.

### 3. Run the Gradle build

Pick the command for your target. `BUILD_NUMBER` is the promoted build number
that ends up in the JavaFX version string; use `0` if you don't care.

```
BUILD_NUMBER=0
```

**Linux x86_64**

```
sh gradlew --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false clean sdk
objcopy --localize-hidden build/sdk/lib/libjavafx_iio.a
```

**Linux aarch64**

```
sh gradlew --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false clean all -x apps
objcopy --localize-hidden build/sdk/lib/libjavafx_iio.a
```

**macOS x86_64**

```
sh ./gradlew --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false clean sdk
```

**macOS aarch64**

```
sh ./gradlew --no-daemon --info -PSTATIC_BUILD=true -PTARGET_ARCH=arm64 -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false clean sdk
```

**Windows x86_64**

```
./gradlew.bat --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false clean sdk
```

**iOS arm64**

```
sh gradlew --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false -PCONF=DebugNative -PCOMPILE_TARGETS=ios -PIOS=true -PIOS_VERSION=14.4 -PUSE_LIPO=true clean sdk
mv build/ios-sdk build/sdk
```

**Android aarch64**

```
ANDROID_NDK=/opt/android-ndk-r20b
FREETYPE_DIR=/opt/freetype/install/arm64-v8a
sh gradlew --no-daemon --info -PSTATIC_BUILD=true -PPROMOTED_BUILD_NUMBER=$BUILD_NUMBER -PCOMPILE_WEBKIT=false -PCOMPILE_MEDIA=false -PCOMPILE_TARGETS=android -PANDROID_NDK=$ANDROID_NDK -PFREETYPE_DIR=$FREETYPE_DIR clean sdk
mv build/android-sdk build/sdk
cp $FREETYPE_DIR/lib/libfreetype.so build/sdk/lib
```

#### Optional: WebKit and Media

* **WebKit** (Linux and macOS only): pass `-PCOMPILE_WEBKIT=true` and, after
  the build, copy the static WebKit libraries into the SDK:

  ```
  cp -r modules/javafx.web/build/linux/Release/lib/* build/sdk/lib   # Linux
  cp -r modules/javafx.web/build/mac/Release/lib/* build/sdk/lib     # macOS
  ```

* **Media** (Linux only): pass `-PCOMPILE_MEDIA=true`.

Both are always disabled for Windows, iOS and Android.

#### Optional: Monocle flavour

For the desktop targets, a second SDK that includes Monocle can be built by
adding `-PINCLUDE_MONOCLE=true` to the same Gradle command. Remove the `build`
directory first if you already built the regular flavour, and name the zip in
step 4 `javafx-static-sdk-monocle-$JFX_VERSION.zip`.

### 4. Package the SDK

The jars are duplicated with a dash instead of a dot in their name
(`javafx-base.jar` next to `javafx.base.jar`), and the `sdk` folder is zipped:

```
for m in base controls fxml graphics media web; do
    if [ -f build/sdk/lib/javafx.$m.jar ]; then
        cp build/sdk/lib/javafx.$m.jar build/sdk/lib/javafx-$m.jar
    fi
done

JFX_VERSION=$(grep "^jfx.release.major.version=" build.properties | cut -d= -f2)
cd build && zip -r javafx-static-sdk-regular-$JFX_VERSION.zip sdk
```

For non-major releases (e.g. `21.0.1`), `JFX_VERSION` is derived from the
`jfx.release.{major,minor,security,patch}.version` properties.

## Result

The static SDK is available in `build/sdk`, with the jars and the static
native libraries in `build/sdk/lib`, and zipped in
`build/javafx-static-sdk-regular-<version>.zip`.

Finally, the zip is named as `openjfx-<version>-<target>-static.zip` (and
`openjfx-<version>-<target>-static-monocle.zip` for the Monocle flavour),
where `<target>` is one of the names in the table above.
