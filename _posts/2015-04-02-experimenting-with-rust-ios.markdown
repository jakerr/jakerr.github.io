---
layout: post
title:  "Experimenting with Rust on iOS"
date:   2015-04-02 12:10:00
categories: rust ios
---

I'm doing some experimenting with using rust to build cross-platform graphical
apps.

The goal is to have a single codebase from which I can build iOS, Android, Linux, OS X and
Windows versions of the app. I'm thinking the easiest path to get
there is to lean on [SDL2](https://wiki.libsdl.org/FrontPage) and the
[Piston](http://www.piston.rs) game framework on top of that.

I guessed that iOS would be the more difficult of the two mobile platforms to
get working within the Rust ecosystem since it gets less attention (There's an
android build as part of the official
[CI](http://buildbot.rust-lang.org/waterfall) but no iOS one yet). That said,
iOS's build system is a lot more sane than Android's NDK so maybe it won't be
so bad. Anyhow, I decided to try to get going on iOS first and to my surprise,
I haven't run in to anything insurmountable just yet, after a day of
experimenting. I've gotten as far as building a Piston program for
`armv7-apple-ios` architecture including linking it against SDL2.framework
(built with [github/manifest/sdl-ios-framework](https://github.com/manifest/sdl-ios-framework)).
I still haven't tried running this on a device yet as I'm pretty sure I'll need
a host app that handles the UIApplicationMain invocation etc, I plan to
experiment with that next.

Walkthrough
-----------
What follows is a document of the steps it took me to successfully build (not
run) a Piston program with SDL2 for the `armv7-apple-ios` architecture.

See what breaks
===============

I started by just running `cargo build --target=armv7-apple-ios` to see what
breaks. The error I got was:

{% highlight bash %}
error: aborting due to previous error
/Users/jakerr/.multirust/toolchains/nightly/cargo/git/checkouts/read_color-5eab2e77aa8fb613/master/src/lib.rs:1:1: 1:1 error: can't find crate for `std`
/Users/jakerr/.multirust/toolchains/nightly/cargo/git/checkouts/read_color-5eab2e77aa8fb613/master/src/lib.rs:1 #![deny(missing_docs)]
                                                                                                                ^
error: aborting due to previous error
   Compiling bitflags v0.1.1
     Running `rustc /Users/jakerr/.multirust/toolchains/nightly/cargo/registry/src/github.com-1ecc6299db9ec823/bitflags-0.1.1/src/lib.rs --crate-name bitflags --crate-type lib -g -C metadata=518ea12e21428edd -C extra-filename=-518ea12e21428edd --out-dir /Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps --emit=dep-info,link --target armv7-apple-ios -L dependency=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps -L dependency=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps -Awarnings`
{% endhighlight %}

"Can't find crate for `std`" isn't the most obvious error but what it means is
the tool chain I had installed (nightly via multirust, in my case) wasn't built
with the target I requested. Ok so I have to build the rust compiler myself.

Building rustc for cross-compilation
====================================

It's as simple as passing `--target=...` to `./configure` when compiling rust.
I wanted my version to be pinned to the rustc version I have installed with
multirust so I wrote a script to update my arm version of the compiler.
It's a bit hacky and I'm not necessarily recommending you do it like this but
I'll share it anyhow.

The script assumes a directory structure like:

{% highlight bash %}
~/code/rustws/rust                  # The rust-lang/rust.git repo
~/code/rustws/rustc-build           # Where my build script will live
~/code/rustws/rustc-build/build/arm # The output directory for the toolchain
{% endhighlight %}

`cat ~/code/rustws/rust-build/arm.sh`:

{% highlight bash %}
#!/bin/sh
set -eu

vers=$(rustc --version |egrep -o "\(\w{9}" |egrep -o "\w+")

wd=$(pwd)
pushd ../rust
git checkout master
git pull
git co -B rustc-$vers $vers
git submodule update --init --recursive
./configure --target=armv7-apple-ios --prefix=$wd/build/arm
make
make install
{% endhighlight %}

Then I just symlink in a `cargo` binary from the system toolchain into the arm
chains bin/ since it doesn't seem to need to be custom. If you're using
multirust, don't symlink $(which cargo) because that's just a proxy script and it
will break if its location changes, you need to find the actual cargo bin and
symlink it in. For me this was:
{% highlight bash %}
ln -s /Users/jakerr/.multirust/toolchains/nightly/bin/cargo /Users/jakerr/code/rustws/rustc-build/build/arm/bin/cargo
{% endhighlight %}

Then make the toolchain available to multirust for overriding:
{% highlight bash %}
multirust update ios --link-local ~/code/rustws/rustc-build/build/arm/bin/cargo
{% endhighlight %}

Cargo build again
=================
First, we should override our toolchain to the freshly built one. If you followed the instructions above that
would be `multirust override ios`
Now that we have an ios rust toolchain we can give `cargo build
--target=armv7-apple-ios` another shot, this time we make it a bit further and
fail to link. Awesome it's exactly as I expected:
{% highlight bash %}
Undefined symbols for architecture armv7:
  "_SDL_Init", referenced from:
      sdl::init::hcad303edffe24aaaX8n in libsdl2-c03ee6bcc7e4fe4f.rlib(sdl2-c03ee6bcc7e4fe4f.o)
  "_SDL_GL_SetAttribute", referenced from:
      video::gl_set_attribute::ha3ecc9302869cd36E5l in libsdl2-c03ee6bcc7e4fe4f.rlib(sdl2-c03ee6bcc7e4fe4f.o)
      # ...
ld: symbol(s) not found for architecture armv7
clang: error: linker command failed with exit code 1 (use -v to see invocation)

error: aborting due to previous error
Could not compile `rustecspong`.

Caused by:
  Process didn't exit successfully: `rustc src/main.rs --crate-name rustecspong --crate-type bin -g --out-dir /Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug --emit=dep-info,link --target armv7-apple-ios -L dependency=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug -L dependency=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps --extern vecmath=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libvecmath-4b9e8c05387d4953.rlib --extern window=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libwindow-20dd55a2e8968db8.rlib --extern shader_version=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libshader_version-c0d3970179066072.rlib --extern input=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libinput-1490dbfb10945cbf.rlib --extern event=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libevent-8cd3eaa4361a82fa.rlib --extern graphics=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libgraphics-3ec65f4f2805722e.rlib --extern opengl_graphics=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libopengl_graphics-4b5470c415aad257.rlib --extern sdl2_window=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libsdl2_window-729940b7a5ce5bbd.rlib --extern quack=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libquack-940ac77f1ace5f08.rlib --extern ecs=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/libecs-35d6c01de75e62f4.rlib --extern rand=/Users/jakerr/code/rustws/rustecspong/target/armv7-apple-ios/debug/deps/librand-808067ff512dc02b.rlib` (exit code: 101)
{% endhighlight %}

I didn't expect it to magically know where SDL.framework is (As I mentioned
earlier I built it from scratch and just have it in my code folder somewhere)
but I have no idea how to get the `-L framework=...` flags that `rustc` needs
through `cargo`. I was hoping to find something in the [crates.io
documentation](http://doc.crates.io/guide.html) but didn't see it. Please do let me know if this is supported
and I'm just missing it. For now I can take the failed `rustc` command reported
from `cargo build --target=... -v` and append the framework flags required to
get everything to build and link:

{% highlight bash %}
rustc src/main.rs --crate-name # ... entire command coppied from cargo build --target=... -v \
-L framework=/Users/jakerr/code/rustws/sdl-ios-framework/build/sdl -l framework=SDL2 -l framework=UIKit -l framework=OpenGLES -l framework=CoreAudio -l framework=QuartzCore -l framework=CoreGraphics -l framework=AudioToolbox
{% endhighlight %}

All of those `-L framework=` and `-l framework=` flags are the ones I needed to
add to satisfy all of the `Undefined symbols for architecture armv7` errors
through trial and error and googling the symbols to determine which iOS
Frameworks each was referring to.

Conclusion
==========
And with that I've compiled a binary for `armv7-apple-ios`. Next stop, figuring
out how to make a host app bundle to stick it into and actually run it on
a device.

I'm really excited to see rust supporting so many architectures right out of the
gate. I'd love to see the cross-compilation toolchain ease-of-use improved.
Better diagnostics for missing toolchains would be great. Rather than just
complaining that the crate `std` can't be found a hint that the requested
toolchain is completely missing would be better. A way to specify the
frameworks to link in my `Cargo.toml` would also be really welcome (though
I have a feeling maybe I'm just missing something obvious here).

