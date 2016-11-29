---
layout: post
title: Compiling OpenSSL with Android NDK 13
---

Surprisingly painful experience. What I've learned is that there are practically 6 platforms (arm, x86 and mips + 64bit versions thereof), and
instead of having a simple enum that everybody respects the community decided to go wild on this one. So for architecture `arm`, we have a toolchain
that is called `arm` but openssl platform `android-armeabi`. For 64 bit ARM we have aliases `aarch64`, `arm64`, `android64-aarch64`. For `x86` we have `x86` 
and `i686`. For `x86_64` we've got that and `android64`. For `mips` it's sometimes `mips` and sometimes `mipsel`, but for `mips64` it's `mips64el`, because we can!

Another issue that might surprise a normal, same person is that `make-standalone-chain` will quitly ignore your requests if
some version of Android API is not supported by your target architecture. So do not try to create `arm64` (or is it `aarch64`) for `android-19`, won't work. 
Won't complaint either. 

Anyhow this beauty is a work of art. It will highly probably stop working after next minor release from Google, 
but it won't stop us, the community, from wasting time on this nonsense.

```bash
#!/usr/bin/env bash

# https://developer.android.com/ndk/guides/standalone_toolchain.html

export ARCH="x86"
export ANDROID_API="android-24"

case $ARCH in
  arm)
    export TOOLCHAIN="arm-linux-androideabi-4.9"
    export OPENSSL_CONFIG_OPTIONS="android-armeabi"
    export TOOL_PREFIX="arm-linux-androideabi-"
    export INSTALL_FOLDER="arch-arm"
    ;;
  aarch64)
    export TOOLCHAIN="aarch64-linux-android-4.9"
    export OPENSSL_CONFIG_OPTIONS="android64-aarch64"
    export TOOL_PREFIX="aarch64-linux-android-"
    export INSTALL_FOLDER="arch-arm64"
    ;;
  x86)
    export TOOLCHAIN="x86-4.9"
    export OPENSSL_CONFIG_OPTIONS="android-x86"
    export TOOL_PREFIX="i686-linux-android-"
    export INSTALL_FOLDER="arch-x86"
    ;;
  x86_64)
    export TOOLCHAIN="x86_64-4.9"
    export OPENSSL_CONFIG_OPTIONS="android64"
    export TOOL_PREFIX="x86_64-linux-android-"
    export INSTALL_FOLDER="arch-x86_64"
    ;;
  mips)
    export TOOLCHAIN="mipsel-linux-android-4.9"
    export OPENSSL_CONFIG_OPTIONS="android-mips"
    export TOOL_PREFIX="mipsel-linux-android-"
    export INSTALL_FOLDER="arch-mips"
    ;;
  mips64)
    export TOOLCHAIN="mips64el-linux-android-4.9"
    export OPENSSL_CONFIG_OPTIONS="android64-mips64"
    export TOOL_PREFIX="mips64el-linux-android-"
    export INSTALL_FOLDER="arch-mips64"
    ;;
  *)
    echo "ERROR: Unknown arch $ARCH"
    exit 1
    ;;
esac

export TOOLCHAIN_DIR="$( pwd )/toolchain-$TOOLCHAIN"
$NDK_ROOT/build/tools/make-standalone-toolchain.sh \
    --install-dir="$TOOLCHAIN_DIR" \
    --platform="$ANDROID_API" \
    --toolchain="$TOOLCHAIN" \
    --arch="$ARCH" \
    --force \
    --verbose

export SYSROOT="$TOOLCHAIN_DIR/sysroot"
export CROSS_SYSROOT="$SYSROOT"
export CROSS_COMPILE="$TOOLCHAIN_DIR/bin/$TOOL_PREFIX"
export ANDROID_DEV="$SYSROOT/usr"

cd openssl-1.1.0c
./Configure $OPENSSL_CONFIG_OPTIONS

make clean
make

cp lib{ssl,crypto}.{so,a,so.1.1} "$NDK_ROOT/platforms/$ANDROID_API/$INSTALL_FOLDER/usr/lib"
cp -R include/openssl "$NDK_ROOT/platforms/$ANDROID_API/$INSTALL_FOLDER/usr/include"

cd ..
```

Enjoy if you can.
