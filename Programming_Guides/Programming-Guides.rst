.. _Programming-Guides:

=================
Programing Guide
=================


HC Programing Guide
===================
**What is the Heterogeneous Compute (HC) API?**
It’s a C++ dialect with extensions to launch kernels and manage accelerator memory. It closely tracks the evolution of C++ and will incorporate parallelism and concurrency features as the C++ standard does. For example, HC includes early support for the C++17 Parallel STL. At the recent ISO C++ meetings in Kona and Jacksonville, the committee was excited about enabling the language to express all forms of parallelism, including multicore CPU, SIMD and GPU. We’ll be following these developments closely, and you’ll see HC move quickly to include standard C++ capabilities.

The Heterogeneous Compute Compiler (HCC) provides two important benefits:

Ease of development

 
   * A full C++ API for managing devices, queues and events
   * C++ data containers that provide type safety, multidimensional-array indexing and automatic data management
   * C++ kernel-launch syntax using parallel_for_each plus C++11 lambda functions
   * A single-source C++ programming environment---the host and source code can be in the same source file and use the same C++     	 language; templates and classes work naturally across the host/device boundary
   * HCC generates both host and device code from the same compiler, so it benefits from a consistent view of the source code using   	   the same Clang-based language parser

Full control over the machine


    * Access AMD scratchpad memories (“LDS”)
    * Fully control data movement, prefetch and discard
    * Fully control asynchronous kernel launch and completion
    * Get device-side dependency resolution for kernel and data commands (without host involvement)
    * Obtain HSA agents, queues and signals for low-level control of the architecture using the HSA Runtime API
    * Use `direct-to-ISA <https://github.com/RadeonOpenCompute/HCC-Native-GCN-ISA>`_ compilation

**When to Use HC**
 Use HC when you're targeting the AMD ROCm platform: it delivers a single-source, easy-to-program C++ environment without compromising performance or control of the machine.

**Download HCC**
 The project now employs git submodules to manage external components it depends upon. It it advised to add --recursive when you clone the project so all submodules are fetched automatically.

For example: ::

  # automatically fetches all submodules
  git clone --recursive -b clang_tot_upgrade https://github.com/RadeonOpenCompute/hcc.git

**Build HCC from source**
*************************

First, install the build dependencies: ::

  # Ubuntu 14.04
  sudo apt-get install git cmake make g++  g++-multilib gcc-multilib libc++-dev libc++1 libc++abi-dev libc++abi1 python findutils     	libelf1 libpci3 file debianutils libunwind8-dev hsa-rocr-dev hsa-ext-rocr-dev hsakmt-roct-dev pkg-config rocm-utils

::  

  # Ubuntu 16.04
  sudo apt-get install git cmake make g++  g++-multilib gcc-multilib python findutils libelf1 libpci3 file debianutils libunwind-     	dev hsa-rocr-dev hsa-ext-rocr-dev hsakmt-roct-dev pkg-config rocm-utils

::

   # Fedora 23/24
   sudo dnf install git cmake make gcc-c++ python findutils elfutils-libelf pciutils-libs file pth rpm-build libunwind-devel   	     	hsa- rocr- dev hsa-ext-rocr-dev hsakmt-roct-dev pkgconfig rocm-utils

Clone the HCC source tree: ::

  # automatically fetches all submodules
  git clone --recursive -b clang_tot_upgrade https://github.com/RadeonOpenCompute/hcc.git

Create a build directory and run cmake to configure the build: ::

  mkdir build; cd build
  cmake ../hcc

Compile HCC: ::

  make -j

Run the unit tests: :: 

  make test

Create an installer package (DEB or RPM file)

::

  make package



To configure and build HCC from source, use the following steps: ::
 
  mkdir -p build; cd build
  # NUM_BUILD_THREADS is optional
  # set the number to your CPU core numbers time 2 is recommended
  # in this example we set it to 96 
  cmake -DNUM_BUILD_THREADS=96 \
      -DCMAKE_BUILD_TYPE=Release \
      ..
  make

To install it, use the following steps: ::
  
  sudo make install

**Use HCC**
***********

For C++AMP source codes: ::

  hcc `clamp-config --cxxflags --ldflags` foo.cpp

For HC source codes: ::
 
 hcc `hcc-config --cxxflags --ldflags` foo.cpp

In case you build HCC from source and want to use the compiled binaries directly in the build directory:

For C++AMP source codes: ::

  # notice the --build flag
  bin/hcc `bin/clamp-config --build --cxxflags --ldflags` foo.cpp

For HC source codes: ::

  # notice the --build flag
  bin/hcc `bin/hcc-config --build --cxxflags --ldflags` foo.cpp

**Compiling for Different GPU Architectures**

By default, HCC would auto-detect all the GPUs available it's running on and set the correct GPU architectures. Users could use the --amdgpu-target=<GCN Version> option to compile for a specific architecture and to disable the auto-detection. The following table shows the different versions currently supported by HCC.

There exists an environment variable HCC_AMDGPU_TARGET to override the default GPU architecture globally for HCC; however, the usage of this environment variable is NOT recommended as it is unsupported and it will be deprecated in a future release.

============ ================== ==============================================================
GCN Version   GPU/APU Family       Examples of Radeon GPU
       
============ ================== ==============================================================
gfx701        GFX7               FirePro W8100, FirePro W9100, Radeon R9 290, Radeon R9 390

gfx801        Carrizo APU        FX-8800P

gfx803        GFX8               R9 Fury, R9 Fury X, R9 Nano, FirePro S9300 x2, Radeon RX 480,
                                 Radeon RX 470, Radeon RX 460

gfx900        GFX9                 Vega10

============ ================== ============================================================== 

Multiple ISA
*************
HCC now supports having multiple GCN ISAs in one executable file. You can do it in different ways:
**use :: --amdgpu-target= command line option**
It's possible to specify multiple --amdgpu-target= option. Example: ::

 # ISA for Hawaii(gfx701), Carrizo(gfx801), Tonga(gfx802) and Fiji(gfx803) would 
 # be produced
 hcc `hcc-config --cxxflags --ldflags` \
    --amdgpu-target=gfx701 \
    --amdgpu-target=gfx801 \
    --amdgpu-target=gfx802 \
    --amdgpu-target=gfx803 \
    foo.cpp

use :: HCC_AMDGPU_TARGET env var

Use , to delimit each AMDGPU target in HCC. Example: ::
  
  export HCC_AMDGPU_TARGET=gfx701,gfx801,gfx802,gfx803
  # ISA for Hawaii(gfx701), Carrizo(gfx801), Tonga(gfx802) and Fiji(gfx803) would 
  # be produced
  hcc `hcc-config --cxxflags --ldflags` foo.cpp

**configure HCC use CMake HSA_AMDGPU_GPU_TARGET variable**
If you build HCC from source, it's possible to configure it to automatically produce multiple ISAs via :: HSA_AMDGPU_GPU_TARGET CMake variable.
Use ; to delimit each AMDGPU target. Example: ::



 # ISA for Hawaii(gfx701), Carrizo(gfx801), Tonga(gfx802) and Fiji(gfx803) would 
 # be produced by default
 cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DROCM_DEVICE_LIB_DIR=~hcc/ROCm-Device-Libs/build/dist/lib \
    -DHSA_AMDGPU_GPU_TARGET="gfx701;gfx801;gfx802;gfx803" \
    ../hcc

**CodeXL Activity Logger**
**************************

To enable the CodeXL Activity Logger, use the  USE_CODEXL_ACTIVITY_LOGGER environment variable.

Configure the build in the following way: ::

  cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DHSA_AMDGPU_GPU_TARGET=<AMD GPU ISA version string> \
    -DROCM_DEVICE_LIB_DIR=<location of the ROCm-Device-Libs bitcode> \
    -DUSE_CODEXL_ACTIVITY_LOGGER=1 \
    <ToT HCC checkout directory>

In your application compiled using hcc, include the CodeXL Activiy Logger header: ::
 
  #include <CXLActivityLogger.h>

For information about the usage of the Activity Logger for profiling, please refer to its documentation.



HC Best Practices
=================

HC comes with two header files as of now:

    * <`hc.hpp <http://scchan.github.io/hcc/hc_8hpp.html>`> : Main header file for HC
    * <`hc_math.hpp <http://scchan.github.io/hcc/hc__math_8hpp_source.html>`> : Math functions for HC

Most HC APIs are stored under "hc" namespace, and the class name is the same as their counterpart in C++AMP "Concurrency" namespace. Users of C++AMP should find it easy to switch from C++AMP to HC.

=============================== =======================
C++AMP 		       			HC 
=============================== =======================
Concurrency::accelerator	 hc::accelerator
Concurrency::accelerator_view	 hc::accelerator_view
Concurrency::extent		 hc::extent
Concurrency::index		 hc::index
Concurrency::completion_future   hc::completion_future
Concurrency::array		 hc::array
Concurrency::array_view     	 hc::array_view

=============================== ======================= 

How to build programs with HC API
**********************************
Use "hcc-config", instead of "clamp-config", or you could manually add "-hc" when you invoke clang++. Also, "hcc" is added as an alias for "clang++".

For example: ::

 `` hcchcc-config –cxxflags –ldflagsfoo.cpp -o foo `` 

HCC built-in macros
*******************
Built-in macros:

==================== =======================================================================
Macro                            Meaning 
==================== =======================================================================
__HCC__		      always be 1 
__hcc_major__	      major version number of HCC 
__hcc_minor__	      minor version number of HCC 
__hcc_patchlevel__    patchlevel of HCC 
__hcc_version__	      combined string of __hcc_major__, __hcc_minor__, __hcc_patchlevel__
==================== =======================================================================

The rule for __hcc_patchlevel__ is: yyWW-(HCC driver git commit #)-(HCC clang git commit #)

   * yy stands for the last 2 digits of the year
   * WW stands for the week number of the year

Macros for language modes in use:

=============== =========================================
 Macro           Meaning 
=============== =========================================
__KALMAR_AMP__    1 in case in C++ AMP mode (-std=c++amp) 
__KALMAR_HC__     1 in case in HC mode (-hc) 
=============== =========================================

Compilation mode: HCC is a single-source compiler where kernel codes and host codes can reside in the same file. Internally HCC would trigger 2 compilation iterations, and the following macros can be user by user programs to determine which mode the compiler is in.

====================== =================================================================
Macro           		Meaning 
====================== =================================================================
__KALMAR_ACCELERATOR_   not 0 in case the compiler runs in kernel code compilation mode
__KALMAR_CPU__          not 0 in case the compiler runs in host code compilation mode
====================== =================================================================

HC-specific features
********************

   * relaxed rules in operations allowed in kernels
   * new syntax of tiled_extent and tiled_index
   * dynamic group segment memory allocation
   * true asynchronous kernel launching behavior
   * additional HSA-specific APIs

Differences between HC API and C++ AMP
***************************************
Despite HC and C++ AMP share a lot of similarities in terms of programming constructs (e.g. parallel_for_each, array, array_view, etc.), there are several significant differences between the two APIs.

**Support for explicit asynchronous parallel_for_each**
In C++ AMP, the parallel_for_each appears as a synchronous function call in a program (i.e. the host waits for the kernel to complete); howevever, the compiler may optimize it to execute the kernel asynchronously and the host would synchronize with the device on the first access of the data modified by the kernel. For example, if a parallel_for_each writes the an array_view, then the first access to this array_view on the host after the parallel_for_each would block until the parallel_for_each completes.

HC supports the automatic synchronization behavior as in C++ AMP. In addition, HC's parallel_for_each supports explicit asynchronous execution. It returns a completion_future (similar to C++ std::future) object that other asynchronous operations could synchronize with, which provides better flexibility on task graph construction and enables more precise control on optimization.


**Annotation of device functions**

C++ AMP uses the restrict(amp) keyword to annotatate functions that runs on the device. ::

 void foo() restrict(amp) { .. } ... parallel_for_each(...,[=] () restrict(amp) { foo(); });

HC uses a function attribute ([[hc]] or __attribute__((hc)) ) to annotate a device function. ::

  void foo() [[hc]] { .. } ... parallel_for_each(...,[=] () [[hc]] { foo(); }); 

The [[hc]] annotation for the kernel function called by parallel_for_each is optional as it is automatically annotated as a device function by the hcc compiler. The compiler also supports partial automatic [[hc]] annotation for functions that are called by other device functions within the same source file:

Since bar is called by foo, which is a device function, the hcc compiler  will automatically annotate bar as a device function void bar() { ... } void foo() [[hc]] { bar(); }


**Dynamic tile size**

C++ AMP doesn't support dynamic tile size. The size of each tile dimensions has to be a compile-time constant specified as template arguments to the tile_extent object: 

 `extent<2> <http://scchan.github.io/hcc/classConcurrency_1_1extent.html>`_  ex(x, y) 

 create a tile extent of 8x8 from the extent object  note that the tile dimensions have to be constant values tiled_extent<8,8> t_ex(ex);

parallel_for_each(t_ex, [=](tiled_index<8,8> t_id) restrict(amp) { ... }); HC supports both static and dynamic tile size: 
`extent<2> <http://scchan.github.io/hcc/classConcurrency_1_1extent.html>`_ ex(x,y)

create a tile extent from dynamically calculated values  note that the the tiled_extent template takes the rank instead of dimensions tx = test_x ? tx_a : tx_b; ty = test_y ? ty_a : ty_b; tiled_extent<2> t_ex(ex, tx, ty);
parallel_for_each(t_ex, [=](tiled_index<2> t_id) [[hc]] { ... });

**Support for memory pointer**

C++ AMP doens't support lambda capture of memory pointer into a GPU kernel.

HC supports capturing memory pointer by a GPU kernel.

allocate GPU memory through the HSA API int* gpu_pointer; hsa_memory_allocate(..., &gpu_pointer); ... parallel_for_each(ext, [=](index i) [[hc]] { gpu_pointer[i[0]]++; }

For HSA APUs that supports system wide shared virtual memory, a GPU kernel can directly access system memory allocated by the host: int* cpu_memory = (int*) malloc(...); ... parallel_for_each(ext, [=](index i) [[hc]] { cpu_memory[i[0]]++; }); 


API documentation
*******************
`API reference of HCC <https://scchan.github.io/hcc/>`_

HIP Programing Guide
====================

HIP provides a C++ syntax that is suitable for compiling most code that commonly appears in compute kernels, including classes, namespaces, operator overloading, templates and more. Additionally, it defines other language features designed specifically to target accelerators, such as the following:

    A kernel-launch syntax that uses standard C++, resembles a function call and is portable to all HIP targets
    Short-vector headers that can serve on a host or a device
    Math functions resembling those in the "math.h" header included with standard C++ compilers
    Built-in functions for accessing specific GPU hardware capabilities

This section describes the built-in variables and functions accessible from the HIP kernel. It’s intended for readers who are familiar with Cuda kernel syntax and want to understand how HIP is different.

  * :ref:`HIP-GUIDE` 


HIP Best Practices
==================

 * :ref:`HIP-porting-guide` 
 * :ref:`HIP-terminology` 
 * :ref:`HIP-run-time-API` 
 * :ref:`HIP-FAQ` 
   



OpenCL Programing Guide
========================
* :ref:`Opencl-Programming-Guide`


OpenCL Best Practices
======================
* :ref:`Opencl-optimization`




