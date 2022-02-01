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
 
