# ACPI, NUMA, SRAT, BDF, VFIO, and GPU MIG Tutorial

This tutorial covers the hardware and firmware concepts related to the Generic Initiator implementation in Cloud Hypervisor.

## Table of Contents

1. [ACPI (Advanced Configuration and Power Interface)](#1-acpi-advanced-configuration-and-power-interface)
2. [NUMA (Non-Uniform Memory Access)](#2-numa-non-uniform-memory-access)
3. [SRAT (System Resource Affinity Table)](#3-srat-system-resource-affinity-table)
4. [PCI BDF (Bus:Device.Function)](#4-pci-bdf-busdevicefunction)
5. [VFIO (Virtual Function I/O)](#5-vfio-virtual-function-io)
6. [GPU MIG (Multi-Instance GPU)](#6-gpu-mig-multi-instance-gpu)
7. [How They Work Together](#7-how-they-work-together)
8. [Generic Initiator Use Case](#8-generic-initiator-use-case)

---

## 1. ACPI (Advanced Configuration and Power Interface)

### What is ACPI?

ACPI is an industry-standard interface between the operating system and the platform firmware (BIOS/UEFI). It provides a standardized way for the OS to discover and configure hardware.

### Key ACPI Components

**ACPI Tables**: Data structures provided by firmware that describe hardware configuration:
- **DSDT (Differentiated System Description Table)**: Defines devices and methods
- **MADT (Multiple APIC Description Table)**: Describes interrupt controllers and CPU topology
- **SRAT (System Resource Affinity Table)**: Describes NUMA topology
- **SLIT (System Locality Information Table)**: Describes distances between NUMA nodes
- **MCFG (Memory Mapped Configuration)**: PCI Express configuration space

**AML (ACPI Machine Language)**: Bytecode language used in ACPI tables to describe hardware operations.

### ACPI in Virtualization

Virtual machine monitors (VMMs) like Cloud Hypervisor generate synthetic ACPI tables to present virtual hardware to guest operating systems. The guest OS reads these tables during boot to understand the system topology.

**Example**: When you create a VM with 4 CPUs and 2 NUMA nodes, Cloud Hypervisor generates ACPI tables (MADT, SRAT) that make the guest OS believe it's running on physical hardware with that configuration.

### ACPI Table Structure

All ACPI tables follow a common header format:

```
Offset  Size  Field
------  ----  -----
0       4     Signature (e.g., "SRAT")
4       4     Length (total table size)
8       1     Revision
9       1     Checksum
10      6     OEM ID
16      8     OEM Table ID
24      4     OEM Revision
28      4     Creator ID
32      4     Creator Revision
36      ...   Table-specific data
```

---

## 2. NUMA (Non-Uniform Memory Access)

### What is NUMA?

NUMA is a computer memory design where memory access time depends on the memory location relative to the processor. Each CPU (or group of CPUs) has "local" memory that it can access quickly, and "remote" memory on other NUMA nodes that it can access more slowly.

### Why NUMA Exists

**Traditional SMP (Symmetric Multi-Processing)**:
- All CPUs share a single memory controller
- Becomes a bottleneck as CPU count increases
- All memory accesses have uniform latency

**NUMA Architecture**:
- Each CPU socket has its own memory controller
- CPUs can access local memory quickly (low latency)
- CPUs can access remote memory via interconnect (higher latency)
- Scales better to many CPUs

### NUMA Topology Example

```
+----------+         +----------+
| CPU 0-7  |         | CPU 8-15 |
| Memory A |<------->| Memory B |
| Node 0   |  Interconnect  | Node 1   |
+----------+         +----------+
    ^                    ^
    |                    |
Local memory        Local memory
(fast access)       (fast access)
    |                    |
    +--------------------+
    Cross-node memory access
    (slower, higher latency)
```

### NUMA Nodes

A **NUMA node** (or proximity domain) is a set of resources with similar access characteristics:
- One or more CPUs
- Local memory
- Sometimes I/O devices (PCIe devices, GPUs)

### NUMA and Performance

**Good NUMA Awareness**:
```c
// Process allocates memory and runs on the same NUMA node
Node 0: CPU 0 → Access Memory 0 (fast, ~100ns)
```

**Poor NUMA Awareness**:
```c
// Process on Node 0 accesses memory on Node 1
Node 0: CPU 0 → Access Memory 1 (slow, ~150-200ns)
```

**Performance Impact**: 1.5x to 2x slower for remote memory access.

### NUMA in Linux

The Linux kernel provides NUMA awareness:
- `numactl`: Control NUMA policy for processes
- `numastat`: Display NUMA statistics
- Memory policies: local allocation, interleaving, binding
- CPU affinity: Pin processes to specific NUMA nodes

**Example Commands**:
```bash
# Show NUMA topology
numactl --hardware

# Run program on NUMA node 0
numactl --cpunodebind=0 --membind=0 ./myprogram

# Show per-node memory usage
numastat
```

---

## 3. SRAT (System Resource Affinity Table)

### What is SRAT?

The System Resource Affinity Table (SRAT) is an ACPI table that describes the association between system resources (CPUs, memory, devices) and NUMA nodes. The OS uses this information to make NUMA-aware scheduling and memory allocation decisions.

### SRAT Structure

The SRAT contains multiple affinity structures that describe different resource types:

**1. Processor Local APIC/SAPIC Affinity (Type 0)**
- Associates a CPU core with a NUMA node
- Contains: APIC ID (CPU identifier), Proximity Domain (NUMA node)

**2. Memory Affinity (Type 1)**
- Associates a memory range with a NUMA node
- Contains: Base address, length, proximity domain, flags (hot-pluggable, non-volatile)

**3. Processor Local x2APIC Affinity (Type 2)**
- Like Type 0, but for x2APIC (supports more CPUs)

**4. GICC Affinity (Type 3)**
- For ARM GIC (Generic Interrupt Controller)

**5. Generic Initiator Affinity (Type 5)** ← **This is what we implemented!**
- Associates a device (like a GPU) with a NUMA node
- Introduced in ACPI 6.4 specification

### SRAT Table Example

Here's what a simple SRAT might describe:

```
SRAT (System Resource Affinity Table):
  Processor Affinity: CPU 0-3 → Node 0
  Processor Affinity: CPU 4-7 → Node 1
  Memory Affinity: 0x0-0x80000000 (2GB) → Node 0
  Memory Affinity: 0x80000000-0x100000000 (2GB) → Node 1
  Generic Initiator: PCI Device 0000:01:00.0 → Node 1
```

### Why SRAT Matters

Without SRAT:
- OS treats all memory as equidistant
- Cannot optimize for NUMA locality
- Poor performance on NUMA systems

With SRAT:
- OS knows which memory is local to each CPU
- Can allocate memory near the CPU that will use it
- Schedules processes on CPUs near their memory
- 20-50% performance improvement on NUMA-aware applications

### How Linux Uses SRAT

1. **Boot Time**: Parse SRAT to build NUMA topology map
2. **Memory Allocation**: Prefer local node memory (`alloc_pages_node()`)
3. **Process Scheduling**: Keep process on same NUMA node (scheduler affinity)
4. **Device Affinity**: Route interrupts to CPUs on same NUMA node as device

---

## 4. PCI BDF (Bus:Device.Function)

### What is PCI?

PCI (Peripheral Component Interconnect) is the standard for connecting peripheral devices (network cards, GPUs, storage controllers) to the computer.

**PCIe (PCI Express)** is the modern high-speed version of PCI, using point-to-point serial links instead of a shared parallel bus.

### PCI Address Space

PCI devices are addressed using a hierarchical scheme:

**Segment (Domain)**: 16-bit (0-65535)
- Groups of PCI buses (typically 0 for most systems)
- Allows for multiple PCI hierarchies

**Bus**: 8-bit (0-255)
- A collection of devices on a shared bus
- Bus 0 is the root bus

**Device**: 5-bit (0-31)
- A physical device slot on a bus
- Each bus can have up to 32 devices

**Function**: 3-bit (0-7)
- A logical function within a device
- Multi-function devices (e.g., GPU with audio) have multiple functions

### BDF Notation

**Common Format**: `segment:bus:device.function`

**Examples**:
```
0000:00:00.0  - Segment 0, Bus 0, Device 0, Function 0 (host bridge)
0000:01:00.0  - Segment 0, Bus 1, Device 0, Function 0 (GPU)
0000:03:00.1  - Segment 0, Bus 3, Device 0, Function 1 (device function 1)
0001:00:00.0  - Segment 1, Bus 0, Device 0, Function 0 (different segment)
```

**Short Format** (when segment is 0): `bus:device.function`
```
01:00.0  - Bus 1, Device 0, Function 0
```

### PCI Enumeration

During boot, the OS discovers PCI devices:

1. **Scan Bus 0**: Look for devices on the root bus
2. **Discover Bridges**: Find PCI-to-PCI bridges (which create secondary buses)
3. **Recurse**: Scan secondary buses for more devices
4. **Assign Resources**: Allocate memory and I/O addresses to devices
5. **Load Drivers**: Match devices to appropriate drivers

### PCI Configuration Space

Each PCI device has a 256-byte (or 4KB for PCIe) configuration space:

```
Offset  Size  Field
------  ----  -----
0x00    2     Vendor ID
0x02    2     Device ID
0x04    2     Command
0x06    2     Status
0x08    1     Revision ID
0x09    3     Class Code
0x0C    1     Cache Line Size
0x0D    1     Latency Timer
0x0E    1     Header Type
0x0F    1     BIST
0x10    4     BAR 0 (Base Address Register)
...
```

**In Linux**:
```bash
# List all PCI devices
lspci

# Detailed info for specific device
lspci -vvv -s 01:00.0

# Show tree structure
lspci -tv

# Configuration space (hexdump)
lspci -xxx -s 01:00.0
```

### PCIe Topology Example

```
Root Complex (0000:00:00.0)
├── PCIe Switch (0000:00:01.0)
│   ├── GPU 0 (0000:01:00.0)
│   └── GPU 1 (0000:02:00.0)
├── NIC (0000:00:02.0 → 0000:03:00.0)
└── NVMe (0000:00:03.0 → 0000:04:00.0)
```

---

## 5. VFIO (Virtual Function I/O)

### What is VFIO?

VFIO (Virtual Function I/O) is a Linux kernel framework that allows safe, direct device access from userspace. It's primarily used for **device passthrough** to virtual machines.

### Why VFIO?

**Traditional Device Access**:
- Kernel drivers control devices
- VMs access devices through emulation (slow)
- Example: Emulated NIC is 10-100x slower than real NIC

**VFIO Device Passthrough**:
- VM has direct access to physical device
- Near-native performance (95-99%)
- Device isolated from host using IOMMU

### IOMMU (Input-Output Memory Management Unit)

The IOMMU is hardware that provides:
- **Memory Protection**: Devices can only access allowed memory regions
- **Address Translation**: Virtual addresses for devices (like MMU for CPUs)
- **Isolation**: Each device has its own address space

**Example**:
```
Without IOMMU:
  Device → Physical Memory (can access anything - security risk!)

With IOMMU:
  Device → IOMMU → Physical Memory (controlled, safe access)
```

**IOMMU Technologies**:
- Intel: VT-d (Virtualization Technology for Directed I/O)
- AMD: AMD-Vi (I/O Virtualization)
- ARM: SMMU (System Memory Management Unit)

### VFIO Architecture

```
┌─────────────────────────────────────┐
│        Virtual Machine (Guest)      │
│  ┌──────────────────────────────┐   │
│  │  Guest Driver (e.g., nvidia) │   │
│  └──────────────────────────────┘   │
│            ↓ Direct Access          │
└────────────┼────────────────────────┘
             ↓
┌────────────┼────────────────────────┐
│    Host    ↓                        │
│  ┌─────────────────┐                │
│  │   VFIO Driver   │                │
│  └─────────────────┘                │
│            ↓                        │
│  ┌─────────────────┐                │
│  │      IOMMU      │                │
│  └─────────────────┘                │
│            ↓                        │
│  ┌─────────────────┐                │
│  │  Physical GPU   │                │
│  └─────────────────┘                │
└─────────────────────────────────────┘
```

### VFIO Setup Steps

**1. Enable IOMMU in BIOS/UEFI**

**2. Enable IOMMU in Kernel**:
```bash
# For Intel (add to kernel command line)
intel_iommu=on iommu=pt

# For AMD
amd_iommu=on iommu=pt
```

**3. Bind Device to VFIO Driver**:
```bash
# Find device BDF
lspci -nn | grep NVIDIA

# Example output: 01:00.0 VGA compatible controller [0300]: NVIDIA [10de:1eb8]

# Unbind from current driver
echo "0000:01:00.0" > /sys/bus/pci/devices/0000:01:00.0/driver/unbind

# Bind to vfio-pci driver
echo "10de 1eb8" > /sys/bus/pci/drivers/vfio-pci/new_id
```

**4. Pass Device to VM**:
```bash
cloud-hypervisor \
    --cpus boot=4 \
    --memory size=8G \
    --device path=/sys/bus/pci/devices/0000:01:00.0
```

### VFIO Groups and Containers

**IOMMU Group**: Set of devices that share IOMMU context
- All devices in a group must be assigned together
- Ensures isolation and security

**Example**:
```bash
# Show IOMMU groups
ls -l /sys/kernel/iommu_groups/

# Devices in group 15
ls /sys/kernel/iommu_groups/15/devices/
# Output: 0000:01:00.0  0000:01:00.1
# Both GPU and GPU audio must be passed together
```

### VFIO Use Cases

1. **GPU Passthrough**: Gaming VMs with native GPU performance
2. **Network Cards**: High-throughput networking in VMs
3. **Storage Controllers**: Direct NVMe access for databases
4. **FPGA/Accelerators**: ML/AI workloads with specialized hardware
5. **MIG Instances**: Partitioned GPU slices (more on this below)

---

## 6. GPU MIG (Multi-Instance GPU)

### What is MIG?

MIG (Multi-Instance GPU) is an NVIDIA technology (introduced with A100 GPUs) that partitions a single physical GPU into multiple isolated GPU instances, each with dedicated resources.

### Why MIG?

**Problem**: Traditional GPU sharing
- Time-slicing (poor performance, context switch overhead)
- Entire GPU to one workload (wasteful if workload is small)
- No isolation between workloads

**MIG Solution**:
- Hardware-level partitioning
- Guaranteed QoS (Quality of Service)
- Full isolation between instances
- Each instance appears as a separate GPU

### MIG Architecture

```
┌─────────────────────────────────────────────────────┐
│          NVIDIA A100 GPU (40GB)                     │
│                                                     │
│  ┌───────────────┐  ┌───────────────┐             │
│  │  MIG Instance │  │  MIG Instance │             │
│  │      1        │  │      2        │             │
│  ├───────────────┤  ├───────────────┤             │
│  │ 3 SMs         │  │ 4 SMs         │             │
│  │ 20GB Memory   │  │ 20GB Memory   │             │
│  │ 200GB/s BW    │  │ 200GB/s BW    │             │
│  └───────────────┘  └───────────────┘             │
│                                                     │
└─────────────────────────────────────────────────────┘
   Each instance has:
   - Dedicated Streaming Multiprocessors (SMs)
   - Dedicated Memory and Memory Controllers
   - Dedicated L2 Cache Slices
```

**SM (Streaming Multiprocessor)**: The core compute unit in NVIDIA GPUs.

### MIG Instance Profiles

NVIDIA A100 supports different MIG profiles:

```
Profile    SMs  Memory  GPU Instances  Use Case
---------  ---  ------  -------------  ---------
1g.5gb     1    5GB     7              Small inference
2g.10gb    2    10GB    3              Medium inference
3g.20gb    3    20GB    2              Training/Large inference
4g.20gb    4    20GB    1              Large workloads
7g.40gb    7    40GB    1              Full GPU (no MIG)
```

**Example Configuration**:
```bash
# Enable MIG mode on GPU 0
nvidia-smi -i 0 -mig 1

# Create MIG instances
nvidia-smi mig -cgi 3g.20gb -C  # Create 3g.20gb instance

# List MIG instances
nvidia-smi -L
# Output:
# GPU 0: NVIDIA A100 (UUID: GPU-xxx)
#   MIG 3g.20gb Device 0: (UUID: MIG-GPU-xxx)
#   MIG 3g.20gb Device 1: (UUID: MIG-GPU-yyy)
```

### MIG Device Representation

Each MIG instance appears as a separate PCI device:

```bash
lspci | grep NVIDIA
# Output:
# 0000:01:00.0 3D controller: NVIDIA A100 (rev a1)
# 0000:01:00.1 3D controller: NVIDIA A100 MIG Instance (rev a1)
# 0000:01:00.2 3D controller: NVIDIA A100 MIG Instance (rev a1)
```

### MIG with VFIO

MIG instances can be passed through to VMs using VFIO:

```bash
# Pass MIG instance 1 to VM
cloud-hypervisor \
    --device path=/sys/bus/pci/devices/0000:01:00.1 \
    ...
```

### MIG Memory as NUMA Node

**Key Problem**: MIG instance has dedicated GPU memory, but how does the OS know to allocate data on that GPU's memory for optimal performance?

**Solution**: Use Generic Initiator in SRAT to expose MIG device memory as a NUMA node!

```
NUMA Node 0: CPU 0-3, 16GB RAM
NUMA Node 1: CPU 4-7, 16GB RAM
NUMA Node 2: MIG Instance 1, 20GB GPU Memory (via Generic Initiator)
NUMA Node 3: MIG Instance 2, 20GB GPU Memory (via Generic Initiator)
```

**Benefits**:
- OS can allocate memory close to the device that needs it
- Better performance for GPU workloads
- Standard NUMA APIs work with GPU memory

---

## 7. How They Work Together

Let's trace through a complete example of how all these concepts interact.

### Scenario: VM with 2 NUMA Nodes and MIG GPU

**Physical Setup**:
- Host with 2 NUMA nodes (Node 0: CPU 0-7, Node 1: CPU 8-15)
- NVIDIA A100 GPU with 2 MIG instances on host NUMA node 1
- MIG Instance 1: PCI BDF 0000:01:00.1
- MIG Instance 2: PCI BDF 0000:01:00.2

**Cloud Hypervisor Command**:
```bash
cloud-hypervisor \
    --cpus boot=8 \
    --memory size=32G \
    --numa guest_numa_id=0,cpus=[0-3],memory_zones=mem0 \
    --numa guest_numa_id=1,cpus=[4-7],memory_zones=mem1 \
    --device path=/sys/bus/pci/devices/0000:01:00.1 \
    --generic-initiator pci_bdf=0000:00:04.0,node=1
```

**What Happens**:

**1. Configuration Parsing** (config.rs):
```
GenericInitiatorConfig { pci_bdf: "0000:00:04.0", numa_node: 1 }
```

**2. Validation** (config.rs):
- Parse BDF string → PciBdf struct
- Verify NUMA node 1 exists
- Check PCI segment is valid

**3. VFIO Setup** (device_manager.rs):
- Open `/sys/bus/pci/devices/0000:01:00.1` on host
- Map device into VM's address space
- Device appears at guest BDF `0000:00:04.0`

**4. ACPI Table Generation** (acpi.rs):

**SRAT Table Contents**:
```
SRAT:
  Type 0 (CPU Affinity):
    APIC ID 0-3 → Proximity Domain 0 (Guest NUMA Node 0)
    APIC ID 4-7 → Proximity Domain 1 (Guest NUMA Node 1)

  Type 1 (Memory Affinity):
    Base 0x0, Length 16GB → Proximity Domain 0
    Base 0x400000000, Length 16GB → Proximity Domain 1

  Type 5 (Generic Initiator):
    Device Handle Type: 1 (PCI)
    Device Handle: [0x00, 0x00, 0x00, 0x04, 0x00, ...]
                    ^^^^^^ segment  ^^bus ^^dev.fn
    Proximity Domain: 1 (Guest NUMA Node 1)
    Flags: 0x1 (Enabled)
```

**5. Guest Boot**:
- UEFI/BIOS passes ACPI tables to guest kernel
- Guest kernel parses SRAT during early boot
- Builds internal NUMA topology map:
  ```
  NUMA Node 0: CPU 0-3, Memory 0-16GB
  NUMA Node 1: CPU 4-7, Memory 16-32GB, Device 0000:00:04.0 (initiator)
  ```

**6. Guest OS Operation**:
```c
// Application running on guest CPU 5 (NUMA node 1)
// Allocates memory for GPU computation

// Linux kernel sees:
// - Process on CPU 5 (Node 1)
// - GPU device 0000:00:04.0 is on Node 1 (from SRAT Type 5)
// - Decision: Allocate from Node 1 memory (local to both CPU and GPU)

void *buffer = numa_alloc_onnode(size, 1);  // Allocate on node 1
cuda_memcpy(buffer, gpu_memory);  // Fast! Same NUMA node
```

**Performance Impact**:
- Without Generic Initiator: Memory allocated randomly, possible cross-NUMA access
- With Generic Initiator: Memory allocated on correct NUMA node, optimal performance

---

## 8. Generic Initiator Use Case

### What is a Generic Initiator?

In ACPI terminology, an **initiator** is any component that can initiate memory transactions. Traditionally, this meant CPUs.

A **Generic Initiator** extends this concept to non-CPU devices (like GPUs, FPGAs, NICs with RDMA) that have compute capabilities and access memory.

### Why Generic Initiators Matter

**Traditional NUMA**: Only CPUs are associated with NUMA nodes
```
Node 0: CPU 0-7  + Memory
Node 1: CPU 8-15 + Memory
Device: GPU (no NUMA association)
```

**With Generic Initiator**: Devices can be associated with NUMA nodes
```
Node 0: CPU 0-7    + Memory
Node 1: CPU 8-15   + Memory
Node 2: GPU MIG #1 + GPU Memory (Generic Initiator)
Node 3: GPU MIG #2 + GPU Memory (Generic Initiator)
```

### Use Cases

**1. GPU MIG Instances** (our implementation):
- Each MIG instance has dedicated GPU memory
- Expose MIG instance as a NUMA node
- OS allocates system memory close to MIG instance
- Better performance for GPU<->CPU data transfers

**2. SmartNICs/DPUs** (Data Processing Units):
- NICs with onboard CPUs and memory
- Expose SmartNIC as NUMA node
- Network packet buffers allocated optimally

**3. FPGAs with HBM** (High Bandwidth Memory):
- FPGA has dedicated HBM memory
- Expose FPGA as NUMA node
- OS-aware memory placement for FPGA workloads

**4. CXL Devices** (Compute Express Link):
- CXL devices have cache-coherent memory
- Can be memory expanders, accelerators, etc.
- Generic Initiator exposes their memory topology

### Benefits of Generic Initiator

1. **Performance**:
   - Optimal memory placement
   - Reduced cross-NUMA latency
   - Better cache utilization

2. **Standard APIs**:
   - Use existing NUMA APIs (`numactl`, `mbind`, `numa_alloc_onnode`)
   - No device-specific code needed
   - Works with existing NUMA-aware applications

3. **Scheduling**:
   - OS can schedule processes near relevant devices
   - Better CPU affinity decisions

4. **Visibility**:
   - `numactl --hardware` shows device topology
   - Monitoring tools understand device placement
   - Better system observability

### Real-World Example

**Scenario**: ML training with MIG instances

```bash
# Without Generic Initiator:
# Memory allocated anywhere, possible cross-NUMA access
python train.py
# Result: 1000 images/sec

# With Generic Initiator:
# Memory allocated on GPU's NUMA node
numactl --membind=2 python train.py
# Result: 1200 images/sec (20% faster!)
```

**Why faster?**
- Input data allocated on NUMA node 2 (GPU's node)
- GPU reads data with optimal locality
- Reduced PCIe traffic
- Lower latency

---

## Summary

Here's how all these concepts connect in the Generic Initiator implementation:

1. **ACPI**: Provides the framework for communicating hardware topology to the OS via standardized tables

2. **NUMA**: The architectural model where memory locality matters for performance

3. **SRAT**: The specific ACPI table that describes NUMA topology, including our new Generic Initiator affinity structures

4. **PCI BDF**: The addressing scheme used to identify devices in the Generic Initiator SRAT entries

5. **VFIO**: The mechanism for passing through physical devices (like MIG instances) to VMs

6. **MIG**: The technology that creates isolated GPU partitions, each of which can be exposed as a separate NUMA node via Generic Initiator

**The Big Picture**:
```
MIG GPU Instance (physical hardware)
    ↓ (passed through via)
VFIO Device (0000:01:00.1 on host → 0000:00:04.0 in guest)
    ↓ (identified by)
PCI BDF (0000:00:04.0)
    ↓ (associated with NUMA node via)
Generic Initiator Affinity Structure (Type 5)
    ↓ (in the)
SRAT Table
    ↓ (part of)
ACPI Tables
    ↓ (describing)
NUMA Topology
    ↓ (used by OS for)
Optimal Memory Allocation and Scheduling
```

This implementation allows Cloud Hypervisor to properly expose GPU MIG instances as NUMA initiators, enabling guest operating systems to make intelligent, performance-optimized decisions about memory placement and CPU scheduling.

---

## Additional Resources

- **ACPI Specification**: https://uefi.org/specifications
- **NUMA**: `man 7 numa` and `man numactl`
- **PCI**: PCI Express Base Specification
- **VFIO**: https://www.kernel.org/doc/html/latest/driver-api/vfio.html
- **MIG**: NVIDIA Multi-Instance GPU documentation
- **Linux NUMA**: https://www.kernel.org/doc/html/latest/vm/numa.html
