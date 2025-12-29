# Cloud Hypervisor Code Organization Tutorial

This tutorial explains how the Cloud Hypervisor codebase is structured and organized, with a focus on the components relevant to the Generic Initiator implementation.

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Core Components](#3-core-components)
4. [Module Deep Dive](#4-module-deep-dive)
5. [Code Flow: CLI to VM Execution](#5-code-flow-cli-to-vm-execution)
6. [Configuration System](#6-configuration-system)
7. [Device Management](#7-device-management)
8. [ACPI Table Generation](#8-acpi-table-generation)
9. [Build System](#9-build-system)
10. [Testing Infrastructure](#10-testing-infrastructure)

---

## 1. Project Overview

### What is Cloud Hypervisor?

Cloud Hypervisor is a modern, open-source Virtual Machine Monitor (VMM) written in Rust. It's designed for running cloud workloads with a focus on:
- **Security**: Rust's memory safety
- **Simplicity**: Minimal codebase, only essential features
- **Performance**: Optimized for cloud/datacenter use cases
- **Modern**: Supports latest CPU features (VT-x, AMD-V, KVM)

### Design Philosophy

1. **Cloud-First**: Optimized for cloud workloads, not desktop virtualization
2. **Minimalism**: No BIOS emulation, no legacy device support
3. **Rust**: Memory-safe systems programming
4. **KVM-Based**: Uses Linux KVM for hardware virtualization
5. **API-Driven**: Designed for programmatic control

### Comparison with QEMU

| Feature | QEMU | Cloud Hypervisor |
|---------|------|------------------|
| Language | C | Rust |
| Legacy Devices | Yes | No |
| BIOS Emulation | SeaBIOS/OVMF | Minimal UEFI |
| Code Size | ~3M LOC | ~100K LOC |
| Use Case | General purpose | Cloud workloads |
| Security | Manual memory mgmt | Rust safety |

---

## 2. Repository Structure

### Top-Level Layout

```
cloud-hypervisor/
├── src/                    # Main binary (CLI)
├── vmm/                    # Virtual Machine Monitor core
├── arch/                   # Architecture-specific code
├── devices/                # Virtual device implementations
├── pci/                    # PCI subsystem
├── virtio-devices/         # VirtIO device implementations
├── hypervisor/             # Hypervisor abstraction (KVM, MSHV)
├── vhost_user_backend/     # Vhost-user backend support
├── acpi_tables/            # ACPI table generation library
├── vm-migration/           # VM migration support
├── net_util/               # Network utilities
├── block/                  # Block device utilities
├── option_parser/          # CLI option parsing
├── tests/                  # Integration tests
├── scripts/                # Build and test scripts
├── resources/              # Test resources
└── docs/                   # Documentation

Key Configuration Files:
├── Cargo.toml              # Rust workspace definition
├── Cargo.lock              # Dependency lock file
├── build.rs                # Build script
└── rust-toolchain.toml     # Rust version specification
```

### Crate Organization

Cloud Hypervisor is organized as a **Rust workspace** with multiple crates:

```rust
// Cargo.toml (workspace level)
[workspace]
members = [
    "arch",
    "devices",
    "pci",
    "vmm",
    "hypervisor",
    "virtio-devices",
    // ... more crates
]
```

**Benefits of Workspace**:
- Shared dependencies
- Parallel compilation
- Clear module boundaries
- Independent versioning

---

## 3. Core Components

### 3.1 The Main Binary (`src/`)

**Location**: `src/main.rs` (~2000 lines)

**Purpose**:
- CLI interface
- Argument parsing
- Top-level control flow
- API server initialization

**Key Responsibilities**:
```rust
// src/main.rs structure

fn main() {
    // 1. Parse CLI arguments
    let cmd_arguments = create_app().get_matches();

    // 2. Setup logging
    log::setup_logging();

    // 3. Handle different commands
    match cmd_arguments.subcommand() {
        Some(("run", args)) => {
            // Create and run VM
            vm::run(args);
        }
        Some(("api", args)) => {
            // Start API server
            api::start_server(args);
        }
        // ...
    }
}
```

**CLI Framework**: Uses `clap` for argument parsing
```rust
use clap::{Arg, ArgAction, ArgMatches, Command};

fn create_app() -> Command {
    Command::new("cloud-hypervisor")
        .arg(Arg::new("cpus").long("cpus"))
        .arg(Arg::new("memory").long("memory"))
        .arg(Arg::new("kernel").long("kernel"))
        // ... 50+ arguments
}
```

### 3.2 VMM Core (`vmm/`)

**Location**: `vmm/src/` (~20,000 lines across multiple files)

**Purpose**: The heart of Cloud Hypervisor - manages VM lifecycle

**Key Files**:
```
vmm/src/
├── lib.rs              # Crate root, main Vm struct
├── vm.rs               # VM lifecycle management (~3000 lines)
├── config.rs           # Configuration parsing/validation (~4700 lines)
├── vm_config.rs        # Configuration data structures (~2000 lines)
├── device_manager.rs   # Device management (~4000 lines)
├── cpu.rs              # CPU/vCPU management
├── memory_manager.rs   # Guest memory management
├── acpi.rs             # ACPI table generation (~900 lines)
├── api/                # HTTP API server
└── seccomp_filters.rs  # Seccomp security policies
```

**The Vm Struct** (core abstraction):
```rust
pub struct Vm {
    kernel: Option<File>,
    initramfs: Option<File>,
    device_manager: Arc<Mutex<DeviceManager>>,
    config: Arc<Mutex<VmConfig>>,
    memory_manager: Arc<Mutex<MemoryManager>>,
    vm: Arc<dyn hypervisor::Vm>,
    state: VmState,
    // ...
}

impl Vm {
    pub fn new() -> Result<Self>;
    pub fn boot(&mut self) -> Result<()>;
    pub fn pause(&mut self) -> Result<()>;
    pub fn resume(&mut self) -> Result<()>;
    pub fn shutdown(&mut self) -> Result<()>;
    // ...
}
```

### 3.3 Architecture Layer (`arch/`)

**Location**: `arch/src/`

**Purpose**: Architecture-specific code (x86_64, AArch64)

**Structure**:
```
arch/src/
├── lib.rs
├── x86_64/
│   ├── mod.rs
│   ├── interrupts.rs     # Interrupt controllers (APIC, IOAPIC)
│   ├── layout.rs         # Memory layout constants
│   ├── mptable.rs        # MP Table generation
│   └── regs.rs           # CPU register setup
└── aarch64/
    ├── mod.rs
    ├── gic.rs            # ARM GIC (Generic Interrupt Controller)
    ├── layout.rs
    └── regs.rs
```

**Example - Memory Layout** (x86_64):
```rust
// arch/src/x86_64/layout.rs
pub const MEM_32BIT_RESERVED_START: u64 = 0xE000_0000;
pub const APIC_START: u64 = 0xFEE0_0000;
pub const IOAPIC_START: u64 = 0xFEC0_0000;
pub const MMIO_HIGH_START: u64 = 0x1_0000_0000; // 4GB
```

### 3.4 Device Layer (`devices/`)

**Location**: `devices/src/`

**Purpose**: Virtual device implementations (legacy, non-VirtIO)

**Key Devices**:
```
devices/src/
├── lib.rs
├── legacy/
│   ├── serial.rs         # Serial port (16550 UART)
│   ├── i8042.rs          # PS/2 keyboard controller
│   └── cmos.rs           # CMOS/RTC
├── ioapic.rs             # I/O APIC
├── gic/                  # ARM GIC
└── acpi_platform.rs      # ACPI platform device
```

**Device Trait**:
```rust
pub trait BusDevice: Send {
    fn read(&mut self, base: u64, offset: u64, data: &mut [u8]);
    fn write(&mut self, base: u64, offset: u64, data: &[u8]);
}
```

### 3.5 VirtIO Devices (`virtio-devices/`)

**Location**: `virtio-devices/src/`

**Purpose**: VirtIO device implementations (modern, efficient)

**Devices**:
```
virtio-devices/src/
├── lib.rs
├── net.rs                # VirtIO network
├── block.rs              # VirtIO block
├── console.rs            # VirtIO console
├── rng.rs                # VirtIO RNG
├── balloon.rs            # Memory balloon
├── vsock.rs              # VirtIO vsock
└── gpu.rs                # VirtIO GPU
```

**VirtIO**: Modern virtio specification for efficient I/O
- Uses ring buffers for communication
- Reduced VM exits
- Better performance than legacy devices

### 3.6 PCI Subsystem (`pci/`)

**Location**: `pci/src/`

**Purpose**: PCI bus emulation and device management

**Key Files**:
```
pci/src/
├── lib.rs
├── bus.rs                # PCI bus implementation
├── device.rs             # PCI device trait
├── configuration.rs      # PCI config space
├── msix.rs               # MSI-X interrupt support
└── segment.rs            # PCI segment support
```

**PCI Device Trait**:
```rust
pub trait PciDevice: BusDevice {
    fn allocate_bars(&mut self, allocator: &mut SystemAllocator) -> Result<Vec<PciBarConfiguration>>;
    fn write_config_register(&mut self, reg_idx: usize, offset: u64, data: &[u8]);
    fn read_config_register(&self, reg_idx: usize) -> u32;
    // ...
}
```

### 3.7 Hypervisor Abstraction (`hypervisor/`)

**Location**: `hypervisor/src/`

**Purpose**: Abstract over different hypervisor backends (KVM, MSHV)

**Structure**:
```
hypervisor/src/
├── lib.rs                # Trait definitions
├── kvm/                  # Linux KVM backend
│   ├── mod.rs
│   └── x86_64/
└── mshv/                 # Microsoft Hypervisor (MSHV) backend
    ├── mod.rs
    └── x86_64/
```

**Hypervisor Trait**:
```rust
pub trait Hypervisor: Send + Sync {
    fn create_vm(&self) -> Result<Arc<dyn Vm>>;
    // ...
}

pub trait Vm: Send + Sync {
    fn create_vcpu(&self, id: u8) -> Result<Arc<dyn Vcpu>>;
    fn set_user_memory_region(&self, region: MemoryRegion) -> Result<()>;
    // ...
}
```

**Why Abstraction?**
- Support multiple hypervisors (KVM on Linux, MSHV on Windows)
- Easier testing (mock hypervisor)
- Platform portability

---

## 4. Module Deep Dive

### 4.1 Configuration Module (`vmm/src/config.rs`)

**Size**: ~4700 lines (one of the largest files)

**Purpose**:
- Parse CLI options
- Validate configuration
- Convert strings to structured config

**Key Patterns**:

**1. Option Parser**:
```rust
use option_parser::OptionParser;

// Parse: "pci_bdf=0000:01:00.0,node=2"
pub fn parse(s: &str) -> Result<Config> {
    let mut parser = OptionParser::new();
    parser.add("pci_bdf").add("node");
    parser.parse(s)?;

    let pci_bdf = parser.get("pci_bdf").ok_or(Error::MissingField)?;
    let node = parser.convert::<u32>("node")?;

    Ok(Config { pci_bdf, node })
}
```

**2. Validation Pattern**:
```rust
impl VmConfig {
    pub fn validate(&self) -> ValidationResult<()> {
        // Validate CPUs
        self.cpus.validate()?;

        // Validate memory
        self.memory.validate()?;

        // Cross-validate: check NUMA nodes reference valid CPUs
        if let Some(numa) = &self.numa {
            for node in numa {
                node.validate(&self.cpus)?;
            }
        }

        // Validate generic initiators reference valid NUMA nodes
        if let Some(gi_list) = &self.generic_initiators {
            for gi in gi_list {
                gi.validate(&self.numa)?;
            }
        }

        Ok(())
    }
}
```

**3. Error Handling with thiserror**:
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum Error {
    #[error("Invalid CPU count: {0}")]
    InvalidCpuCount(u8),

    #[error("Invalid NUMA node: {0}")]
    InvalidNumaNode(u32),

    #[error("Failed to parse generic initiator: {0}")]
    ParseGenericInitiator(String),
}
```

### 4.2 VM Config Structures (`vmm/src/vm_config.rs`)

**Size**: ~2000 lines

**Purpose**: Define configuration data structures

**Pattern - Builder-style Configuration**:
```rust
#[derive(Clone, Debug, PartialEq, Eq, Deserialize, Serialize)]
pub struct VmConfig {
    pub cpus: CpusConfig,
    pub memory: MemoryConfig,
    pub kernel: Option<KernelConfig>,
    pub disks: Option<Vec<DiskConfig>>,
    pub net: Option<Vec<NetConfig>>,
    pub devices: Option<Vec<DeviceConfig>>,
    pub numa: Option<Vec<NumaConfig>>,
    pub generic_initiators: Option<Vec<GenericInitiatorConfig>>,
    // ... 20+ more fields
}

impl VmConfig {
    pub fn parse(vm_params: VmParams) -> Result<Self> {
        // Convert VmParams (from CLI) to VmConfig (structured)
    }
}
```

**Serde Integration**:
```rust
use serde::{Deserialize, Serialize};

// Config can be serialized to/from JSON
let config = VmConfig { ... };
let json = serde_json::to_string(&config)?;

// Load config from JSON file
let config: VmConfig = serde_json::from_str(&json)?;
```

### 4.3 Device Manager (`vmm/src/device_manager.rs`)

**Size**: ~4000 lines

**Purpose**:
- Create and manage all devices
- PCI bus topology
- Device hotplug
- MMIO and I/O port allocation

**Key Structures**:
```rust
pub struct DeviceManager {
    // PCI bus and segments
    pci_segments: Vec<PciSegment>,

    // Device collections
    virtio_devices: Vec<Arc<Mutex<dyn VirtioDevice>>>,
    vfio_devices: HashMap<String, VfioDeviceArc>,

    // Memory and I/O allocators
    address_manager: Arc<AddressManager>,

    // Configuration
    config: Arc<Mutex<VmConfig>>,

    // Event handling
    exit_evt: EventFd,
    reset_evt: EventFd,
}

impl DeviceManager {
    pub fn new() -> Result<Self>;

    pub fn create_devices(&mut self) -> Result<()> {
        self.add_legacy_devices()?;
        self.add_console_device()?;
        self.add_block_devices()?;
        self.add_net_devices()?;
        self.add_vfio_devices()?;
        // ...
    }

    pub fn add_vfio_device(&mut self, config: &DeviceConfig) -> Result<PciBdf>;
}
```

**Device Creation Flow**:
```rust
// vmm/src/device_manager.rs

fn add_vfio_devices(&mut self) -> Result<()> {
    if let Some(device_list) = &self.config.lock().unwrap().devices {
        for device_cfg in device_list {
            // Open VFIO device
            let vfio_device = VfioDevice::new(&device_cfg.path)?;

            // Get PCI BAR (Base Address Registers)
            let bars = vfio_device.allocate_bars(&mut self.address_manager)?;

            // Add to PCI bus
            let bdf = self.pci_segment.add_device(vfio_device)?;

            // Track device
            self.vfio_devices.insert(device_cfg.path.clone(), vfio_device);
        }
    }
    Ok(())
}
```

### 4.4 ACPI Module (`vmm/src/acpi.rs`)

**Size**: ~900 lines

**Purpose**: Generate ACPI tables for guest firmware/OS

**ACPI Tables Generated**:
- RSDP (Root System Description Pointer)
- RSDT (Root System Description Table)
- XSDT (Extended System Description Table)
- FACP (FADT - Fixed ACPI Description Table)
- DSDT (Differentiated System Description Table)
- MADT (Multiple APIC Description Table)
- SRAT (System Resource Affinity Table) ← **We modified this!**
- SLIT (System Locality Information Table)

**Table Generation Pattern**:
```rust
// acpi_tables crate provides base types
use acpi_tables::{Rsdp, Sdt};

fn create_srat_table(
    numa_nodes: &NumaNodes,
    generic_initiators: &Option<Vec<GenericInitiatorConfig>>,
) -> Sdt {
    let mut srat = Sdt::new(*b"SRAT", 40, 1, *b"CLOUDH", *b"CLOUDHYP", 1);

    // Add CPU affinity structures
    for (node_id, node) in numa_nodes {
        for cpu in &node.cpus {
            srat.append(ProcessorAffinity {
                type_: 0,
                proximity_domain: *node_id,
                apic_id: *cpu,
                flags: 1,  // Enabled
            });
        }
    }

    // Add memory affinity structures
    for (node_id, node) in numa_nodes {
        for memory_zone in &node.memory_zones {
            srat.append(MemoryAffinity {
                type_: 1,
                proximity_domain: *node_id,
                base_addr: memory_zone.start,
                length: memory_zone.size,
                flags: 1,  // Enabled
            });
        }
    }

    // Add generic initiator structures (NEW!)
    if let Some(gi_list) = generic_initiators {
        for gi_config in gi_list {
            let bdf = PciBdf::from_str(&gi_config.pci_bdf)?;
            srat.append(GenericInitiatorAffinity::from_pci_bdf(
                bdf,
                gi_config.numa_node,
            ));
        }
    }

    srat
}
```

**Memory Representation - Packed Structs**:
```rust
#[repr(C, packed)]
#[derive(Default, IntoBytes, Immutable, FromBytes)]
struct GenericInitiatorAffinity {
    pub type_: u8,          // Offset 0
    pub length: u8,         // Offset 1
    _reserved1: u16,        // Offset 2-3
    pub device_handle_type: u8,  // Offset 4
    pub proximity_domain: u32,   // Offset 5-8 (unaligned!)
    pub device_handle: [u8; 16], // Offset 9-24
    pub flags: u32,              // Offset 25-28 (unaligned!)
    _reserved2: u32,             // Offset 29-32
}

// #[repr(C, packed)] ensures no padding between fields
// Matches ACPI specification exactly
assert_eq!(std::mem::size_of::<GenericInitiatorAffinity>(), 32);
```

---

## 5. Code Flow: CLI to VM Execution

Let's trace a complete execution path from CLI command to running VM.

### Example Command
```bash
cloud-hypervisor \
    --cpus boot=4 \
    --memory size=4G \
    --kernel /path/to/vmlinux \
    --disk path=/path/to/disk.img \
    --numa guest_numa_id=0,cpus=[0-1] \
    --numa guest_numa_id=1,cpus=[2-3] \
    --device path=/sys/bus/pci/devices/0000:01:00.0 \
    --generic-initiator pci_bdf=0000:00:04.0,node=1
```

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. CLI Parsing (src/main.rs)                               │
│    - clap parses arguments                                  │
│    - Creates VmParams struct                                │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Config Parsing (vmm/src/config.rs)                      │
│    - VmConfig::parse(vm_params)                             │
│    - Parse each field: cpus, memory, numa, generic_initiators│
│    - GenericInitiatorConfig::parse() for each GI string     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Validation (vmm/src/config.rs)                          │
│    - VmConfig::validate()                                   │
│    - Check NUMA nodes exist                                 │
│    - GenericInitiatorConfig::validate()                     │
│    - Check BDF format, check NUMA node references           │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. VM Creation (vmm/src/lib.rs, vm.rs)                     │
│    - Vm::new(config)                                        │
│    - Create hypervisor backend (KVM/MSHV)                   │
│    - Create memory manager                                  │
│    - Create device manager                                  │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Memory Setup (vmm/src/memory_manager.rs)                │
│    - Allocate guest physical memory                         │
│    - Setup NUMA memory zones                                │
│    - Create memory mappings                                 │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Device Creation (vmm/src/device_manager.rs)             │
│    - DeviceManager::create_devices()                        │
│    - Add serial, RTC, i8042 (legacy)                        │
│    - Add VirtIO devices (net, block, console)               │
│    - Add VFIO devices (GPU MIG instance)                    │
│    - Build PCI topology                                     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. ACPI Generation (vmm/src/acpi.rs)                       │
│    - create_acpi_tables()                                   │
│    - create_srat_table(numa_nodes, generic_initiators)      │
│    - Generate RSDP, XSDT, FACP, DSDT, MADT, SRAT           │
│    - Write tables to guest memory                           │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. CPU Setup (vmm/src/cpu.rs, arch/)                       │
│    - Create vCPUs (KVM_CREATE_VCPU)                         │
│    - Configure registers (RIP, RSP, RFLAGS)                 │
│    - Set entry point (kernel start or firmware)             │
│    - Configure CPUID, MSRs                                  │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. VM Boot (vmm/src/vm.rs)                                 │
│    - Vm::boot()                                             │
│    - Load kernel into guest memory                          │
│    - Setup boot params (Linux boot protocol)                │
│    - Start vCPU threads                                     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 10. vCPU Execution Loop (vmm/src/cpu.rs)                   │
│     - Each vCPU runs in its own thread                      │
│     - KVM_RUN ioctl (enter guest mode)                      │
│     - Handle VM exits:                                      │
│       * MMIO reads/writes → Device emulation                │
│       * I/O port access → Legacy device access              │
│       * Interrupts → Inject into guest                      │
│     - Repeat until VM shutdown                              │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Code Path

**1. src/main.rs** (~line 500):
```rust
fn main() {
    let app = create_app();  // Define CLI args
    let matches = app.get_matches();

    match matches.subcommand() {
        Some(("run", args)) => {
            let vm_params = VmParams::from_arg_matches(args)?;
            vmm::run_vm(vm_params)?;
        }
        // ...
    }
}
```

**2. vmm/src/config.rs** (~line 2000):
```rust
impl VmConfig {
    pub fn parse(vm_params: VmParams) -> Result<Self> {
        // Parse generic initiators
        let generic_initiators = if let Some(gi_list) = vm_params.generic_initiators {
            let mut initiators = Vec::new();
            for gi_str in gi_list {
                initiators.push(GenericInitiatorConfig::parse(&gi_str)?);
            }
            Some(initiators)
        } else {
            None
        };

        let config = VmConfig {
            cpus: CpusConfig::parse(vm_params.cpus)?,
            memory: MemoryConfig::parse(vm_params.memory)?,
            // ...
            generic_initiators,
        };

        config.validate()?;
        Ok(config)
    }
}
```

**3. vmm/src/lib.rs** (~line 300):
```rust
pub fn run_vm(config: VmConfig) -> Result<()> {
    // Create hypervisor
    let hypervisor = hypervisor::new()?;

    // Create VM
    let mut vm = Vm::new(
        Arc::new(Mutex::new(config)),
        hypervisor,
    )?;

    // Boot VM
    vm.boot()?;

    // Run until exit
    vm.run()?;

    Ok(())
}
```

**4. vmm/src/device_manager.rs** (~line 1500):
```rust
impl DeviceManager {
    pub fn create_devices(&mut self, numa_nodes: &NumaNodes) -> Result<()> {
        // ... create other devices ...

        // Add VFIO devices
        self.add_vfio_devices()?;

        // Generate ACPI tables (includes SRAT with GI)
        let acpi_tables = self.create_acpi_tables(numa_nodes)?;

        Ok(())
    }
}
```

**5. vmm/src/acpi.rs** (~line 767):
```rust
pub fn create_acpi_tables() -> Result<Vec<u8>> {
    let srat = create_srat_table(
        &numa_nodes,
        &device_manager.lock().unwrap()
            .config.lock().unwrap()
            .generic_initiators,  // Pass generic initiators
        topology,
    );

    // ... package tables ...
}
```

**6. vmm/src/vm.rs** (~line 800):
```rust
impl Vm {
    pub fn boot(&mut self) -> Result<()> {
        // Setup memory
        self.memory_manager.lock().unwrap().setup_memory()?;

        // Create devices
        self.device_manager.lock().unwrap().create_devices()?;

        // Load kernel
        self.load_kernel()?;

        // Create and start vCPUs
        self.start_vcpus()?;

        Ok(())
    }
}
```

---

## 6. Configuration System

### Configuration Flow

```
CLI String → VmParams → VmConfig → Internal Structures
  (user)     (parsing)  (validation)  (runtime)
```

### Example: Generic Initiator Configuration

**CLI String**:
```
"pci_bdf=0000:00:04.0,node=1"
```

**VmParams** (intermediate):
```rust
pub struct VmParams {
    // Raw strings from CLI
    pub generic_initiators: Option<Vec<String>>,
}
```

**VmConfig** (structured):
```rust
pub struct VmConfig {
    pub generic_initiators: Option<Vec<GenericInitiatorConfig>>,
}

pub struct GenericInitiatorConfig {
    pub pci_bdf: String,      // Still string, validated later
    pub numa_node: u32,       // Parsed u32
}
```

**Runtime** (fully resolved):
```rust
// In ACPI generation
let bdf = PciBdf::from_str(&gi_config.pci_bdf)?;  // Parse to struct
let gi_structure = GenericInitiatorAffinity::from_pci_bdf(bdf, gi_config.numa_node);
```

### Configuration Validation Layers

**Layer 1: Syntax Validation** (config.rs parse methods)
- Parse string format
- Check required fields present
- Convert to appropriate types

**Layer 2: Semantic Validation** (config.rs validate methods)
- Check references (NUMA nodes exist, BDF format valid)
- Check ranges (CPU IDs, memory sizes)
- Check for duplicates

**Layer 3: Runtime Validation** (device_manager.rs)
- Verify VFIO device exists
- Check device actually at BDF
- Verify device is VFIO-bound

---

## 7. Device Management

### PCI Topology

Cloud Hypervisor creates a PCI topology similar to real hardware:

```
PCI Segment 0
├── 00:00.0 - Host Bridge
├── 00:01.0 - VirtIO Net
├── 00:02.0 - VirtIO Block
├── 00:03.0 - VirtIO Console
├── 00:04.0 - VFIO Device (GPU MIG Instance) ← Our GI target!
└── 00:1f.0 - ACPI Platform Device
```

### Device Addition Flow

```rust
// vmm/src/device_manager.rs

impl DeviceManager {
    fn add_vfio_device(&mut self, cfg: &DeviceConfig) -> Result<PciBdf> {
        // 1. Open VFIO device
        let device_fd = std::fs::OpenOptions::new()
            .read(true)
            .write(true)
            .open(&cfg.path)?;

        // 2. Create VFIO device wrapper
        let vfio_device = VfioDevice::new(device_fd)?;

        // 3. Allocate PCI resources (BARs, IRQs)
        let bars = vfio_device.allocate_bars(&mut self.address_manager)?;

        // 4. Assign BDF
        let bdf = self.pci_segment.next_available_bdf()?;

        // 5. Add to PCI bus
        self.pci_segment.add_device(bdf, vfio_device)?;

        // 6. Setup DMA mapping (IOMMU)
        vfio_device.setup_dma_map(&self.memory_manager)?;

        Ok(bdf)
    }
}
```

### Device Lifecycle

```
Create → Configure → Attach → Run → Detach → Destroy
   ↓         ↓          ↓       ↓       ↓        ↓
  new()   allocate   add to   handle  remove  drop()
          resources   bus      I/O     from
                                      bus
```

---

## 8. ACPI Table Generation

### Table Generation Pipeline

```
NumaNodes + GenericInitiators
         ↓
create_srat_table()
         ↓
   SRAT (binary)
         ↓
create_acpi_tables()
         ↓
RSDP → XSDT → [FACP, DSDT, MADT, SRAT, SLIT, ...]
         ↓
   Binary blob
         ↓
Write to guest memory at fixed address
         ↓
Guest firmware/OS reads tables
```

### ACPI Table Memory Layout

```
Guest Physical Memory:

0x0000_0000 - 0x0009_FFFF: Low memory (640KB)
0x000A_0000 - 0x000F_FFFF: VGA, BIOS (384KB)
0x0010_0000 - 0xXXXX_XXXX: Main RAM

ACPI Tables Region: 0x00E0_0000 - 0x00E0_FFFF (typically)
  ├── RSDP (Root System Description Pointer)
  ├── XSDT (Extended System Description Table)
  ├── FACP (Fixed ACPI Description Table)
  ├── DSDT (Differentiated System Description Table)
  ├── MADT (Multiple APIC Description Table)
  ├── SRAT (System Resource Affinity Table) ← Includes GI!
  └── SLIT (System Locality Information Table)
```

### ACPI Table Format

All ACPI tables use the `acpi_tables` crate:

```rust
use acpi_tables::{Rsdp, Sdt, aml};

// Create table
let mut srat = Sdt::new(
    *b"SRAT",              // Signature
    40,                    // Header length
    1,                     // Revision
    *b"CLOUDH",            // OEM ID
    *b"CLOUDHYP",          // OEM Table ID
    1,                     // OEM Revision
);

// Append structures
srat.append(some_structure);

// Finalize and get bytes
let srat_bytes = srat.finalize();
```

---

## 9. Build System

### Cargo Workspace

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "arch",
    "devices",
    "vmm",
    "hypervisor",
    # ... all crates
]

[workspace.dependencies]
# Shared dependencies
libc = "0.2"
anyhow = "1.0"
thiserror = "1.0"
serde = { version = "1.0", features = ["derive"] }
```

### Build Commands

```bash
# Build all (release mode)
cargo build --release

# Build specific crate
cargo build -p vmm

# Build with features
cargo build --features "mshv"

# Run tests
cargo test

# Check without building
cargo check

# Format code
cargo fmt

# Lint
cargo clippy
```

### Binary Output

```
target/
├── debug/
│   └── cloud-hypervisor      # Debug binary
└── release/
    └── cloud-hypervisor      # Release binary (optimized)
```

---

## 10. Testing Infrastructure

### Test Types

**1. Unit Tests** (in each file):
```rust
// vmm/src/config.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_generic_initiator_parse() {
        let gi = GenericInitiatorConfig::parse(
            "pci_bdf=0000:01:00.0,node=1"
        ).unwrap();
        assert_eq!(gi.pci_bdf, "0000:01:00.0");
        assert_eq!(gi.numa_node, 1);
    }
}
```

**2. Integration Tests** (`tests/`):
```rust
// tests/integration_tests.rs
#[test]
fn test_vm_boot_with_generic_initiator() {
    let config = VmConfig {
        // ... config ...
        generic_initiators: Some(vec![
            GenericInitiatorConfig {
                pci_bdf: "0000:00:04.0".to_string(),
                numa_node: 1,
            }
        ]),
    };

    let vm = Vm::new(config).unwrap();
    vm.boot().unwrap();
    // ... verify SRAT contents ...
}
```

**3. Performance Tests**:
```bash
# scripts/run-performance-tests.sh
./target/release/cloud-hypervisor --cpus 4 --memory 4G ...
# Measure boot time, throughput, latency
```

---

## Summary: Key Architectural Patterns

### 1. Separation of Concerns
- CLI parsing (src/main.rs)
- Configuration (vm_config.rs, config.rs)
- Device management (device_manager.rs)
- ACPI generation (acpi.rs)

### 2. Trait-Based Abstraction
- `BusDevice` trait for device I/O
- `PciDevice` trait for PCI devices
- `Hypervisor` trait for backend abstraction

### 3. Interior Mutability
- `Arc<Mutex<T>>` for shared mutable state
- Safe concurrent access to VM resources

### 4. Error Handling
- `Result<T, Error>` everywhere
- `thiserror` for error types
- Propagation with `?` operator

### 5. Configuration Flow
- Parse → Validate → Transform → Execute
- Early validation prevents runtime errors

### 6. Memory Safety
- Rust ownership prevents use-after-free
- No manual memory management
- Safe FFI with careful `unsafe` blocks

---

## File Size Reference

```
Large files (>2000 lines):
- vmm/src/config.rs:        ~4700 lines (parsing, validation)
- vmm/src/device_manager.rs: ~4000 lines (device lifecycle)
- vmm/src/vm.rs:            ~3000 lines (VM lifecycle)
- src/main.rs:              ~2000 lines (CLI, API server)
- vmm/src/vm_config.rs:     ~2000 lines (config structures)

Medium files (500-2000 lines):
- vmm/src/acpi.rs:          ~900 lines (ACPI generation)
- vmm/src/cpu.rs:           ~800 lines (vCPU management)
- vmm/src/memory_manager.rs: ~1500 lines (memory management)

Our Changes:
- vm_config.rs:    +10 lines (struct definition)
- config.rs:       +150 lines (parsing, validation, errors)
- acpi.rs:         +50 lines (GenericInitiatorAffinity struct + logic)
- main.rs:         +5 lines (CLI arg)
```

---

## Navigation Tips

When working with Cloud Hypervisor:

1. **Start with `src/main.rs`**: Understand CLI and entry points
2. **Then `vmm/src/vm_config.rs`**: See available configuration
3. **Then `vmm/src/config.rs`**: Understand parsing and validation
4. **Then `vmm/src/vm.rs`**: See VM lifecycle
5. **Then `vmm/src/device_manager.rs`**: Device creation and management
6. **Then specialized files**: ACPI, CPU, memory as needed

**Finding Code**:
```bash
# Find definition
rg "pub struct VmConfig"

# Find usage
rg "VmConfig::parse"

# Find tests
rg "#\[test\].*generic_initiator" -A 10
```

This tutorial should give you a solid understanding of Cloud Hypervisor's architecture and how your Generic Initiator changes fit into the broader codebase!
