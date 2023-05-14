# Template SIMD Library (TSL)

## **Current Status**

### Library Generation using python

[![py38](./doc/badges/generate_py3.8.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)
[![py39](./doc/badges/generate_py3.9.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)
[![py310](./doc/badges/generate_py3.10.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)
[![py311](./doc/badges/generate_py3.11.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)

### Building on __Intel__ cores with SSE, AVX(2) and AVX512, __Ubuntu (latest)__
[![build_g++](./doc/badges/build_g++.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)
[![build_clang++](./doc/badges/build_clang++.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)


### Running test cases on __Intel__ cores with SSE, AVX(2) and AVX512, __Ubuntu (latest)__
[![test_g++](./doc/badges/test_g++.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)
[![test_clang++](./doc/badges/test_clang++.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)


### Docker image containing all requirements
[![dockerio](./doc/badges/docker.io.svg)](https://github.com/db-tu-dresden/TVLGen/actions/workflows/tslgen_merge_main.yml)


---

|Table of contents|
|:--|
[1. About the TSL](#tsl-intro)|
[2. Motivation](#tsl-motivation)|
[3. Prerequisites](#tsl-prerequisites)|
[4. Usage](#tsl-usage)|
[4.1 Integration into your project](#tsl-include)|
[4.2 Starting from scratch](#tsl-scratch)|
[5. Contribute](tsl-contribute)|

---
## <a id="tsl-intro"></a>About the TSL

TSL is a C++ _template header-only library_ which abstracts elemental operations on hardware specific instructions sets, with main focus on SIMD operations. 
Consequently, you can use SIMD functionality in a hardware agnostic way and the library will take care of the adequate mapping. 
The provided functionality (aka. "primitives") is either directly mapped to the underlying hardware or uses scalar workarounds. 
Thus, you can be sure, that your code is compilable on every supported platform and does what you would expect, however the performance may differ depending on the available hardware features. 

As you may notice, this repository mainly consists of _python_ instead of C++ code. 
This is, because we decided to write a library-generator rather than a "one-size-fits-all" library. 

### _Why the indirection through a generator? Why no plain header only library?_

First of all, it is a very hard, laborious, and maybe even impossible task to design a hardware abstraction library, which suits all currently available and upcoming hardware. 
So we didn't even try but decided to make it considerably simple to change nearly every detail of the library provided interface and implementation. 
To enable such flexibility, we wrote a generator that relies on Jinja2 - a powerful and mature template engine. 
Through our generator, it is possible to "cherry-pick" the generated library. 
Consequently, the generated TSL will be tailored to your specific underlying hardware. 
While this seems to be unnecessary at first glance, through (i) modern compilers also support hardware detection for auto-vectorization and (ii) non-compliant code parts can be disabled using preprocessor macros, we want to support (i) explicit vectorization and reduce the code size and compile time and increase the readability of the to be shipped library (ii).

Second, as the TSL is a template library, there is a lot of redundant code (when looking into the generated files, this should be clear!). 
We argue that a generator is a perfect fit to reduce the manual effort which ultimately makes it easier to extend the library and adopt to disruptive change requests.

### _Why are there missing functionality?_

The TSL is developed for effectively explore the capabilities of SIMD for In-Memory-Databases at the [Dresden Database Research Group](wwwdb.inf.tu-dresden.de).
We develop and add functionality to the TSL using a top-down approach: if we need a particular functionality for an algorithm, we find a suitable level of generality and add new capability to the library. 
Since we are IMDB people, we probably miss out on interesting features due to a lack of necessity. 
If you miss any operation relevant for your specific problem, please don't hesitate to add the functionality and contribute to the library so that everyone can benefit from! 
For a detailed explanation on how to contribute, see section contribute. <!-- Todo: Add link -->


### _What are the target plattform?_

We tested the generator and the TSL on different Linux distributions (Centos7, Ubuntu 22.04, Arch Linux) and the WSL using python 3.8-3.11 and clang/gcc.

[Go back to toc](#toc)

---
## <a id="tsl-motivation"></a>**Motivation**

Implementing high-performance algorithms entail - among other things - hardware near optimizations. 
That is, why most of the performance-crucial algorithms are written using C/C++. 
This language family provide the ability to facilitate hardware specific capabilities like special Instruction Set Extensions like <b>S</b>ingle **I**nstruction **M**ultiple **D**ata. 
SIMD is a standard technique to speed up compute-heavy tasks by applying the same instruction on multiple data elements in parallel (data parallelism). 

SIMD can be used indirectly by taking advantage of the "auto-vectorization" capabilities of modern compilers. 
However, to enable compilers to detect simdifiable parts, the code must be annotated and follow specific rules (see [gcc docu](https://gcc.gnu.org/projects/tree-ssa/vectorization.html), [clang docu](https://llvm.org/docs/Vectorizers.html)). 
While this seems to be a good choice at first glance, making a given code auto-vectorizable can be very laborious, since the emitted assembler instructions should be checked to ensure that SIMD was applied properly (which may differ between compilers and even different versions of the same compiler). 

Another possibility of facilitating existing hardware instructions is by explicit simdification through vendor specific libraries (see [intel](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html), [arm](https://developer.arm.com/architectures/instruction-sets/intrinsics/)).
Those libraries expose functions or macros (so called _intrinsics_) to access the underlying functionality. 
By using explicit simdification, a developer can ensure, that the intended hardware provided functionality is used. 
We argue, that using explicit simdification provides a developer with the necessary tools to tailor and optimize an algorithm to a specific hardware to the maximum extent.

However, using vendor specific libraries come with the downside of reduced or even nonexistent portability of the code. 
To cope with that issue, different approaches exist which fall, without loss of generality, into two categories: (i) code translation, e.g. [SIMD everywhere](https://github.com/simd-everywhere/simde), and (ii) template libraries, e.g. [Highway](https://github.com/google/highway), [xsimd](https://github.com/xtensor-stack/xsimd). 
__TSL__ falls in the latter category, providing C++-_function-templates_ which are specialized for given hardware ISAs and datatypes. 

[Go back to toc](#toc)

---

## <a id="tsl-prerequisites"></a>**Prerequisites**

### **1. Getting the repository**

Clone the git repositiory of the [TSL Generator](https://github.com/db-tu-dresden/TVLGen) (or one of the forks) and navigate into your freshly cloned TSLGen directory.

~~~console
user@host:~ git clone https://github.com/db-tu-dresden/TVLGen tslroot
user@host:~ cd tslroot
~~~

### **2. Initialize required environemnt**

To generate the TSL you need python and some additional python packages. You can either [manually](#custom-install) setup the environemnt or use a prebuild [docker image](#docker-install).

 #### <a id="custom-install"></a>**Install dependencies manually (Option 1)**

In the following section we will give an explanation on how to install the requirements for Linux environments which uses ```apt``` as package manager (Debian / Ubuntu). 
For other linux distribution you may have to adopt the package manager as well as the package names. 

~~~console
# udpate apt
user@host:~/tslroot  sudo apt update
# install TSL-generator specific dependencies
user@host:~/tslroot  sudo apt -y install python3.10 graphviz-dev
# install required CMake version
user@host:~/tslroot  sudo apt -y install cmake
# make sure, that the CMake version is at least 3.19
user@host:~/tslroot  cmake --version
# install pip
user@host:~/tslroot  python3 -m pip install --upgrade pip
~~~

As the next step, install all python dependencies for instance using pip:

~~~console
user@host:~/tslroot  pip install -r ./requirements.txt
~~~
> **Note:** Please ensure, that all packages could be installed properly, otherwise the generator may fail.

<!-- To generate and build the TSL please look at section [Host Build](#custom-build)-->

#### <a id="docker-install"></a>**Install docker and pull the image (Option 2)**

Make sure you installed the [Docker engine](https://docs.docker.com/engine/install/) properly. 
If everything is up and running, just pull the image and tag it:

~~~console
user@host:~/tslroot  docker pull jpietrzyktud/tslgen:latest
user@host:~/tslroot  docker tag jpietrzyktud/tslgen:latest tslgen:latest
~~~

The provided image defines a console as entry point and exposes a path `/tslgen` as mount point for the host maschine.

[Go back to toc](#toc)

---


## <a id="tsl-usage"></a>**Usage**

In the following section we distinguish between two use-cases: (i) using the TSL within an existing project (or start from scratch), and (ii) contribute primitives and extensions. 

### <a id="tsl-include"></a>__Using the TSL in your project__ (recommended for users)

Assuming you have the following existing project structure:
```
project
|   CMakeLists.txt          #<-- top level CMakelists.txt of the project
|   Readme.md
|
+--+src
|  |   ...
|
+--+include
|  |   ...
|
+--+libs
+--+tools
   |
   +--+tsl
      |   main.py           #<-- TSL Generator script
      |   CMakeLists.txt    #<-- TSL Generator CMakelists.txt
      |
      +--+generator
      |
      +--+primitive_data
```
In the given scenario, `./libs` contains third-party library code and `./tools` contain third-party tools and scripts. 
The TSL generator was added as git _submodule_, _subtree_ or simply as a _sub repository_ to `./tools/tsl`.
As you may have noticed, in the top level directory of the TSL Generator there is a CMakeLists.txt. 
This can be directly used within your projects top-level CMakeLists:
~~~cmake
# ...
project(<PROJECTNAME>)
# ...
add_subdirectory(${PROJECT_SOURCE_DIR}/tools/tsl)
add_executable(<target> [source1] [source2...])
target_include_directories(<target> INTERFACE ${PROJECT_SOURCE_DIR}/libs/tsl)
target_link_libraries(<target> INTERFACE ${PROJECT_SOURCE_DIR}/libs/tsl)
~~~
When call cmake in your project root, the `./tools/tsl/CMakeLists.txt` will be invoked. 
This identifies the capabilities of your hardware and generates a hardware tailored TSL, which than will be accessible from your files. 
Thus, the only thing you have to do is to include the top-level TSL header file. 
This will include all necessary files properly.
~~~cpp
#include <tslintrin.h>
// now we can access the TSL functionality through their namespace:
int main() {
  // now we can access the TSL functionality through their namespace:
  auto _vec = tsl::set1<tsl::avx2, uint32_t>(42);
  // of course you can also use the namespace
  using namespace tsl;
  to_ostream<avx2, uint32_t>(std::cout, _vec) << std::endl;
  // result: [42,42,42,42,42,42,42,42]
  return 0;
}
~~~
> **Note:** Don't include other files from the TSL directory, as they depend on the include order of tvlintrin.h.

Now, since everything is up and running, please refer to the [Getting Started](doc/GettingStarted.md) page to find more examples. 

[Go back to toc](#toc)


### <a id="tsl-scratch"></a>__Starting from scratch__ (recommended for contributors)

Assuming you have the following TSL Generator project structure:
```
tsl
|   main.py
|   CMakeLists.txt          #<-- top level CMakelists.txt of the TSL Generator
|   Readme.md
|   
+--+generator               #<-- All generator associated files are placed here
|  |...
|
+--+supplementary           #<-- Supplementary files like RTL codes reside here
|  |...
|
+--+primitive_data
   |
   +--+extensions           #<-- Hardware extension structs are defined here
   |  |
   |  +--+simd              #<-- Hardware extension structions for SIMD processing
   |  |  |
   |  |  +--+intel          #<-- Intel specific structs (e.g., sse, avx,...)
   |  |  |
   |  |  +--+arm            #<-- ARM specific structs (e.g., Neon)
   |  |...
   |  
   +--+primitives           #<-- TSL Primitives are defined here
      |
      |   binary.yaml       #<-- all primitives which fall in the category of __binary operations__
      |   calc.yaml         #<-- all primitives which fall in the category of __arithmetic operations__
      |   ...
```

As the TSL is tailored to the underlying hardware we have to provide the specification of the system to the generator.
The code shown below can be used to check what FLAGS are exposed by your hardware (the result is produced by an i7-8550U).
~~~console
user@host:~/tsl  LSCPU_FLAGS=$(LANG=en;lscpu|grep -i flags | tr ' ' '\n' | grep -E -v '^Flags:|^$' | sort -d | tr '\n' ' ')
user@host:~/tsl  echo $LSCPU_FLAGS
3dnowprefetch abm acpi adx aes aperfmperf apic arat arch_capabilities arch_perfmon art avx avx2 bmi1 bmi2 bts clflush clflushopt cmov constant_tsc cpuid cpuid_fault cx16 cx8 de ds_cpl dtes64 dtherm dts epb ept ept_ad erms est f16c flexpriority flush_l1d fma fpu fsgsbase fxsr ht hwp hwp_act_window hwp_epp hwp_notify ibpb ibrs ida intel_pt invpcid invpcid_single lahf_lm lm mca mce md_clear mmx monitor movbe mpx msr mtrr nonstop_tsc nopl nx pae pat pbe pcid pclmulqdq pdcm pdpe1gb pebs pge pln pni popcnt pse pse36 pti pts rdrand rdseed rdtscp rep_good sdbg sep sgx smap smep ss ssbd sse sse2 sse4_1 sse4_2 ssse3 stibp syscall tm tm2 tpr_shadow tsc tsc_adjust tsc_deadline_timer vme vmx vnmi vpid x2apic xgetbv1 xsave xsavec xsaveopt xsaves xtopology xtpr
~~~
This list can than be passed to the generator:
~~~console
user@host:~/tsl  python3 ./main.py --targets ${LSCPU_FLAGS} -o ./lib
# you can fillter for types, which may come in handy for fast prototyping
user@host:~/tsl  python3 ./main.py --targets ${LSCPU_FLAGS} -o ./libUI64 --types "uint64_t"
# you can filter for primitives (mainly for development reasons)
user@host:~/tsl  python3 ./main.py --targets ${LSCPU_FLAGS} -o ./libRNDMem --primitives "gather scatter"
# get a list of all available arguments:
user@host:~/tsl  python3 ./main.py -h
~~~
> **Note:** If your compiler does not support C++-20 concepts, you should use the `--no-concepts` argument to avoid the generator using concepts.

A more convenient way is to use the provided top-level CMakeLists.txt to do all of this in one go:
~~~console
user@host:~/tsl  cmake -S . -D GENERATOR_OUTPUT_PATH=<path to output directory> -B lib
~~~
> **Note:** You can pass further flags which are consumed by the generator using the `-D` notation.

Under the hood, cmake utilizes py-cpuinfo to detect system specific hardware properties (i.e. which cpu flags are available) and hand them over to the generator.
This way the generated TSL library is tailored for your system.
But you can also specify the cpu flags manually, if you want to.
For more information on how to do that, take a look at the [Generator Usage](GeneratorUsage.md) page.

[Go back to toc](#toc)

### <a id="tsl-tests"></a>__Build and run primitive tests__ (recommended for contributors)

If you want to build the generated TSL together with the specified tests, utilizing [Catch2](https://github.com/catchorg/Catch2) run the following commands after CMake:
~~~console
user@host:~/tsl make -C lib
user@host:~/tsl ./lib/generator_output/build/src/test/tsl_test
# <output truncated>
===============================================================================
All tests passed (968 assertions in 93 test cases)
## You can list all tests using the -l flag. On the example maschine the output looks like the following:
user@host:~/tsl ./lib/generator_output/build/src/test/tsl_test -l
#<output truncated>
  Testing primitive 'mask_equal' for sse
      [mask_equal][sse]
  Testing primitive 'convert_down' for avx2
      [avx2][convert_down]
  Testing primitive 'convert_down' for scalar
      [convert_down][scalar]
  Testing primitive 'convert_down' for sse
      [convert_down][sse]
  Testing primitive 'convert_up' for avx2
      [avx2][convert_up]
  Testing primitive 'convert_up' for scalar
      [convert_up][scalar]
  Testing primitive 'convert_up' for sse
      [convert_up][sse]
93 test cases
## And you can test specific primitives:
user@host:~/tsl ./lib/generator_output/build/src/test/tsl_test "[equal]"
Filters: [equal]
#<output truncated>
===============================================================================
All tests passed (56 assertions in 3 test cases)
## You can filter for specific extensions
user@host:~/tsl ./lib/generator_output/build/src/test/tsl_test "[avx2]"
Filters: [avx2]
#<output truncated>
===============================================================================
All tests passed (328 assertions in 31 test cases)
## And you can filter for both
user@host:~/tsl ./lib/generator_output/build/src/test/tsl_test "[equal][avx2]"
Filters: [equal][avx2]
===============================================================================
All tests passed (20 assertions in 1 test case)
~~~

[Go back to toc](#toc)

## <a id="tsl-contribute"></a>**Contribute**

As we are a small group mainly focussed on In-Memory-Database we certainly miss crucial functionality which is needed by algorithms from other domains or even from our own. 
That is why we would highly appreciate it, if you can bring in your own expertise and use-cases and contribute to the existing library. 
To help you with this, we developed a [Visual Studio Code Extension](https://marketplace.visualstudio.com/items?itemName=DBTUD.tslgen-edit) that provides useful features like TSL-specific auto-completion, code-preview generation, ad-hoc build and testing and much more. 
We would highly recommend to use this, since writing YAML files can be quite nerve-racking some time ;).

[Go back to toc](#toc)

## Note

Our template library was formerly denotated as **T**emplate **V**ector **L**ibrary (TVL). 
However, as we are database people, we frequently experienced confusion regarding the term _Vector_ (since it is predominately associated with batched processing). 
To prevent any confusion, we decided to rename our library to **T**emplate **S**IMD **L**ibrary (TSL).

## Todo 

- [ ] Readme
  - [ ] Explain the differences between (xsimd, ghy) and (tsl) 
  - [ ] Proofread
  - [ ] Reproduce the workflow for the Prerequisites and Usage section
  - [ ] Explain how (if possible) the docker workflow can be integrated into an existing CMake-Project
