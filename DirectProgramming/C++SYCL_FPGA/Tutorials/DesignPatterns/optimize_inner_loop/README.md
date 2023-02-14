# Optimizing Inner Loop Throughput
This FPGA tutorial discusses optimizing the throughput of an inner loop with a low trip count.

| Optimized for                     | Description
---                                 |---
| OS                                | Linux* Ubuntu* 18.04/20.04 <br> RHEL*/CentOS* 8 <br> SUSE* 15 <br> Windows* 10
| Hardware                          | Intel® Agilex™, Arria® 10, and Stratix® 10 FPGAs
| Software                          | Intel® oneAPI DPC++/C++ Compiler
| What you will learn               | How to optimize the throughput of an inner loop with a low trip.
| Time to complete                  | 45 minutes

> **Note**: Even though the Intel DPC++/C++ OneAPI compiler is enough to compile for emulation, generating reports and generating RTL, there are extra software requirements for the simulation flow and FPGA compiles.
>
> For using the simulator flow, Intel® Quartus® Prime Pro Edition and one of the following simulators must be installed and accessible through your PATH:
> - Questa*-Intel® FPGA Edition
> - Questa*-Intel® FPGA Starter Edition
> - ModelSim® SE
>
> When using the hardware compile flow, Intel® Quartus® Prime Pro Edition must be installed and accessible through your PATH.
>
> :warning: Make sure you add the device files associated with the FPGA that you are targeting to your Intel® Quartus® Prime installation.

## Prerequisites

This sample is part of the FPGA code samples.
It is categorized as a Tier 3 sample that demonstrates a design pattern.

```mermaid
flowchart LR
   tier1("Tier 1: Get Started")
   tier2("Tier 2: Explore the Fundamentals")
   tier3("Tier 3: Explore the Advanced Techniques")
   tier4("Tier 4: Explore the Reference Designs")
   
   tier1 --> tier2 --> tier3 --> tier4
   
   style tier1 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
   style tier2 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
   style tier3 fill:#f96,stroke:#333,stroke-width:1px,color:#fff
   style tier4 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
```

Find more information about how to navigate this part of the code samples in the [FPGA top-level README.md](/DirectProgramming/DPC++FPGA/README.md).
You can also find more information about [troubleshooting build errors](/DirectProgramming/DPC++FPGA/README.md#troubleshooting), [running the sample on the Intel® DevCloud](/DirectProgramming/DPC++FPGA/README.md#build-and-run-the-samples-on-intel-devcloud-optional), [using Visual Studio Code with the code samples](/DirectProgramming/DPC++FPGA/README.md#use-visual-studio-code-vs-code-optional), [links to selected documentation](/DirectProgramming/DPC++FPGA/README.md#documentation), etc.

## Purpose
This tutorial will show how to optimize the throughput of an inner loop with a low trip count. A *low* trip count is relative. In this tutorial, we will consider *low* to be on the order of 100 or fewer iterations.

### Suggested Prerequisites
This is an advanced tutorial that relies on understanding f<sub>MAX</sub>/II and *speculated iterations*. We suggest first completing the **Speculated Iterations** (speculated_iterations) tutorial.

### Low Trip Count Inner Loops
Consider the following snippet of pseudocode:
```c++
for (int i; i < kOuterLoopBound; i++) {
  int inner_loop_iterations = rand() % kInnerLoopBound;
  for (int j; j < inner_loop_iterations; j++) {
    /* ... */
  }
}
```

In this tutorial, we will focus on optimizing inner loops with low trip counts, so let's assume that the code snippet above `kOuterLoopBound` is some large number (>1 million) and that `kInnerLoopBound` is 3. This means that the value of `inner_loop_iterations` is *dynamic*, but we know it is in the range `[0,3)`. Furthermore, let's assume that the II of the inner loop is 1, which means that a new inner loop iteration can start every cycle. This means that the outer loop II is *dynamic* and depends on how many inner loop iterations need to be started by the previous outer loop iteration. A possible timing diagram for this loop structure is shown in the figure below, where the numbers in the squares are the values of `i` and `j`, respectively.

![](timing_base.png)

In general, the compiler optimizes loops for throughput with the assumption that the loop has a high trip count. These optimizations include (but are not limited to) speculating iterations and inserting pipeline registers in the circuit that starts loops. The next two subsections will describe how these optimizations can substantially *decrease* throughput and how you can disable them to improve your design when applied to inner loops with low trip counts.

#### Speculated Iterations
Loop speculation enables loop iterations to be initiated before determining whether they should have been initiated. *Speculated iterations* are the iterations of a loop that launch before the exit condition computation has been completed. This is beneficial when the computation of the exit condition is preventing effective loop pipelining. However, when an inner loop has a low trip count, speculating iterations results in a relatively high proportion of invalid loop iterations.

For example, suppose we speculated 2 inner loop iterations for the earlier code snippet. In that case, our timing diagram may look like the figure below, where the red blocks with the `S` denote an invalid speculated iteration.

![](timing_2_speculated.png)

In our case, where the inner loop iteration count is in the range `[0,3)`, speculating 2 iterations can cause up to a 3x **reduction** in the design's throughput. This happens when each outer loop iteration launches 1 inner loop iteration (i.e., `inner_loop_iterations` is always 1), but 2 iterations are speculated. For this reason, **it is advised to force the compiler to not speculate iterations for inner loops with known small trip counts using the `[[intelfpga::speculated_iterations(0)]]` attribute**.

For more information on speculated iterations, see the **Speculated Iterations** (speculated_iterations) tutorial.

#### Dynamic Trip Counts
As mentioned earlier, the compiler's default behavior is to optimize loops for throughput. However, as we saw in the previous section, loops with low trip counts have unique throughput characteristics that lead to the compiler making different optimizations. The compiler will try its best to determine if a loop has a high or low trip count and optimizes accordingly. However, in some circumstances, we may need to provide it with more information to make a better decision.

In the previous section, this additional information was the `speculated_iterations` attribute. However, it's not just speculated iterations that cause delays in the launching of inner loops. The compiler has other heuristics at play. For example, the compiler may attempt to improve the f<sub>MAX</sub> of a loop circuit by adding a pipeline register on the circuit path that starts a loop, which results in a 1 cycle delay in starting the loop. For outer loops with large trip counts, this 1 cycle delay is negligible. However, for inner loops with small trip counts, this 1 cycle delay can cause throughput degradation. Like the speculated iteration case discussed in the previous section, this 1 cycle delay can result in up to a 2x **reduction** in the design's throughput.

If the inner loop bounds are known to the compiler, it will decide whether to turn on/off this delay register depending on the (known) trip count. However, in the earlier pseudocode snippet, the inner loop's trip count is not a constant (`inner_loop_iterations` is a random number at runtime). **In cases like this, we suggest explicitly bounding the trip count of the inner loop**. This is illustrated in the pseudocode snippet below, where we have added the `j < kInnerLoopBound` exit condition to the inner loop. This gives the compiler more explicit information about the loop's trip count and allows it to optimize accordingly.

```c++
for (int i; i < kOuterLoopBound; i++) {
  int inner_loop_iterations = rand() % kInnerLoopBound;
  for (int j; j < inner_loop_iterations && j < kInnerLoopBound; j++) {
    /* ... */
  }
}
```

### Code Sample Details
The sample code finds the sum of an array, albeit in a roundabout way, to better illustrate the optimizations. The `Producer` kernel performs the logic in the pseudocode below. We fill the `input_array` array with random values in the range `[0,3]`. As a result, the number of inner loop iterations will be in the range `[0,3]` for all outer loop iterations.

```c++
for (int i = 0; i < input_array.size(); i++) {
  // write a true to the pipe 'inner_loop_iterations' times
  // this is the inner loop with the low trip count
  int inner_loop_iterations = input_array[i];
  for (int j = 0; j < inner_loop_iterations; j++) {
    Pipe::write(true);
  }
}

// tells the consumer that the data is done
Pipe::write(false);
```

The `Consumer` kernel reads from the pipe, tracks the number of valid reads, and returns it as output data, as shown in the pseudocode below. The result is the sum of the values in `input_array`. Again, this is a roundabout way to sum the values in an array, but it is a simple way to showcase the inner loop optimizations discussed in this tutorial.

```c++
int result = 0;
while (Pipe::read()) {
  result++;
}
```

## Key Concepts
* Optimizing the throughput of inner loops with low trip counts by using the `speculated_iterations` attribute and explicit loop bounding

## Building the `optimize_inner_loop` Tutorial

> **Note**: When working with the command-line interface (CLI), you should configure the oneAPI toolkits using environment variables. 
> Set up your CLI environment by sourcing the `setvars` script located in the root of your oneAPI installation every time you open a new terminal window. 
> This practice ensures that your compiler, libraries, and tools are ready for development.
>
> Linux*:
> - For system wide installations: `. /opt/intel/oneapi/setvars.sh`
> - For private installations: ` . ~/intel/oneapi/setvars.sh`
> - For non-POSIX shells, like csh, use the following command: `bash -c 'source <install-dir>/setvars.sh ; exec csh'`
>
> Windows*:
> - `C:\Program Files(x86)\Intel\oneAPI\setvars.bat`
> - Windows PowerShell*, use the following command: `cmd.exe "/K" '"C:\Program Files (x86)\Intel\oneAPI\setvars.bat" && powershell'`
>
> For more information on configuring environment variables, see [Use the setvars Script with Linux* or macOS*](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-linux-or-macos.html) or [Use the setvars Script with Windows*](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-windows.html).

### On a Linux* System

1. Generate the `Makefile` by running `cmake`.
  ```
  mkdir build
  cd build
  ```
  To compile for the default target (the Agilex™ device family), run `cmake` using the command:
  ```
  cmake ..
  ```

  > **Note**: You can change the default target by using the command:
  >  ```
  >  cmake .. -DFPGA_DEVICE=<FPGA device family or FPGA part number>
  >  ``` 
  >
  > Alternatively, you can target an explicit FPGA board variant and BSP by using the following command: 
  >  ```
  >  cmake .. -DFPGA_DEVICE=<board-support-package>:<board-variant>
  >  ``` 
  >
  > You will only be able to run an executable on the FPGA if you specified a BSP.

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
     ```
     make fpga_emu
     ```
   * Generate the optimization report:
     ```
     make report
     ```
   * Compile for simulation (fast compile time, targets simulated FPGA device, reduced data size):
     ```
     make fpga_sim
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     make fpga
     ```

### On a Windows* System

1. Generate the `Makefile` by running `cmake`.
  ```
  mkdir build
  cd build
  ```
  To compile for the default target (the Agilex™ device family), run `cmake` using the command:
  ```
  cmake -G "NMake Makefiles" ..
  ```
  > **Note**: You can change the default target by using the command:
  >  ```
  >  cmake -G "NMake Makefiles" .. -DFPGA_DEVICE=<FPGA device family or FPGA part number>
  >  ``` 
  >
  > Alternatively, you can target an explicit FPGA board variant and BSP by using the following command: 
  >  ```
  >  cmake -G "NMake Makefiles" .. -DFPGA_DEVICE=<board-support-package>:<board-variant>
  >  ``` 
  >
  > You will only be able to run an executable on the FPGA if you specified a BSP.

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
     ```
     nmake fpga_emu
     ```
   * Generate the optimization report:
     ```
     nmake report
     ```
   * Compile for simulation (fast compile time, targets simulated FPGA device, reduced data size):
     ```
     nmake fpga_sim
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     nmake fpga
     ```

*Note:* If you encounter any issues with long paths when compiling under Windows*, you may have to create your ‘build’ directory in a shorter path, for example c:\samples\build.  You can then run cmake from that directory, and provide cmake with the full path to your sample directory.

## Examining the Reports
Locate `report.html` in the `optimize_inner_loop.prj/reports/` directory. Open the report in any of Chrome*, Firefox*, Edge*, or Internet Explorer*.

Open the reports and look at the *Loop Analysis* pane. Examine the loop attributes for the three different versions of the `Producer` kernel (`Producer<0>`, `Producer<1>` and `Producer<2>`). Note that each has an outer loop with an II of 1 and an inner loop with an II of 1. As discussed earlier in this tutorial, the II of the outer loop will be *dynamic* and depend on the inner loop's execution for each outer loop iteration. Also, note the *Speculated Iterations* column, which should show 2 speculated loop iterations on the inner loop for `Producer<0>` and 0 for `Producer<1>` and `Producer<2>`. At this time, there is no information in the reports indicating whether there will be a 1 cycle delay in starting the loop. We are working on improving our reports to help you better debug throughput bottlenecks!

### Version 0
Version 0 of the kernel (`Producer<0>`) does **not** bound the inner loop trip count and speculates 2 iterations. Since we expect 1 inner loop iteration for every outer loop iteration. This results in 3 invalid iterations for every 1 valid inner loop iteration; 2 (invalid) speculated iterations are launched, and there is a 1 cycle delay starting the inner loop. Therefore, this version only achieves ~1/4 the maximum throughput.

### Version 1
Version 1 of the kernel (`Producer<1>`) does **not** bound the inner loop trip count but explicitly turns off speculation for the inner loop (using the `[[intelfpga::speculated_iterations(0)]]` attribute). Compared to version 0, we have removed 2 of the 3 invalid iterations. However, since we did not bound the inner loop's trip count, the compiler will still insert a pipeline register in the path that starts it. This results in a 1 cycle delay starting the inner loop and up to a 50% drop in the design's throughput.

### Version 2
Version 2 of the kernel (`Producer<2>`) explicitly bounds the inner loop trip count and turns off loop speculation for the inner loop. This version maximizes throughput by removing the delay in launching inner loop iterations for consecutive outer loop iterations, as shown later in the [Example of Output](#example-of-output) section.

## Running the Sample

 1. Run the sample on the FPGA emulator (the kernel executes on the CPU):
     ```
     ./optimize_inner_loop.fpga_emu    (Linux)
     optimize_inner_loop.fpga_emu.exe  (Windows)
     ```
2. Run the sample on the FPGA simulator device:
  * On Linux
    ```
    CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=1 ./loop_carried_dependency.fpga_sim
    ```
  * On Windows
    ```   
    set CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=1
    loop_carried_dependency.fpga_sim.exe
    set CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=
    ```
3. Run the sample on the FPGA device (only if you ran `cmake` with `-DFPGA_DEVICE=<board-support-package>:<board-variant>`):
     ```
     ./optimize_inner_loop.fpga        (Linux)
     optimize_inner_loop.fpga.exe      (Windows)
     ```

### Example of Output
You should see the following output in the console:

1. When running on the FPGA emulator or simulator
    ```
    generating 5000 random numbers in the range [0,3)
    Running kernel 0
    Running kernel 1
    Running kernel 2
    PASSED
    ```

2. When running on the FPGA device
    ```
    generating 5000000 random numbers in the range [0,3]
    Running kernel 0
    Running kernel 1
    Running kernel 2
    Kernel 0 throughput: 192.19 MB/s
    Kernel 1 throughput: 359.47 MB/s
    Kernel 2 throughput: 636.29 MB/s
    PASSED
    ```
    NOTE: These throughput numbers were collected using the Intel® PAC with Intel Arria® 10 GX FPGA.

## License

Code samples are licensed under the MIT license. See
[License.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/License.txt) for details.

Third party program Licenses can be found here: [third-party-programs.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/third-party-programs.txt).