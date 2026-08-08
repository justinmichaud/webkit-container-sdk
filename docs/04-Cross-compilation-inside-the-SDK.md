## Cross-compiling inside the SDK

Two flows, for two kinds of target:

* An **Ubuntu device** -- armv7, arm64, riscv64, ppc64el: use the sysroot flow
  below. It builds natively, at full speed, with no emulation.
* An **embedded image** you flash onto a board: use WebKit's Yocto flow, at the
  end of this page.

### Cross-compiling for an Ubuntu device

1. Build the sysroot image, on the host. It holds whatever WebKit's own
`Tools/{wpe,gtk}/install-dependencies` install, for the target architecture:

```sh
wkdev-sdk-bakery --mode=build --name=wkdev-sysroot --arch=riscv64
```

`--arch` is podman's: `arm` (armv7), `arm64`, `riscv64` or `ppc64el`. This runs
the target's package manager, so an architecture your machine cannot execute needs
a binfmt handler on the host, such as `qemu-user-static` installs.

2. Unpack it and generate a CMake toolchain file, on the host:

```sh
wkdev-cross-sysroot --arch=riscv64 --release=noble
```

`--release` is the release the image was built from; target the **oldest** device
you need to run on, as glibc is backwards but not forwards compatible. The sysroot
lands in `${HOME}/.cache/wkdev-cross/ubuntu-<release>-<arch>`.

3. Build, inside the container, where your home is mounted at `${HOST_HOME}` --
which is why the toolchain file reads the sysroot path from the environment
instead of baking one in:

```sh
export WKDEV_CROSS_SYSROOT="${HOST_HOME}/.cache/wkdev-cross/ubuntu-noble-riscv64"

cmake -S . -B WebKitBuild-riscv64/JSCOnly/Release -G Ninja \
    -DPORT=JSCOnly -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE="${WKDEV_CROSS_SYSROOT}/toolchain.cmake"
ninja -C WebKitBuild-riscv64/JSCOnly/Release jsc
```

Keep cross builds in their own build directory, so they do not invalidate your
native one. `ccache` and `sccache` work unchanged.

A full port needs a few options that cannot be auto-detected when cross-compiling:

```sh
cmake -S . -B WebKitBuild-riscv64/WPE/Release -G Ninja \
    -DPORT=WPE -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE="${WKDEV_CROSS_SYSROOT}/toolchain.cmake" \
    -DENABLE_WPE_PLATFORM=ON -DENABLE_WPE_LEGACY_API=OFF -DENABLE_WPE_QT_API=OFF \
    -DENABLE_INTROSPECTION=OFF -DENABLE_DOCUMENTATION=OFF -DUSE_LIBBACKTRACE=OFF \
    -DBWRAP_EXECUTABLE=/usr/bin/bwrap -DDBUS_PROXY_EXECUTABLE=/usr/bin/xdg-dbus-proxy
```

Ubuntu does not package `libwpe` or `wpebackend-fdo`, hence WPEPlatform, nor
libbacktrace for any architecture; introspection and documentation run target
binaries; and the bubblewrap paths are baked into the binary, so they are *device*
paths.

### Running and testing what you built

`wkdev-cross-emulate` runs cross-built binaries, emulating only where needed. It
never writes the host's `binfmt_misc` registry.

| What you have | Mode |
|---|---|
| Just checking the setup | `--mode=status` |
| One binary | `--mode=run -- <binary> [args]` |
| A test runner that takes a binary path | `--mode=wrapper`, then `--artifact-exec-wrapper` |
| Something that execs target binaries itself, such as WebKitTestRunner | `--mode=exec -- <command>` |

```sh
wkdev-cross-emulate --mode=run --build-directory=WebKitBuild-riscv64/JSCOnly/Release -- \
    WebKitBuild-riscv64/JSCOnly/Release/bin/jsc -e 'print(1 + 1)'
```

For the JSC test suite, hand a wrapper to `run-jsc-stress-tests`. Pass the **real**
binary, so the runner can read its ELF header, and use `--arch` rather than
`--force-architecture`, which makes it prepend the macOS-only `/usr/bin/arch`:

```sh
wkdev-cross-emulate --mode=wrapper --arch=riscv64 --output=/tmp/qemu-wrap
Tools/Scripts/run-jsc-stress-tests --jsc WebKitBuild-riscv64/JSCOnly/Release/bin/jsc \
    --arch riscv64 --artifact-exec-wrapper /tmp/qemu-wrap JSTests/stress
```

`--mode=exec` registers a qemu handler in a **private** `binfmt_misc` instance,
inside a throwaway user and mount namespace, so it affects no other process and is
gone when the command exits. It needs namespaced `binfmt_misc`, Linux 6.7 or newer.

Emulation only models the target ISA. It does not faithfully model weak memory
ordering, so tests passing under emulation say nothing about whether concurrent
code is correct on real hardware.

If the host executes the target natively -- an arm64 machine implementing AArch32
runs armv7 binaries -- `--mode=run` skips the emulator, and `--mode=exec` gets out
of the way, `binfmt_misc` being unable to intercept what the kernel already
handles. To run a whole multi-process build there, bake the sysroot's loader in at
link time instead:

```sh
-DCMAKE_EXE_LINKER_FLAGS="-Wl,--dynamic-linker=${WKDEV_CROSS_SYSROOT}/lib/arm-linux-gnueabihf/ld-linux-armhf.so.3 -Wl,--disable-new-dtags -Wl,-rpath,${WKDEV_CROSS_SYSROOT}/usr/lib/arm-linux-gnueabihf"
```

`--disable-new-dtags` is required: the default `DT_RUNPATH` is not inherited by
dependencies, so transitive libraries are not found. Such a build runs here but is
not deployable, so keep it in its own build directory.

### Deploying to the device

The device needs the runtime counterparts of the sysroot's `-dev` packages. If it
runs the release the sysroot was built from, installing the port's runtime
dependencies with `apt` covers it. Then copy `bin/` and `lib/` over together, or,
for a build laid out where WebKit's scripts expect it:

```sh
Tools/CISupport/built-product-archive --platform=wpe --release archive
```

### Cross-compiling for an embedded image, with Yocto

WebKit provides a Yocto based environment, which builds both a toolchain and a
flashable image; see `/path/to/WebKit/Tools/yocto/README.md`. You do not need to
run it from within the SDK, but if you want to:

First set up the environment, so that Yocto caches downloads and sstate:

```sh
export DL_DIR="${HOME}/.cache/yocto/downloads"
export SSTATE_DIR="${HOME}/.cache/yocto/sstate"
export BB_ENV_PASSTHROUGH_ADDITIONS="${BB_ENV_PASSTHROUGH_ADDITIONS} DL_DIR SSTATE_DIR"
```

Then proceed with the compilation:

```sh
unset LD_LIBRARY_PATH
export NUMBER_OF_PROCESSORS=12 # Adapt to your machine.
export WEBKIT_USE_SCCACHE=0
Tools/Scripts/cross-toolchain-helper --cross-target rpi3-32bits-mesa --build-image
Tools/Scripts/cross-toolchain-helper --cross-target rpi3-32bits-mesa --build-toolchain
Tools/Scripts/cross-toolchain-helper --cross-target rpi3-32bits-mesa --cross-toolchain-run-cmd CFLAGS="" CPPFLAGS="" WebKitBuild/CrossToolChains/rpi3-32bits-mesa/build/toolchain/sysroots/x86_64-pokysdk-linux/post-relocate-setup.d/meson-setup.py
Tools/Scripts/cross-toolchain-helper --cross-target rpi3-32bits-mesa --cross-toolchain-run-cmd Tools/Scripts/build-webkit --wpe --release
```

It is important to unset `LD_LIBRARY_PATH`, otherwise `cross-toolchain-helper`
will partly fail while `build-webkit` continues, leading to an inconsistent build
environment that fails to produce binaries.

sccache is also not supported in that mode, and will interfere with Yocto --
disable it.

Follow the instructions in `Tools/yocto/README.md` to flash the image onto your
target machine.
