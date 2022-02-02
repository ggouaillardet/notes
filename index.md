FujitsuClang, cmake and fapp

let's have a look at the following trivial OpenMP C++ program

```
#include <stdio.h>
#include <omp.h>

int main(int argc, char *argv[]) {
#pragma omp parallel
#pragma omp master
    printf("there are %d threads\n", omp_get_num_threads());
}
```

and the `CMakeLists.txt` file below


```
cmake_minimum_required(VERSION 3.10)

# set the project name and version
project(Test VERSION 1.0)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

find_package(OpenMP)

add_executable(omp omp.cpp)

target_link_libraries(omp PUBLIC OpenMP::OpenMP_CXX)
```

Now let's build it with the latest CMake 3.22.2 on a Fugaku compute node with Fujitsu C++ compiler (FCC) in clang mode

```
$ module purge

$ module load lang/tcsds-1.2.34

$ export FCC_ENV=-Nclang

$ ~/local/cmake-3.22.2-linux-aarch64/bin/cmake -DCMAKE_CXX_COMPILER=FCC ..
-- The C compiler identification is GNU 8.4.1
-- The CXX compiler identification is FujitsuClang 4.7.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /opt/FJSVxtclanga/tcsds-1.2.34/bin/FCC - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Found OpenMP_C: -fopenmp (found version "4.5") 
-- Found OpenMP_CXX: -fopenmp (found version "4.5") 
-- Found OpenMP: TRUE (found version "4.5")  
-- Configuring done
-- Generating done
-- Build files have been written to: /home/rist/r00018/src/cmake-test/build

$ make
[ 50%] Building CXX object CMakeFiles/omp.dir/omp.cpp.o
[100%] Linking CXX executable omp
[100%] Built target omp
```

The program runs as expected
```
$ ./omp
there are 48 threads
```

Now let's profile it with `fapp`

```
$ fapp -C -d fapp ./omp
fapp: internal error : Tools environment unauthorized thread <1152921500311879680> access
fapp: internal error : Tools environment unauthorized thread <1152921500311879680> access
```

`fapp` fails with a cryptic error message:

Let's see how the binary was linked

```
$ rm omp
[r00018@i34-3108c build]$ make VERBOSE=1
/vol0004/rist/r00018/local/cmake-3.22.2-linux-aarch64/bin/cmake -S/home/rist/r00018/src/cmake-test -B/home/rist/r00018/src/cmake-test/build --check-build-system CMakeFiles/Makefile.cmake 0
/vol0004/rist/r00018/local/cmake-3.22.2-linux-aarch64/bin/cmake -E cmake_progress_start /home/rist/r00018/src/cmake-test/build/CMakeFiles /home/rist/r00018/src/cmake-test/build//CMakeFiles/progress.marks
make  -f CMakeFiles/Makefile2 all
make[1]: Entering directory '/vol0004/rist/r00018/src/cmake-test/build'
make  -f CMakeFiles/omp.dir/build.make CMakeFiles/omp.dir/depend
make[2]: Entering directory '/vol0004/rist/r00018/src/cmake-test/build'
cd /home/rist/r00018/src/cmake-test/build && /vol0004/rist/r00018/local/cmake-3.22.2-linux-aarch64/bin/cmake -E cmake_depends "Unix Makefiles" /home/rist/r00018/src/cmake-test /home/rist/r00018/src/cmake-test /home/rist/r00018/src/cmake-test/build /home/rist/r00018/src/cmake-test/build /home/rist/r00018/src/cmake-test/build/CMakeFiles/omp.dir/DependInfo.cmake --color=
Dependencies file "CMakeFiles/omp.dir/omp.cpp.o.d" is newer than depends file "/home/rist/r00018/src/cmake-test/build/CMakeFiles/omp.dir/compiler_depend.internal".
Consolidate compiler generated dependencies of target omp
make[2]: Leaving directory '/vol0004/rist/r00018/src/cmake-test/build'
make  -f CMakeFiles/omp.dir/build.make CMakeFiles/omp.dir/build
make[2]: Entering directory '/vol0004/rist/r00018/src/cmake-test/build'
[ 50%] Linking CXX executable omp
/vol0004/rist/r00018/local/cmake-3.22.2-linux-aarch64/bin/cmake -E cmake_link_script CMakeFiles/omp.dir/link.txt --verbose=1
/opt/FJSVxtclanga/tcsds-1.2.34/bin/FCC CMakeFiles/omp.dir/omp.cpp.o -o omp  /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomp.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90i.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90fmt_sve.a /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90f.so -lfjsrcinfo /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjcrt.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjompcrt.so /usr/lib/gcc/aarch64-redhat-linux/8/libatomic.so 
make[2]: Leaving directory '/vol0004/rist/r00018/src/cmake-test/build'
[100%] Built target omp
make[1]: Leaving directory '/vol0004/rist/r00018/src/cmake-test/build'
/vol0004/rist/r00018/local/cmake-3.22.2-linux-aarch64/bin/cmake -E cmake_progress_start /home/rist/r00018/src/cmake-test/build/CMakeFiles 0
```

The link command line is
```
/opt/FJSVxtclanga/tcsds-1.2.34/bin/FCC CMakeFiles/omp.dir/omp.cpp.o -o omp  /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomp.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90i.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90fmt_sve.a /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90f.so -lfjsrcinfo /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjcrt.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjompcrt.so /usr/lib/gcc/aarch64-redhat-linux/8/libatomic.so 
```

As we can see, CMake chose **not** to use the `-fopenmp` option, but instead exlicitly link OpenMP libraries and dependencies.
The root cause of the fapp failure is CMake did not explicitly link the binary with `/opt/FJSVxtclanga/tcsds-1.2.34/clang-comp/bin/../../lib64/fjomp.o`.
I noted `/opt/FJSVxtclanga/tcsds-1.2.34/clang-comp/bin/../../lib64/fjcrt0.o` and `/opt/FJSVxtclanga/tcsds-1.2.34/clang-comp/bin/../../lib64/fjlang08.o` were also not linked,
but that did not seem to influence the outcome of `fapp`.


Now let's try to fix this:

 1. the bad:
    let's pass `-fopenmp` to the linker

```diff
diff -ruN orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake
--- orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake   2022-01-25 22:56:11.000000000 +0900
+++ cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake        2022-02-01 11:39:02.000000000 +0900
@@ -573,6 +573,10 @@
         if(CMAKE_${LANG}_COMPILER_ID STREQUAL "Fujitsu")
           set_property(TARGET OpenMP::OpenMP_${LANG} PROPERTY
             INTERFACE_LINK_OPTIONS "${OpenMP_${LANG}_FLAGS}")
+        elseif(CMAKE_${LANG}_COMPILER_ID STREQUAL "FujitsuClang")
+          # OpenMP flags for Fujitsu need to be passed to both compiler AND linker
+          set_property(TARGET OpenMP::OpenMP_${LANG} PROPERTY
+            INTERFACE_LINK_OPTIONS "$<$<COMPILE_LANGUAGE:${LANG}>:${_OpenMP_${LANG}_OPTIONS}>")
         endif()
         unset(_OpenMP_${LANG}_OPTIONS)
       endif()
```

It kind of works: the `-fopenmp` option is passed to the linker, and `fapp` is a happy panda

```
/opt/FJSVxtclanga/tcsds-1.2.34/bin/FCC -fopenmp CMakeFiles/omp.dir/omp.cpp.o -o omp  /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomp.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90i.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90fmt_sve.a /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90f.so -lfjsrcinfo /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjcrt.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjompcrt.so /usr/lib/gcc/aarch64-redhat-linux/8/libatomic.so
```

But it is also (kind of) bad since the `-fopenmp` option is redundant with explicitly linking OpenMP libraries and dependencies.

 2. the ugly
   let's (try to) do it right and have CMake link with `fjomp.o`!
```diff
diff -ruN orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-C.cmake cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-C.cmake
--- orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-C.cmake      2022-01-25 22:56:11.000000000 +0900
+++ cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-C.cmake   2022-02-01 11:53:39.000000000 +0900
@@ -4,3 +4,4 @@
 set(CMAKE_C_COMPILER_VERSION "${CMAKE_C_COMPILER_VERSION_INTERNAL}")
 include(Compiler/Clang-C)
 set(CMAKE_C_COMPILER_VERSION "${_fjclang_ver}")
+set(CMAKE_CXX_IMPLICIT_OBJECT_REGEX "/fjomp\\.o$")
diff -ruN orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-CXX.cmake cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-CXX.cmake
--- orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-CXX.cmake    2022-01-25 22:56:11.000000000 +0900
+++ cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/Compiler/FujitsuClang-CXX.cmake 2022-02-01 11:53:37.000000000 +0900
@@ -4,3 +4,4 @@
 set(CMAKE_CXX_COMPILER_VERSION "${CMAKE_CXX_COMPILER_VERSION_INTERNAL}")
 include(Compiler/Clang-CXX)
 set(CMAKE_CXX_COMPILER_VERSION "${_fjclang_ver}")
+set(CMAKE_CXX_IMPLICIT_OBJECT_REGEX "/fjomp\\.o$")
```
   that should do it ... except it does not :-(
```
$ make
[ 50%] Building CXX object CMakeFiles/omp.dir/omp.cpp.o
[100%] Linking CXX executable omp
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/fjomp.o: In function `__jwe_compiler_OMP':
jwe_xomp.c:(.text+0x0): multiple definition of `__jwe_compiler_OMP'
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/fjomp.o:jwe_xomp.c:(.text+0x0): first defined here
CMakeFiles/omp.dir/omp.cpp.o: In function `main':
/home/rist/r00018/src/cmake-test/omp.cpp:5: undefined reference to `__kmpc_fork_call'
CMakeFiles/omp.dir/omp.cpp.o: In function `.omp_outlined.':
/home/rist/r00018/src/cmake-test/omp.cpp:6: undefined reference to `__kmpc_master'
/home/rist/r00018/src/cmake-test/omp.cpp:7: undefined reference to `__kmpc_end_master'
/home/rist/r00018/src/cmake-test/omp.cpp:7: undefined reference to `__kmpc_end_master'
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so: undefined reference to `__kmp_threads'
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so: undefined reference to `__kmp_init_parallel'
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so: undefined reference to `__kmp_global'
/opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so: undefined reference to `__kmp_init_implicit_task'
clang-7: error: linker command failed with exit code 1 (use -v to see invocation)
make[2]: *** [CMakeFiles/omp.dir/build.make:108: omp] Error 1
make[1]: *** [CMakeFiles/Makefile2:83: CMakeFiles/omp.dir/all] Error 2
make: *** [Makefile:91: all] Error 2
```

   and that is the root cause (from `CMakeCache.txt`)

```
OpenMP_CXX_LIB_NAMES:STRING=fjomp;fjomphk;fjomp;fj90i;fj90fmt_sve;fj90f;fjsrcinfo;fjcrt;fjompcrt;atomic

```

   yep, `fjomp.o` and `libfjomp.so` are both referred as `fjomp`.
   That leads to `libfjomp.so` not being passed to the linker, and hence the linker error.
   So let's fix that - and that's why it is ugly

```
diff -ruN orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake
--- orig/cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake   2022-01-25 22:56:11.000000000 +0900
+++ cmake-3.22.2-linux-aarch64/share/cmake-3.22/Modules/FindOpenMP.cmake        2022-02-01 12:02:38.000000000 +0900
@@ -247,6 +247,9 @@
           get_filename_component(_OPENMP_IMPLICIT_LIB_DIR "${_OPENMP_IMPLICIT_LIB}" DIRECTORY)
           get_filename_component(_OPENMP_IMPLICIT_LIB_NAME "${_OPENMP_IMPLICIT_LIB}" NAME)
           get_filename_component(_OPENMP_IMPLICIT_LIB_PLAIN "${_OPENMP_IMPLICIT_LIB}" NAME_WE)
+          if (_OPENMP_IMPLICIT_LIB_NAME MATCHES "\\.o$")
+              set(_OPENMP_IMPLICIT_LIB_PLAIN "${_OPENMP_IMPLICIT_LIB_PLAIN}_o")
+          endif()
           string(REGEX REPLACE "([][+.*?()^$])" "\\\\\\1" _OPENMP_IMPLICIT_LIB_PLAIN_ESC "${_OPENMP_IMPLICIT_LIB_PLAIN}")
           string(REGEX REPLACE "([][+.*?()^$])" "\\\\\\1" _OPENMP_IMPLICIT_LIB_PATH_ESC "${_OPENMP_IMPLICIT_LIB}")
           if(NOT ( "${_OPENMP_IMPLICIT_LIB}" IN_LIST CMAKE_${LANG}_IMPLICIT_LINK_LIBRARIES
```

   that did the trick, fwiw, here is the link command line:
```
/opt/FJSVxtclanga/tcsds-1.2.34/bin/FCC CMakeFiles/omp.dir/omp.cpp.o -o omp  /opt/FJSVxtclanga/tcsds-1.2.34/lib64/fjomp.o /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomphk.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjomp.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90i.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90fmt_sve.a /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfj90f.so -lfjsrcinfo /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjcrt.so /opt/FJSVxtclanga/tcsds-1.2.34/lib64/libfjompcrt.so /usr/lib/gcc/aarch64-redhat-linux/8/libatomic.so 
```
 
   3. The good: well I cannot do that alone ... My best preference is to get rid of the ugly patch, which means either `fjomp.o` or `libfjomp.so` should be renammed into something that does not cause any conflict with CMake way of handling things.
