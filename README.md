# GCC cross compiler for Raspberry PI Zero W

The best tutorial for building the cross compiler for Raspberry PI Zero W I've found there https://solarianprogrammer.com/2018/05/06/building-gcc-cross-compiler-raspberry-pi/
However this tutorial needs some tweaking so that you can compile Java 17 and successfully run it on Raspberry PI Zero W. The linked tutorial has serveral external ressources and links, so I recommend to have a look at it.

Versions may change, for example to compile gcc 10.2 I needed gcc 8.5.0, trying with gcc 8.3.0 lead to an error. Lets begin with gcc 10.1 which lead to a successful build of OpenJDK 17 for Raspberry PI Zero W https://github.com/JsBergbau/OpenJDK-Raspberry-Pi-Zero-W-armv6

For this tutorial we will build the files in our homefolder. Debian 11 Bullseye 64 bit is used for build the the cross compiler. First gcc 8.3 is build and then with this gcc 10.1 is build, because otherwise there are errors in the build process.

## First step, compile gcc 8.3.0 cross compiler

```
cd ~
mkdir gcc_all && cd gcc_all
```

Lets begin with downloading all the required files:

```
wget https://ftpmirror.gnu.org/binutils/binutils-2.31.tar.bz2
wget https://ftpmirror.gnu.org/glibc/glibc-2.28.tar.bz2
wget https://ftpmirror.gnu.org/gcc/gcc-8.3.0/gcc-8.3.0.tar.gz
wget https://ftpmirror.gnu.org/gcc/gcc-10.1.0/gcc-10.1.0.tar.gz
git clone --depth=1 https://github.com/raspberrypi/linux
```


Extract them all:

```
tar xf binutils-2.31.tar.bz2
tar xf glibc-2.28.tar.bz2
tar xf gcc-8.3.0.tar.gz
tar xf gcc-10.1.0.tar.gz
```
and finally remove the archives via `rm *.tar.*`

Download some prequsites for gcc:

```
cd gcc-8.3.0
contrib/download_prerequisites
rm *.tar.*
cd ..
cd gcc-10.1.0
contrib/download_prerequisites
rm *.tar.*
```

Create the output folder:

```
cd ~/gcc_all
sudo mkdir -p /opt/cross-pi-gcc
sudo chown $USER /opt/cross-pi-gcc
export PATH=/opt/cross-pi-gcc/bin:$PATH
```

Copy the kernel headers to the cross compiler output folder:

```
cd ~/gcc_all
cd linux
export KERNEL=kernel
make ARCH=arm INSTALL_HDR_PATH=/opt/cross-pi-gcc/arm-linux-gnueabihf headers_install
```
Here is a significant change towards the original tutorial in line 3. We use `export` otherwise the environment variable can't be seen by make and we use `kernel` instead of kernel7, because kernel7 is for Rasbperry Pi 2 and Pi 3, see https://raspberrypi.stackexchange.com/a/104726/


Build the Binutils:

```
cd ~/gcc_all
mkdir build-binutils && cd build-binutils
../binutils-2.31/configure --prefix=/opt/cross-pi-gcc --target=arm-linux-gnueabihf --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib
make -j 12
make install
```

`-j` means the number of jobs in parallel. Choose according to your number of CPUs.


Do a partial compile of gcc 8.3, because gcc and glibc are interdependant, so there are several steps:

```
cd ~/gcc_all
mkdir build-gcc && cd build-gcc
../gcc-8.3.0/configure --prefix=/opt/cross-pi-gcc --target=arm-linux-gnueabihf --enable-languages=c,c++,fortran --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib
make -j 12 all-gcc
make install-gcc
```

Partial build of glibc:

```
cd ~/gcc_all
mkdir build-glibc && cd build-glibc
../glibc-2.28/configure --prefix=/opt/cross-pi-gcc/arm-linux-gnueabihf --build=$MACHTYPE --host=arm-linux-gnueabihf --target=arm-linux-gnueabihf --with-arch=armv6 --with-fpu=vfp --with-float=hard --with-headers=/opt/cross-pi-gcc/arm-linux-gnueabihf/include --disable-multilib libc_cv_forced_unwind=yes
make install-bootstrap-headers=yes install-headers
make -j 12 csu/subdir_lib
install csu/crt1.o csu/crti.o csu/crtn.o /opt/cross-pi-gcc/arm-linux-gnueabihf/lib
arm-linux-gnueabihf-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /opt/cross-pi-gcc/arm-linux-gnueabihf/lib/libc.so
touch /opt/cross-pi-gcc/arm-linux-gnueabihf/include/gnu/stubs.h
```

Next step of compiling gcc:

```
cd ..
cd build-gcc
make -j 12 all-target-libgcc
make install-target-libgcc
```

Now glibc is completely compiled:

```
cd ..
cd build-glibc
make -j 12
make install
```

gcc 8.3.0 compiling is now completed: 

```
cd ..
cd build-gcc
make -j 12
make install
cd ..
```

### Test gcc 8.3.0 cross compiler

You can now compile a hello world program, transfer it to Raspberry PI Zero W and execute

```hello.c
#include <stdio.h>
int main() {
        printf("hello\n");
        return 0;
}
```
Compile with `arm-linux-gnueabihf-g++ test.cpp -o hello`

## Second and last step: Commpile gcc 10.1

Edit `gcc-10.1.0/libsanitizer/asan/asan_linux.cpp` and add at about line 66 if there is no PATH_MAX

```
#ifndef PATH_MAX
#define PATH_MAX 4096
#endif
```

Get a sysroot, therefore install via `sudo apt install debootstrap`

```
sudo qemu-debootstrap \
  --arch=armhf \
  --verbose \
  --include=fakeroot,symlinks,build-essential,libx11-dev,libxext-dev,libxrender-dev,libxrandr-dev,libxtst-dev,libxt-dev,libcups2-dev,libfontconfig1-dev,libasound2-dev,libfreetype6-dev,libpng-dev,libffi-dev \
  --resolve-deps \
  buster \
  ~/sysroot-armhf \
  http://httpredir.debian.org/debian/
 ```

Ready to compile gcc 10.1
```
cd ~/gcc_all
mkdir build-gcc10 && cd build-gcc10
../gcc-10.1.0/configure --prefix=/opt/cross-pi-gcc --target=arm-linux-gnueabihf --enable-languages=c,c++,fortran --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib --enable-multiarch --with-sysroot=/home/pi/sysroot-armhf
make -j 12
make install
```
Note: Sysroot has to be specified as absolute path, without I got errors.

You can delete the gcc 8.3.0 folder in /opt/cross-pi-gcc/libexec/gcc/arm-linux-gnueabihf

Now you have in /opt/cross-pi-gcc the build environment you need, for example to compile OpenJDK 17 for Raspberry PI Zero W, see https://github.com/JsBergbau/OpenJDK-Raspberry-Pi-Zero-W-armv6


