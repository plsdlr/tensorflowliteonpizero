## **Tensorflow lite on Raspberry Pi Zero - a comprehensive guide**

This guide is written to help with crosscompilation and building Piwheels for tensorflow lite on Raspberry Pi Zero. The official tensorflow documentation seem to be out of date and also dosen't document how to build a  working crosscompilation toolchain. One warning: the whole process is quite time intensive. It was tested on **Ubuntu 18.04.5 LTS** and **Raspbian GNU/Linux 10 (buster)**. This repository also contains the precompiled *libtensorflow-lite.a* and *minimal* and the precompiled piwheel. If  you are lucky the piwheel might work out of the box. (jump to step 4)

Therefore this guide is seperated in four parts:

 1. Building the arm-linux-gnueabihf-g++ toolchain
 2. Compiling libtensorflow-lite.a and minimal
 3. Building piwheel for tensorflow lite
 4. Alternative: Using the precompiled Wheels



## 1. Building arm-linux-gnueabihf-g++

This part of the guide is competly taken from [here](https://gist.github.com/sol-prog/94b2ba84559975c76111afe6a0499814). Give it a star if you use this.

**Sources:**

```
http://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/
https://www.raspberrypi.org/documentation/linux/kernel/building.md


https://wiki.osdev.org/Why_do_I_need_a_Cross_Compiler%3F
https://wiki.osdev.org/GCC_Cross-Compiler
https://wiki.osdev.org/Building_GCC
http://www.ifp.illinois.edu/~nakazato/tips/xgcc.html
```


1. Update system
```
sudo apt update
sudo apt upgrade
```

2. Install prereq:
```
sudo apt install build-essential gawk git texinfo bison
```

3. Download software:
```
wget https://ftpmirror.gnu.org/binutils/binutils-2.30.tar.bz2
wget https://ftpmirror.gnu.org/gcc/gcc-8.1.0/gcc-8.1.0.tar.gz
wget https://ftpmirror.gnu.org/glibc/glibc-2.27.tar.bz2
git clone --depth=1 https://github.com/raspberrypi/linux
```

4. Extract archives and remove them
```
tar xf binutils-2.30.tar.bz2
tar xf glibc-2.27.tar.bz2
tar xf gcc-8.1.0.tar.gz
rm *.tar.*
```

5. Get GCC prereq
```
cd gcc-*
contrib/download_prerequisites
rm *.tar.*
cd ..
```

6. Create the folder in which we'll put the cross compiler and add it to the PATH
```
sudo mkdir -p /opt/cross-pi-gcc
sudo chown $USER /opt/cross-pi-gcc
export PATH=/opt/cross-pi-gcc/bin:$PATH
```

7. Build binutils
```
mkdir build-binutils && cd build-binutils
../binutils-2.30/configure --prefix=/opt/cross-pi-gcc --target=arm-linux-gnueabihf --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib
make -j 8
make install
```

8. Install kernel headers
```
# See also https://www.raspberrypi.org/documentation/linux/kernel/building.md

cd linux
KERNEL=kernel7
make ARCH=arm INSTALL_HDR_PATH=/opt/cross-pi-gcc/arm-linux-gnueabihf headers_install
```

9. Build compilers
```
cd ..
mkdir build-gcc && cd build-gcc
../gcc-8.1.0/configure --prefix=/opt/cross-pi-gcc --target=arm-linux-gnueabihf --enable-languages=c,c++,fortran --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib
make -j8 all-gcc
make install-gcc
```

10. Partially build glibc
```
cd ..
mkdir build-glibc && cd build-glibc
../glibc-2.27/configure --prefix=/opt/cross-pi-gcc/arm-linux-gnueabihf --build=$MACHTYPE --host=arm-linux-gnueabihf --target=arm-linux-gnueabihf --with-arch=armv6 --with-fpu=vfp --with-float=hard --with-headers=/opt/cross-pi-gcc/arm-linux-gnueabihf/include --disable-multilib libc_cv_forced_unwind=yes
make install-bootstrap-headers=yes install-headers
make -j8 csu/subdir_lib
install csu/crt1.o csu/crti.o csu/crtn.o /opt/cross-pi-gcc/arm-linux-gnueabihf/lib
arm-linux-gnueabihf-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o /opt/cross-pi-gcc/arm-linux-gnueabihf/lib/libc.so
touch /opt/cross-pi-gcc/arm-linux-gnueabihf/include/gnu/stubs.h
```

11. Build compiler support library
```
cd ..
cd build-gcc
make -j8 all-target-libgcc
make install-target-libgcc
```

12. Finish building Glibc
```
cd ..
cd build-glibc
make -j8
make install
```

13. Finish building GCC
```
cd ..
cd build-gcc
make -j8
make install
cd ..
```
Additional Note:
If input now into your shell:
```
arm-linux-gnueabihfg++
```
it should say something like:
```
arm-linux-gnueabihf-g++: fatal error: no input files
compilation terminated.
```
That means that arm-linux-gnueabihfg++ is in path as it should be.

## **Compiling libtensorflow-lite.a and minimal**

1. Clone tensorflow:

```
git clone https://github.com/tensorflow/tensorflow.git tensorflow_src
```

2. Download dependencies tensorflow lite:

```
cd tensorflow_src && ./tensorflow/lite/tools/make/download_dependencies.sh
```

3. Use the makefile through the build script provided by tensorflow to compile libtensorflow-lite.a and minimal:
```
./tensorflow/lite/tools/make/build_rpi_lib.sh TARGET_ARCH=armv6 TARGET_TOOLCHAIN_PREFIX=arm-linux-gnueabihf-  >errors.txt 2>&1
```
(this will also pipe errors into a file called errors.txt in the same dictionary)
After the process is finished check errors.txt and check ./tensorflow/lite/tools/make/gen/ .
It should contain a folder called rpi_armv6 containing the three folders: bin, lib and obj.
libtensorflow-lite.a can be used to build python wheels while minimal can run on the pi to load a model. (its basicly a tensorflow hello world in c++)
