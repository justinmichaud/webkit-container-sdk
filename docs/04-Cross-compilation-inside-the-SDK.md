## Cross-compiling inside the SDK

There are two ways to cross-compile WebKit, for two different kinds of target.

* For an **Ubuntu device** — armv7, RISC-V, arm64, ppc64el — use the sysroot flow
  below. It compiles at full native speed and needs no emulation to build.
* For an **embedded image** you flash onto a board, use WebKit's Yocto flow,
  documented at the end of this page.

### Cross-compiling for an Ubuntu device

The sysroot is a container image, `wkdev-sysroot`, built from
`images/wkdev_sysroot/Containerfile`. Its contents are whatever WebKit's own
`Tools/{wpe,gtk}/install-dependencies` ask for, installed natively for the target
architecture, so there is no package list in this repository to keep in sync with
WebKit. The target architecture is `--arch`, the target Ubuntu release is the
`FROM` tag.

1. Build the sysroot image, on the host:

```sh
wkdev-sdk-bakery --mode=build --name=wkdev-sysroot --arch=riscv64
```

Use `--arch=arm` for armv7 (armhf), `--arch=arm64`, or `--arch=ppc64el`. Building
an image for an architecture your machine cannot execute needs a binfmt handler
on the host; it is only installing packages, so it is slow but not painfully so.

2. Unpack it into a sysroot and generate a CMake toolchain file, on the host:

```sh
wkdev-cross-sysroot --arch=riscv64 --release=noble
```

Pick `--release` to match the **oldest** device you need to run on: glibc is
backwards but not forwards compatible, so binaries built against 26.04 will not
load on a 24.04 device. The default sysroot location is
`${HOME}/.cache/wkdev-cross/ubuntu-<release>-<arch>`.

3. Build, inside the container. `wkdev-create` bind-mounts your host home and
   exports `${HOST_HOME}`, so the sysroot is reachable from in here — which is why
   the generated toolchain file takes its path from the environment rather than
   baking one in:

```sh
export WKDEV_CROSS_SYSROOT="${HOST_HOME}/.cache/wkdev-cross/ubuntu-noble-riscv64"

cmake -S . -B WebKitBuild-riscv64/JSCOnly/Release -G Ninja \
    -DPORT=JSCOnly -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE="${WKDEV_CROSS_SYSROOT}/toolchain.cmake"
ninja -C WebKitBuild-riscv64/JSCOnly/Release jsc
```

Keep cross builds in their own build directory so they do not invalidate your
native one. `ccache` and `sccache` work unchanged.

For a full port, a few options cannot be auto-detected when cross-compiling:

```sh
cmake -S . -B WebKitBuild-riscv64/WPE/Release -G Ninja \
    -DPORT=WPE -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE="${WKDEV_CROSS_SYSROOT}/toolchain.cmake" \
    -DENABLE_WPE_PLATFORM=ON -DENABLE_WPE_LEGACY_API=OFF -DENABLE_WPE_QT_API=OFF \
    -DENABLE_INTROSPECTION=OFF -DENABLE_DOCUMENTATION=OFF -DUSE_LIBBACKTRACE=OFF \
    -DBWRAP_EXECUTABLE=/usr/bin/bwrap -DDBUS_PROXY_EXECUTABLE=/usr/bin/xdg-dbus-proxy
```

`libwpe` and `wpebackend-fdo` are not packaged by Ubuntu, hence WPEPlatform;
introspection and documentation run target binaries; libbacktrace is not packaged
for any architecture; and the bubblewrap paths are baked into the binary, so they
are *device* paths.

### Running and testing what you built

`wkdev-cross-emulate` runs cross-built binaries, using emulation only where it is
actually needed. It never writes the host's `binfmt_misc` registry.

```sh
wkdev-cross-emulate --mode=status --arch=riscv64
```

| What you have | Mode |
|---|---|
| One binary | `--mode=run -- <binary> [args]` |
| A test runner that takes a binary path | `--mode=wrapper`, then `--artifact-exec-wrapper` |
| Something that execs target binaries itself, such as WebKitTestRunner | `--mode=exec -- <command>` |

```sh
wkdev-cross-emulate --mode=run --build-directory=WebKitBuild-riscv64/JSCOnly/Release -- \
    WebKitBuild-riscv64/JSCOnly/Release/bin/jsc -e 'print(1 + 1)'
```

For the JSC test suite, generate a wrapper and hand it to `run-jsc-stress-tests`,
which has a hook for exactly this. Pass the **real** binary so the runner can read
its ELF header, and use `--arch` rather than `--force-architecture`, which makes
the runner prepend the macOS-only `/usr/bin/arch`:

```sh
wkdev-cross-emulate --mode=wrapper --arch=riscv64 --output=/tmp/qemu-wrap
Tools/Scripts/run-jsc-stress-tests --jsc WebKitBuild-riscv64/JSCOnly/Release/bin/jsc \
    --arch riscv64 --artifact-exec-wrapper /tmp/qemu-wrap JSTests/stress
```

`--mode=exec` registers a qemu handler in a **private** `binfmt_misc` instance
inside a throwaway user and mount namespace, so the registration disappears when
the command exits and no other process on the machine is affected. It needs a
kernel with namespaced `binfmt_misc`, Linux 6.7 or newer.

Note that emulation only models the target ISA. It does not faithfully model weak
memory ordering, so passing tests under emulation is not evidence that concurrent
code is correct on real hardware.

If the host can execute the target architecture natively — for instance an arm64
machine that implements AArch32, running armv7 binaries — `--mode=run` skips the
emulator, and `--mode=exec` says so and gets out of the way, because `binfmt_misc`
cannot usefully intercept an architecture the kernel already handles. To run a
whole multi-process build in that situation, bake the sysroot's loader in at link
time instead:

```sh
-DCMAKE_EXE_LINKER_FLAGS="-Wl,--dynamic-linker=${WKDEV_CROSS_SYSROOT}/lib/arm-linux-gnueabihf/ld-linux-armhf.so.3 -Wl,--disable-new-dtags -Wl,-rpath,${WKDEV_CROSS_SYSROOT}/usr/lib/arm-linux-gnueabihf"
```

`--disable-new-dtags` is required: the default `DT_RUNPATH` is not inherited by
dependencies, so transitive libraries are not found. Such a build runs here but is
not deployable, so keep it in its own build directory.

### Deploying to the device

The device needs the runtime counterparts of the sysroot's `-dev` packages. If it
runs the same Ubuntu release the sysroot was built from, installing the port's
runtime dependencies with `apt` covers it.

```sh
Tools/CISupport/built-product-archive --platform=wpe --release archive
```

### Cross-compiling for an embedded image, with Yocto

WebKit also provides a Yocto based environment, which builds both a toolchain and
a flashable image. See `/path/to/WebKit/Tools/yocto/README.md`. You do not need to
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
