---
layout: post
title: Linking with the Rust Cross-Compiler
category: rust
---

I'd like to start this blog by saying that it's really easy to build a
Rust cross-compiler for Windows and Android on Linux. It's just:

{% highlight bash %}
cd /tmp/
$ANDROID_NDK/build/tools/make-standalone-toolchain.sh \
    --platform=android-14 \
    --arch=arm \
    --install-dir=$NDK_TOOLCHAIN
git clone https://github.com/rust-lang/rust
cd rust
mkdir build
cd build
../configure --target=arm-linux-androideabi,x86_64-unknown-linux-gnu,x86_64-pc-windows-gnu \
    --android-cross-path=$NDK_TOOLCHAIN
make
sudo make install
wget https://static.rust-lang.org/cargo-dist/cargo-nightly-x86_64-unknown-linux-gnu.tar.gz
tar xf cargo-nightly-x86_64-unknown-linux-gnu.tar.gz
./cargo-nightly-x86_64-unknown-linux-gnu/install.sh
{% endhighlight %}

where $ANDROID_NDK is where you put the [Android NDK](https://developer.android.com/tools/sdk/ndk/index.html)
and $NDK_TOOLCHAIN is a place you can keep a ~300MB compiler toolchain.

However, there is a slight trick to getting Cargo to link code. If you get:

```
/usr/bin/ld: /home/tommy/workspace/xlang-test/hello_world/target/arm-linux-androideabi/debug/hello_world.o: Relocations in generic ELF (EM: 40)
/home/tommy/workspace/xlang-test/hello_world/target/arm-linux-androideabi/debug/hello_world.o: error adding symbols: File in wrong format
collect2: error: ld returned 1 exit status
```

when you compile for Android or

```
note: /usr/bin/ld: unrecognized option '--enable-long-section-names'
/usr/bin/ld: use the --help option for usage information
collect2: error: ld returned 1 exit status
```

then you need to set your linkers. The first step is wrapping the Android linker, as all 
Android executables need to be compiled as position independent executables and
there's no way to add arguments to linkers in the config. I'd recommend putting the
wrapper in the `~/.cargo/` directory, in `android-linker`.

{% highlight bash %}
#!/bin/bash
set -e
$NDK_TOOLCHAIN/arm-linux-androideabi/bin/gcc -pie "$@"
{% endhighlight %}

except with `$NDK_TOOLCHAIN` replaced with the absolute path to the toolchain, unless
you also decide to set that in your shell profile. Then, you can setup your Cargo configuration.

{% highlight toml %}
[target.arm-linux-androideabi]
linker = "$HOME/.cargo/android-linker"
[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"
{% endhighlight %}

`$HOME` needs to be replaced with the actual absolute path of your home directory. Alternately,
you could put `android-linker` in your path, but this avoids polluting your path with something
that will probably be rendered obsolete in the next few months.

If you actually want to run your application in the context of Android, there are two ways to
do it. If you want to run a NativeActivity, you apparently need to extern your main function
and add support for an Android app struct. [android-rs-glue](https://github.com/tomaka/android-rs-glue)
seems to have this down, but it wasn't working with my current nightlies and I wasn't interested
in that use case. I wanted to run the Rust program as an executable from an app. So, I put the
Rust program in the `jniLibs` directory in the app directory and renamed it so that it started
with `lib` and ended with `.so`. If you called it `libprogram.so`, then it can be started like
this:

{% highlight java %}
String fileName = getApplicationContext().getFilesDir().getParentFile().getPath() + "/lib/libprogram.so";
Process p = new ProcessBuilder(fileName)
    .redirectErrorStream(true)
    .start();
{% endhighlight %}
