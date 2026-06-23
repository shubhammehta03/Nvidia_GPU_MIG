# NVIDIA Multi-Instance GPU (MIG) steps:
This repository provides step-by-step guides for enabling and partitioning NVIDIA GPUs into **Multi-Instance GPU (MIG)** instances.

## Supported Hardware & Prerequisites

* **Supported GPUs:** NVIDIA A100, H100, H200 (SXM and PCIe variants)
* **OS / Environment:** Enterprise Linux (Rocky Linux / AlmaLinux / RHEL 8 or 9), Ubuntu Server
* **Drivers:** NVIDIA Datacenter Drivers (`>= 525.xx` recommended)
* **Access:** `sudo` / root privileges on target compute nodes

**Important Note:** Enabling or disabling MIG mode requires resetting the GPU or reloading the NVIDIA kernel modules, which will terminate any running GPU workloads.
By default, MIG mode is not enabled on the GPU. For example, running nvidia-smi shows that MIG mode is disabled:

## Getting Started
### 1. Enable MIG Mode
First Verify, your GPU index and current MIG status:
nvidia-smi --query-gpu=index,name,mig.mode.current --format=csv
The GPUs can be selected using comma separated GPU indexes, PCI Bus IDs or UUIDs. If no GPU ID is specified, then MIG mode is applied to all the GPUs on the system. When MIG is enabled on the GPU, depending on the GPU product, the driver will attempt to reset the GPU so that MIG mode can take effect. 

Before creating GPU slices, MIG mode can be enabled on a per-GPU basis with the following command:

```# nvidia-smi -i <index no> -mig 1    #index no can be 0,1 depending upon above command ```

```# nvidia-smi -mig 1                  # Enable MIG mode on all GPUs ```

To reset GPU:

``` #nvidia-smi --gpu-reset ```

### 2. List GPU Instance Profiles:
The NVIDIA driver provides a number of profiles that users can opt-in for when configuring the MIG feature in A100. The profiles are the sizes and capabilities of the GPU instances that can be created by the user. The driver also provides information about the placements, which indicate the type and number of instances that can be created.

``` # nvidia-smi mig -lgip -i 0 ```
It will list GPU instance profiles available for creating Multi-Instance GPU (MIG) instances on an NVIDIA GPU

Name: Describes the profile (e.g., 1g.10gb means 1 GPU slice with 10 GB memory).

List the possible placements available using the following command. The syntax of the placement is {<index>}:<GPU Slice Count> and shows the placement of the instances on the GPU. The placement index shown indicates how the profiles are mapped on the GPU as shown in the figure:

``` # nvidia-smi mig -lgipp [-i 0] ```

The command shows that the user can create two instances of type 3g.40gb (profile ID 9) or seven instances of 1g.10gb (profile ID 19).

### 3. Creating GPU Instances:
Before starting to use MIG, the user needs to create GPU instances using the -cgi option. One of three options can be used to specify the instance profiles to be created:
1. Profile ID (e.g. 9, 14, 5)
2. Short name of the profile (such as 3g.40gb)
3. Full profile name of the instance (such as MIG 3g.40gb)

Once the GPU instances are created, you need to create the corresponding Compute Instances (CI). By using the -C option, nvidia-smi creates these instances.
Without creating GPU instances (and corresponding compute instances), CUDA workloads cannot be run on the GPU. In other words, simply enabling MIG mode on the GPU is not sufficient. Also note that, the created MIG devices are not persistent across system reboots. Thus, the user or system administrator needs to recreate the desired MIG configurations if the GPU or system is reset. For automated tooling support for this purpose, refer to the NVIDIA MIG Partition Editor (or mig-parted) tool, including creating a systemd service that could recreate the MIG geometry at system startup.

The following example shows how the user can create GPU instances (and corresponding compute instances). In this example, the user can create two GPU instances (of type 1g.10gb), with each GPU instance having half of the available compute and memory capacity.

``` # nvidia-smi mig -cgi 19,1g.10gb -i 0 -C ```

- Create GPU Instance to create a GPU instance with a specified profile.
- 19,1g.10gb: Specifies the GPU instance profile.
-i 0: Specifies the GPU index. In this case, it’s creating the instance on GPU 0.
-C: Enables compute mode for the instance, which means it will be configured to run 	compute workloads (e.g., CUDA, AI, HPC).

### 4. Now list the available GPU instances:

``` #  nvidia-smi mig -lgi ```

This will only show GPU instances

Now verify that the GIs and corresponding CIs are created:

``` # nvidia-smi ```

To create seven instances with equal memory and available compute:
``` # nvidia-smi mig -cgi 19,19,19,19,19,19,19 -i 0 -C ```

### 5. Integrating MIG with Slurm GRES

Gres.conf:
In gpu:
cat /etc/slurm/gres.conf
```# GPU0 1g.10gb instances
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap93
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap102
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap111
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap120
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap66
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap75
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap84

# GPU1 1g.10gb instances
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap237
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap246
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap255
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap264
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap201
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap210
Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap219
```

Rdgpu02:
cat /etc/slurm/gres.conf
```# GPU 0 MIG instances
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap66
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap75
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap84
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap102
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap111
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap120
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap129
# GPU 1 MIG instances
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap201
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap210
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap219
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap228
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap237
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap246
NodeName=rdgpu02 Name=gpu Type=1g.10gb File=/dev/nvidia-caps/nvidia-cap255
```

### In slurm.conf:
NodeName=rdgpu01 CPUs=48 Boards=1 SocketsPerBoard=2 CoresPerSocket=24 ThreadsPerCore=1 RealMemory=192032  Feature=gpu Gres=gpu:1g.10gb:14
NodeName=rdgpu02 CPUs=48 Boards=1 SocketsPerBoard=2 CoresPerSocket=24 ThreadsPerCore=1 RealMemory=192032  Feature=gpu Gres=gpu:1g.10gb:14

Note: TaskPlugin=task/affinity,task/cgroup and cgroup should be enabled in gpu 

### GPU Utilization Metrics
NVML (and nvidia-smi) does not support attribution of utilization metrics to MIG devices. From the previous example, the utilization is displayed as N/A when running CUDA programs:
For monitoring MIG devices on MIG capable GPUs such as the A100, including attribution of GPU metrics (including utilization and other profiling metrics), it is recommended to use NVIDIA DCGM v2.0.13 or later. See the Profiling Metrics section in the DCGM User Guide for more details on getting started.

### Destroying GPU Instances
Once the GPU is in MIG mode, GIs and CIs can be configured dynamically. The following example shows how the CIs and GIs created in the previous examples can be destroyed.

Note: If the intention is to destroy all the CIs and GIs, then this can be accomplished with the following commands:

``` # nvidia-smi mig -dci && sudo nvidia-smi mig -dgi ```

To delete the specific CIs created under GI 1.

``` # nvidia-smi mig -dci -ci 0,1,2 -gi 1 ```

Now the GIs have to be deleted:

``` # nvidia-smi mig -dgi ```


To see all usage of GPU:

``` # nvidia-smi -q [-i 0] ```

Check nvml is there:

``` # rpm -qlp slurm-slurmd-23.11.5-1.el8.x86_64 | grep nvml ```


In gres.conf:
```
#RDGPU01: 
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap66,/dev/nvidia-caps/nvidia-cap67 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap75,/dev/nvidia-caps/nvidia-cap76 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap84,/dev/nvidia-caps/nvidia-cap85 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap93,/dev/nvidia-caps/nvidia-cap94 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap102,/dev/nvidia-caps/nvidia-cap103 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap111,/dev/nvidia-caps/nvidia-cap112 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia0,/dev/nvidia-caps/nvidia-cap120,/dev/nvidia-caps/nvidia-cap12 AutoDetect=nvml
RDGPU01: GPU02
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap201,/dev/nvidia-caps/nvidia-cap202 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap210,/dev/nvidia-caps/nvidia-cap211 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap219,/dev/nvidia-caps/nvidia-cap220 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap237,/dev/nvidia-caps/nvidia-cap238 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap246,/dev/nvidia-caps/nvidia-cap247 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap255,/dev/nvidia-caps/nvidia-cap256 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia1,/dev/nvidia-caps/nvidia-cap264,/dev/nvidia-caps/nvidia-cap265 AutoDetect=nvml
```

#RDGPU02: 
```
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap66,/dev/nvidia-caps/nvidia-cap67 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap75,/dev/nvidia-caps/nvidia-cap76 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap84,/dev/nvidia-caps/nvidia-cap85 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap102,/dev/nvidia-caps/nvidia-cap103 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap111,/dev/nvidia-caps/nvidia-cap112 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap120,/dev/nvidia-caps/nvidia-cap121 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap129,/dev/nvidia-caps/nvidia-cap130 AutoDetect=nvml
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap201,/dev/nvidia-caps/nvidia-cap202
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap210,/dev/nvidia-caps/nvidia-cap211
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap219,/dev/nvidia-caps/nvidia-cap220
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap228,/dev/nvidia-caps/nvidia-cap229
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap237,/dev/nvidia-caps/nvidia-cap238
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap246,/dev/nvidia-caps/nvidia-cap247
Name=gpu Type=nvidia_a100_80gb_pcie_1g.10gb MultipleFiles=/dev/nvidia-caps/nvidia-cap255,/dev/nvidia-caps/nvidia-cap256
```

** Shortform meaning:
-mig    : --multi-instance-gpu= Enable or disable Multi Instance GPU: 0/DISABLED, 1/ENABLED.                               			Requires root.
-cgi    : create gpu instances
-C 	    : Enables compute mode for the instance, which means it will be configured to run 		compute workloads (e.g., CUDA, AI, HPC).
-i,--id :                 Target a specific GPU or Unit.
-u,--unit                Show unit, rather than GPU, attributes.
-f,--filename           Log to a specified file, rather than to stdout.
-x,--xml-format          Produce XML output.
--dtd                 When showing xml output, embed DTD.
-L,--list-gpus           Display a list of GPUs connected to the system.
-d,--display            Display only selected information: MEMORY, UTILIZATION, ECC, TEMPERATURE, POWER, CLOCK, COMPUTE, PIDS, PERFORMANCE, SUPPORTED_CLOCKS,
                                    PAGE_RETIREMENT, ACCOUNTING, ENCODER_STATS, SUPPORTED_GPU_TARGET_TEMP, VOLTAGE, FBC_STATS
                                    ROW_REMAPPER, RESET_STATUS, 
                        Flags can be combined with comma e.g. ECC,POWER.
                                Sampling data with max/min/avg is also returned
                                for POWER, UTILIZATION and CLOCK display types.

-r,--gpu-reset	: Trigger reset of the GPU.Can be used to reset the GPU HW state in situations that 		would otherwise require a machine reboot.Typically useful if a double bit ECC error 		has occurred.
--query-gpu     :           Information about GPU.
                                Call --help-query-gpu for more info.
--query-supported-clocks    List of supported clocks.
                                Call --help-query-supported-clocks for more info.
--query-compute-apps        List of currently active compute processes.
                                Call --help-query-compute-apps for more info.
--query-accounted-apps      List of accounted compute processes.
                                Call --help-query-accounted-apps for more info.
topo     :                   Displays device/system topology. "nvidia-smi topo -h" for more information.
pmon     :                  Displays process stats in scrolling format.
Device Monitoring:
    dmon                        Displays device stats in scrolling format.
                                "nvidia-smi dmon -h" for more information.

    daemon                      Runs in background and monitor devices as a daemon process.
                                This is an experimental feature. Not supported on Windows baremetal
                                "nvidia-smi daemon -h" for more information.

    replay                      Used to replay/extract the persistent stats generated by daemon.
                                This is an experimental feature.
                                "nvidia-smi replay -h" for more information.


### Following Information is for Nvidia A100 GPU for knowledge purpose only:
The NVIDIA A100 GPU has 108 streaming multiprocessors (SMs), which are often referred to as compute cores. These SMs are the fundamental units responsible for executing parallel workloads on the GPU. Here's a detailed breakdown of the compute architecture:
________________________________________
1. Core Components of the NVIDIA A100

Streaming Multiprocessors (SMs):

The A100 GPU has 108 SMs.
Each SM is a self-contained compute unit, including CUDA cores, Tensor cores, registers, and shared memory.

CUDA Cores:

Each SM in the A100 contains 64 CUDA cores for general-purpose floating-point and integer operations.
Total CUDA cores: 108 SMs×64 CUDA cores/SM=6,912 CUDA cores.\text{108 SMs} \times \text{64 CUDA cores/SM} = \text{6,912 CUDA cores}. 

Tensor Cores:

Each SM in the A100 contains 4 third-generation Tensor Cores, optimized for matrix operations in AI/ML tasks.
Total Tensor cores: 108 SMs×4 Tensor Cores/SM=432 Tensor cores.\text{108 SMs} \times \text{4 Tensor Cores/SM} = \text{432 Tensor cores}. 
________________________________________
2. Compute Performance
The A100 is designed for high-performance computing (HPC), AI/ML, and data analytics workloads, and its compute capabilities include: 
FP64 (Double Precision): For scientific calculations.
FP32 (Single Precision): For general-purpose compute tasks.
TF32 and Mixed Precision: Optimized for AI/ML training and inference tasks.
INT8 and INT4: For inference acceleration.
________________________________________
3. Compute Cores in MIG (Multi-Instance GPU) Mode
When the A100 is running in MIG (Multi-Instance GPU) mode, its 108 SMs can be divided into up to 7 GPU instances, each with an independent portion of the compute cores and memory.
For example:
1g.10gb profile: Allocates 1/7th of the compute power and memory, so: 
SMs allocated per instance: 108 SMs÷7≈14SMs.\text{108 SMs} \div 7 \approx 14 SMs. 
Larger profiles like 2g.20gb or 4g.40gb allocate more SMs.
________________________________________

