# Open64 for Iridium's EDG-to-WHIRL front end

This branch, `iridium-edg-whirl`, is Open64 `d69db73` with the fixes
that let its MIPS-target back end (`be`) compile the n32 WHIRL that
Iridium's EDG-to-WHIRL front end writes, for IRIX 6.5 (step 7 of that
front end: `docs/edg-whirl.md` in https://github.com/acerjanic/Iridium).

The front end writes WHIRL in MIPSpro 7.4's layout. Kintsugi's
`whirlconv to-open64` (https://github.com/acerjanic/kintsugi,
`toolchain/whirl`) converts it to Open64's, this back end compiles it
to assembly, Iridium's `fix-asm` rewrites that for IRIX's assembler, and
MIPSpro's `asm` and `ld32` make the program on IRIX. Iridium's
`eccw-irix -backend=open64` runs those steps through its relay
(`tools/edg-whirl/open64/be-open64`, `serve.sh --open64 DIR`).

Each commit on top of `d69db73` is one fix: first the 13 from the
Kintsugi workstream 5 handoff (building the MIPS back end on a modern
x86-64 Linux host), then Iridium's (what our WHIRL holds that Open64's
own front ends never write, the n32 register width, long double as
SGI's double-double on a little-endian host, and others). Each message
says what and why. GPL, as Open64.

## Building (podman, x86-64 Linux host)

The image (Ubuntu 20.04 and its compilers):

    FROM docker.io/library/ubuntu:20.04
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update -qq && apt-get install -y -qq build-essential gcc g++ \
        make cmake flex bison perl gawk binutils libc6-dev zlib1g-dev \
        libncurses-dev libtinfo-dev python3 python-is-python3 texinfo \
        clang-11 llvm-11-dev libclang-11-dev

Run with this tree at `/open64` (writable: the libjson build writes into
it) and an empty build directory at `/build`, at most 4 jobs:

    podman run --rm --security-opt label=disable --cpus 4 \
        -v $PWD:/open64 -v $BUILD:/build open64-build bash /build/build.sh

First the x86-64 tools (`ir_b2a` and the rest), which the MIPS build's
makefiles start from:

    export CLANG_HOME=/usr/lib/llvm-11
    mkdir -p /build/tree && cd /build/tree
    /open64/configure --prefix=/opt/open64 --disable-jfe --disable-multilib \
        --disable-fortran --with-build-optimize=DEBUG
    make build -j4

Then the MIPS-target back end, in its own build directory
(`osprey/targdir_mips`) with a copy of `Makefile.gsetup` that allows an
x86-64 host with a MIPS target:

    export PATH=$CLANG_HOME/bin:$PATH
    cd /build/tree/osprey
    (cd targdir; find . -maxdepth 2 -name Makefile) | while read f; do
        mkdir -p targdir_mips/$(dirname $f); cp targdir/$f targdir_mips/$f; done
    python3 - <<'PY'
    import re
    s = open('Makefile.gsetup').read()
    blk = ''' ifeq ($(BUILD_ABI), I64BIT)
     ifneq ($(BUILD_ARCH), IA64)
      ABORT_BUILD = BUILD_ABI=I64BIT requires BUILD_ARCH=IA64
     endif
     endif'''
    s = s.replace(blk, ' # x86-64 host, MIPS target')
    s = re.sub(r'(BUILD_AREA\s*=\s*\S*/targdir)\n', r'\1_mips\n', s, count=1)
    open('Makefile.gsetup.mips', 'w').write(s)
    PY
    find targdir_mips -name Makefile |
        xargs sed -i 's|osprey/Makefile.gsetup$|osprey/Makefile.gsetup.mips|'
    cd /build/tree
    make -k NATIVE_BUILD_DIR=osprey/targdir_mips BUILD_TARGET=MIPS -j4 \
        be.so cg.so wopt.so ipl.so lno.so be

The result is `osprey/targdir_mips/be/be` with `be.so`, `cg.so`,
`wopt.so`, `lno.so` and `ipl.so` beside it in their directories. It runs
on the host directly (the build directory is what `serve.sh --open64`
takes). After a source change, the last `make` line alone rebuilds it.

## Running the back end

    LD_LIBRARY_PATH=$T/be:$T/cg:$T/wopt:$T/lno:$T/ipl \
    $T/be/be -PHASE:w:c -TARG:abi=n32 -TARG:isa=mips4 -TARG:processor=r10000 \
        -TARG:endian=big -TENV:PIC -G8 -O2 -LANG:=cplus -s -fs,out.s in.B

with `T=osprey/targdir_mips` under the build directory and `in.B` from
`whirlconv --no-sidecar to-open64`. The `-TARG:` options select n32, the
MIPS IV ISA and the R10000, big-endian; `-TENV:PIC -G8` give the
position-independent code IRIX's shared libraries need, as MIPSpro's
driver passes them. The phases follow MIPSpro's driver: `-PHASE:c` at
`-O0` and `-O1` (and with `-g`), `-PHASE:w:c` at `-O2`, `-PHASE:l:w:c`
at `-O3`. `-LANG:=ansi_c` for C, `-LANG:=cplus` for C++ (built with
exceptions off: this back end writes no exception tables MIPSpro's
runtime reads). The assembly is GNU syntax; Iridium's
`tools/edg-whirl/open64/fix-asm` turns it into IRIX `as` input.
