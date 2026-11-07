[Github source for fuzzing harness](https://github.com/suitengu/fuzzing_investigations/tree/main/assimp-fuzz)

# Fuzzing assimp with afl++ writeup
A harness and notes for fuzzing the assimp asset importer library.

*Currently only FBX format is targeted.*

## Notes
This build was made for a WSL2 environment, so some package versions may be unusual and some build issues may be unnecessary when compared to a more current Linux environment.

## Checking out the assimp source to ./assimp that will be instrumented by afl++
``` bash
git clone git@github.com:assimp/assimp.git
```

## Setup the seeds/inputs dir
This is likely overkill and could be refined, but I just used all the test .fbx models that ship with assimp. Some of these may be highly nonstandard or sub-optimal for fuzzing, but using larger, real scenes (ie. Sponza) would be much worse.

``` bash
mkdir inputs
cp assimp/test/models/FBX/*.fbx inputs/
cp assimp/test/models-nonbsd/FBX/2013_ASCII/*.fbx inputs/
cp assimp/test/models-nonbsd/FBX/2013_BINARY/*.fbx inputs/
```

## Configuring and building assimp with and without AFL instrumentation
(On WFL, I first had to edit out all occurrences of `-Wno-deprecated-non-prototype` in the CMakeLists.txt files, because the versions of clang++ and afl-clang++ available on WSL2 do not support this option and the build fails. On an up-to-date Linux distro, this is not a problem)

``` bash
cd assimp
cmake -DCMAKE_C_COMPILER=afl-clang-fast -DCMAKE_CXX_COMPILER=afl-clang-fast++ -DBUILD_SHARED_LIBS=OFF -DASSIMP_BUILD_ZLIB=ON -DCMAKE_BUILD_TYPE=Release -B build-afl
cmake --build build-afl --parallel $(nproc)
cmake -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DBUILD_SHARED_LIBS=OFF -DASSIMP_BUILD_ZLIB=ON -DCMAKE_BUILD_TYPE=Release -B build-test
cmake --build build-test --parallel $(nproc)
```

Release build is required (or to retain Debug build functionality, code that defines the `ai_assert` macro could be modified, but I found this gets too messy). The `ai_assert` macro in Debug builds causes the program to exit on the invalid inputs that afl++ generates, but the macro does nothing in release builds, which is where any interesting findings would likely apply, anyway.

## Running with the fuzzer

``` bash
make clean
make all
afl-fuzz -i inputs/ -o findings/ -- ./harness
```

WSL2 only: issue with `/proc/sys/kernel/core_pattern`
``` bash
sudo su
echo core > /proc/sys/kernel/core_pattern
exit
```

On first run, I was getting crash findings within a few seconds. This seems too "good" to be true -- too easy, right? Using the test program so I can trace through execution with `gdb`:
``` bash
gdb test
run ./findings/default/crashes/id:000000,sig:11,src:001838,time:435378,execs:4388236,op:havoc,rep:2
```

Output:
```
Program received signal SIGSEGV, Segmentation fault.
0x0000555555c54c28 in std::__shared_ptr_access<Assimp::FBX::PropertyTable const, (__gnu_cxx::_Lock_policy)2, false, false>::_M_get() const ()
```

Investigating the backtrace, I see this:

```
bt
#0  0x0000555555c54c28 in std::__shared_ptr_access<Assimp::FBX::PropertyTable const, (__gnu_cxx::_Lock_policy)2, false, false>::_M_get() const ()                                                                                                                     [0/1879]
#1  0x0000555555c54c17 in std::__shared_ptr_access<Assimp::FBX::PropertyTable const, (__gnu_cxx::_Lock_policy)2, false, false>::operator*() const ()
#2  0x0000555555c7ef2c in Assimp::FBX::Model::Props() const ()
#3  0x0000555555c7f085 in Assimp::FBX::Model::RotationOrder() const ()
#4  0x0000555555c64d50 in Assimp::FBX::FBXConverter::GenerateTransformationNodeChain(Assimp::FBX::Model const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, std::vector<Assimp::FBX::FBXConverter::PotentialNode, std::allocator<Assimp::FBX::FBXConverter::PotentialNode> >&, std::vector<Assimp::FBX::FBXConverter::PotentialNode, std::allocator<Assimp::FBX::FBXConverter::PotentialNode> >&) ()
#5  0x0000555555c64267 in Assimp::FBX::FBXConverter::ConvertNodes(unsigned long, aiNode*, aiNode*, aiMatrix4x4t<float> const&) ()
#6  0x0000555555c64700 in Assimp::FBX::FBXConverter::ConvertNodes(unsigned long, aiNode*, aiNode*, aiMatrix4x4t<float> const&) ()
#7  0x0000555555c64700 in Assimp::FBX::FBXConverter::ConvertNodes(unsigned long, aiNode*, aiNode*, aiMatrix4x4t<float> const&) ()
```

(an enormous, multi-page output of lines that are all like the bottom line follows)

The segmentation fault is due to a stack overflow. 

Increasing the stack size via:
``` bash
ulimit -s unlimited
```
results in these cases being reported instead as "hangs" where the program receives SIGKILL instead.

Investigating the cause of this, there is a recursive call to ConvertNodes within FBXConverter.cpp (as of this writing, line 348):

```c++
            // recursion call - child nodes
            ConvertNodes(model->ID(), last_parent, root_node, new_abs_transform);
```

Inspecting the connections segment of the FBX (using some console debugging), I found node 0 has a connection that also refers to node 0, resulting in a never-ending loop of ConvertNodes calls. The current assimp library is written with the reasonable assumption that all real FBX models are directed acyclic graphs, and there are some todo: items for building out the FBX node hierarchy which is likely where a proper fix could be implemented in the future. A quick fix would be to put a recursive depth limit inside FBXConverter, and raise an error when the maximum limit is reached.
