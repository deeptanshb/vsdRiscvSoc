# Week 1: RISC-V Bare-Metal Toolchain & Debugging

This document outlines the tasks completed in Week 1 of the RISC-V SoC Lab, focusing on setting up the bare-metal RISC-V toolchain, cross-compiling, analyzing assembly, debugging, and understanding core RISC-V concepts.

## 1. RISC-V Toolchain Setup and Verification

This section details the process of unpacking the xPack RISC-V toolchain, adding it to the system's PATH, and verifying the functionality of `gcc`, `objdump`, and `gdb` binaries.

### Steps:
1. **Download the Toolchain**: Used `wget` to download the `xpack-riscv-none-elf-gcc` archive.
```bash
wget https://vsd-labs.sgp1.cdn.digitaloceanspaces.com/vsd-labs/riscv-toolchain-rv32imac-x86_64-ubuntu.tar.gz
```

2. **Unpack the Archive**: Extracted the contents of the downloaded tarball, creating a folder named `xpack-riscv-none-elf-gcc-14.2.0-3`.
```bash
tar -xvzf xpack-riscv-none-elf-gcc-14.2.0-3-linux-x64.tar.gz
```

3. **Add the Toolchain to PATH (Temporarily)**: Added the toolchain binaries to the current session's PATH.
```bash
export PATH=$HOME/Desktop/vscodeflow/opt/riscv/bin:$PATH
```

4. **Check if Binaries Work**: Verified the functionality of `gcc`, `objdump`, and `gdb` by checking their versions.
```bash
riscv32-unknown-elf-gcc --version
riscv32-unknown-elf-gdb --version
riscv32-unknown-elf-objdump --version

```

### Output:
![image](https://github.com/user-attachments/assets/f8eca9ed-26af-4582-9179-4ff1069ae1a3)


## 2. Minimal C "Hello World" Cross-Compilation for RV32IMC

This section demonstrates how to create a minimal bare-metal C "Hello, World!" program and cross-compile it for the RV32IMC architecture, producing an ELF binary.

### Files:
- **hello.c**: Minimal Bare-Metal Code
```c
// hello.c
#include <stdio.h>
int main() {
    printf("Hello, RISC-V\n");
    return 0;
}

// Minimal _start (no C runtime)
void _start() {
    main();
    while (1); // Trap CPU after main returns
}
```
-- **linker.ld**: Minimal Linker Script for RISC-V RV32IMC
```ld
/* linker.ld - Minimal linker script for RISC-V RV32IMC */
ENTRY(_start)
SECTIONS {
    . = 0x80000000; /* Code starts here */
    .text : {
        *(.text*)
    }
    .data : {
        *(.data*)
    }
    .bss : {
        *(.bss*)
        *(COMMON)
    }
}
```
Both files were saved in the same directory.

### Compile the ELF Binary:
The following command was used to cross-compile `hello.c` into an ELF executable using the specified RISC-V architecture and ABI.
```bash
riscv32-unknown-elf-gcc -march=rv32imc -mabi=ilp32 -o hello.elf hello.c
```

### Verify the ELF:
Used `readelf` to verify the generated ELF file.
```bash
file hello.elf
```

### Output:
![Screenshot from 2025-06-06 17-00-53](https://github.com/user-attachments/assets/86b7da62-4527-4038-86e1-3db128e39c37)
![image](https://github.com/user-attachments/assets/c8fddcb0-dfb5-436d-8b96-69ed91fe6ffe)

## 3. Generating .s File and Explaining Prologue/Epilogue

This section covers generating the assembly (`.s`) file from the C code and explaining the prologue and epilogue sequences of the `main` function.

### Step 1: Generate the .s File
The `-S` flag was used with `gcc` to generate the assembly output.
```bash
riscv32-unknown-elf-gcc -S -O0 hello.c
riscv32-unknown-elf-gcc -c hello.c

```

### Step 2: Look at the Assembly (hello.s)
An example snippet of the `main` function assembly, illustrating stack allocation and register saving/restoring.
```assembly
main:
    addi    sp,sp,-32   # Allocate 32 bytes stack
    sw      ra,28(sp)   # Save return address
    sw      s0,24(sp)   # Save frame pointer
    addi    s0,sp,32    # Set up frame pointer (s0=old sp)
    # ... function body ...
    lw      ra,28(sp)   # Restore return address
    lw      s0,24(sp)   # Restore frame pointer
    addi    sp,sp,32    # Deallocate stack
    ret                 # Return
```

### Step 3: Explain Prologue and Epilogue
- **Prologue (Function Entry)**: Allocates stack space, saves the return address (`ra`), and the caller’s frame pointer (`s0`). Sets up the new frame pointer for safe function calls and variable isolation.
```assembly
addi    sp,sp,-32   # Allocate stack space
sw      ra,28(sp)   # Save return address
sw      s0,24(sp)   # Save frame pointer (s0)
addi    s0,sp,32    # Set new frame pointer
```

- **Epilogue (Function Exit)**: Restores the saved `ra` and `s0` registers, frees the allocated stack space, and returns control to the caller.
```assembly
lw      ra,28(sp)   # Restore return address
lw      s0,24(sp)   # Restore old frame pointer
addi    sp,sp,32    # Free stack space
ret                 # Return to caller
```

### Output:
![Screenshot from 2025-06-06 00-11-04](https://github.com/user-attachments/assets/61a45241-310e-41fd-95ed-80f9c176930f)

## 4. Converting ELF to Raw Hex and Disassembly with Objdump

This section describes converting the ELF file into a raw binary or hex format and disassembling it using `objdump`, explaining the columns of the disassembly output.

### 1. Convert ELF to Raw Binary or Hex
- **Option A: Generate .bin (raw binary)**
```bash
riscv32-unknown-elf-objdump -d hello.elf > hello.dump
```
This produces a flat binary without headers, suitable for direct memory loading.

- **Option B: Generate .hex (Intel HEX format)**
```bash
riscv32-unknown-elf-objcopy -O ihex hello.elf hello.hex
```
This generates an Intel HEX file, often used by flash programmers.

### 2. View the ELF File
- **Basic disassembly**:
```bash
cat hello.dump
```

- View the hex file:
```bash
cat hello.hex
```

### 3. Understanding the Disassembly Output
Example output:
```
80000000 <main>:
80000000: 1101          addi    sp,sp,-32
80000002: ce06          sw      ra,28(sp)
80000004: cc22          sw      s0,24(sp)
```
Each line contains:
- `80000000`: Address of the instruction in memory.
- `1101`: Machine code in hexadecimal.
- `addi sp,sp,-32`: Assembly instruction in human-readable form.

This output shows the exact encoding of each instruction and its location in memory.

### 4. View Binary Contents in Hex
The `hexdump` utility was used to inspect the raw binary contents.
```bash
hexdump -C hello.bin
```
This outputs hexadecimal and ASCII representations of the binary data.

### Output:
![Screenshot from 2025-06-06 17-10-41](https://github.com/user-attachments/assets/aa0fb075-bd81-4c18-acca-ceb9550a316a)


## 5. RV32 Integer Registers and Calling Convention Roles

This section provides a complete list of the 32 integer registers in RV32I, along with their ABI names and conventional roles in the calling convention.

### RV32I Integer Register Table:
| Reg No | ABI Name | Role / Usage |
|--------|----------|--------------|
| x0     | zero     | Constant zero (hardwired, always 0) |
| x1     | ra       | Return address |
| x2     | sp       | Stack pointer |
| x3     | gp       | Global pointer |
| x4     | tp       | Thread pointer |
| x5     | t0       | Temporary register 0 |
| x6     | t1       | Temporary register 1 |
| x7     | t2       | Temporary register 2 |
| x8     | s0 / fp  | Saved register 0 / Frame pointer |
| x9     | s1       | Saved register 1 |
| x10    | a0       | Argument 0 / Return value |
| x11    | a1       | Argument 1 / Return value |
| x12    | a2       | Argument 2 |
| x13    | a3       | Argument 3 |
| x14    | a4       | Argument 4 |
| x15    | a5       | Argument 5 |
| x16    | a6       | Argument 6 |
| x17    | a7       | Argument 7 |
| x18    | s2       | Saved register 2 |
| x19    | s3       | Saved register 3 |
| x20    | s4       | Saved register 4 |
| x21    | s5       | Saved register 5 |
| x22    | s6       | Saved register 6 |
| x23    | s7       | Saved register 7 |
| x24    | s8       | Saved register 8 |
| x25    | s9       | Saved register 9 |
| x26    | s10      | Saved register 10 |
| x27    | s11      | Saved register 11 |
| x28    | t3       | Temporary register 3 |
| x29    | t4       | Temporary register 4 |
| x30    | t5       | Temporary register 5 |
| x31    | t6       | Temporary register 6 |

### Calling Convention Summary:
- **Caller-saved**: `a0–a7`, `t0–t6`. These registers may be overwritten by a called function. The caller must save them before the call if preservation is needed.
- **Callee-saved**: `s0–s11`. These registers must be preserved by the called function across calls.
- **Special-purpose**:
  - `x0` (zero): Always returns 0.
  - `x1` (ra): Stores the return address from a function call.
  - `x2` (sp): Points to the top of the stack.
  - `x8` (s0/fp): Often used as the frame pointer for stack frames.

## 6. Debugging with riscv32-unknown-elf-gdb

This section demonstrates how to start `riscv32-unknown-elf-gdb` on an ELF, set a breakpoint, step through code, and inspect registers.

### Compile:
The `hello1.c` and `linker.ld` files were compiled to `hello2.elf`.
```bash
riscv32-unknown-elf-gcc -march=rv32imc -mabi=ilp32 -nostartfiles -nostdlib -T   hello1.c -o hello1.elf
```

### Verify ELF:
```bash
riscv32-unknown-elf-gdb hello.elf
```

### Debug with GDB:
```bash
riscv-none-elf-gdb hello2.elf
(gdb) set breakpoint auto-hw off
(gdb) target sim
(gdb) load
(gdb) break main
(gdb) run
(gdb) step
(gdb) stepi
(gdb) info registers
```

### Output:
![image](https://github.com/user-attachments/assets/3efd8625-3d20-4cc2-af88-9c5929112eb7)

![Screenshot from 2025-06-07 22-06-49](https://github.com/user-attachments/assets/379b498f-f9ed-4730-be6b-a78044db3769)

## 7. Booting Bare-Metal ELF with SPIKE 

This section provides the SPIKE to run a printf file

### 1. Compile:
The `hello.c` files were compiled to `hello.elf`
```bash
riscv32-unknown-elf-gcc -march=rv32imc -mabi=ilp32 -nostartfiles -nostdlib -T   hello1.c -o hello1.elf
```

### 2. Verify ELF:
```bash
riscv32-unknown-elf-gdb hello.elf
```
Confirmed: Machine: RISC-V, Class: ELF32, Entry point address: 0x80000000. Ensured `.text` and `.rodata` in FLASH with `R E` flags, and `.data` and `.bss` in RAM with `R W` flags.

### 3. Run with SPIKE:
```bash
spike --isa=rv32imc_zicsr /home/deeptanshu/Desktop/vsdcodeflow/opt/riscv/bin/pk hello.elf
```

### Expected Output:
```
Hello RISC-V
```

### Output:
![Screenshot from 2025-06-08 00-17-37](https://github.com/user-attachments/assets/02c9dace-dbd2-43cd-944e-97abfd448de2)


## 8. Compiler Optimization (-O0 vs. -O2) Analysis

This section compares the assembly differences when compiling with `-O0` (no optimization) versus `-O2` (optimization level 2) and explains why these differences occur.

### 1. Compile with -O0 (No Optimization):
- **Generate the assembly**:
```bash
 riscv32-unknown-elf-gcc -S -O0 hello.c -o hello_O0.s
```

- **Generate view**:
```bash
cat hello_O0.s
```

### 3. Compile with -O2 (Optimization Level 2):
- **Generate the ELF**:
```bash
riscv32-unknown-elf-gcc -S -O2 hello.c -o hello_O2
```

- **Generate assembly**:
```bash
cat hello2_O2.s
```

### 4. Verify Compilation:
```bash
riscv-none-elf-readelf -h hello2_O0.elf
riscv-none-elf-readelf -h hello2_O2.elf
```
### Differences and Why They Occur:
- **Stack Frame Size**:
  - `-O0`: Allocates a larger 48-byte stack frame.
  - `-O2`: Uses a smaller 16-byte stack frame.
  - **Why**: `-O0` prioritizes debuggability by storing all variables on the stack. `-O2` minimizes stack usage by keeping variables in registers.
- **Variable Storage**:
  - `-O0`: Stores variables (e.g., `x` and `i`) on the stack with frequent loads/stores.
  - `-O2`: Keeps variables in registers, avoiding stack access.
  - **Why**: `-O0` avoids register optimization for debugging. `-O2` uses registers to reduce memory access latency.
- **Instruction Count**:
  - `-O0`: More instructions due to redundant loads/stores and verbose stack management.
  - `-O2`: Fewer instructions by eliminating redundant operations.
  - **Why**: `-O0` translates C code literally. `-O2` applies optimizations like register allocation and dead code elimination.
- **Loop Efficiency**:
  - `-O0`: Stack-based addressing and repeated loads for loop variables.
  - `-O2`: Uses a single stack slot for buffers and keeps variables in registers, reducing memory operations.
  - **Why**: `-O2` optimizes loops by minimizing memory accesses and reusing registers.
- **Function Calls**:
  - `-O0`: Explicit calls with full stack frame setup.
  - `-O2`: Similar calls, but surrounding code is optimized.
  - **Why**: `-O0` avoids inlining to preserve function boundaries for debugging. `-O2` could inline small functions but might keep others separate due to complexity.
- **Debug Information**:
  - `-O0`: Includes detailed debug info, making tracing easier.
  - `-O2`: Retains some debug info, but optimized code may complicate debugging.
  - **Why**: `-O0` is designed for debugging. `-O2` prioritizes performance.

### Output:
![Screenshot from 2025-06-07 18-03-05](https://github.com/user-attachments/assets/861da16e-8b35-4807-856b-66559bd433d3)

## 9. Reading Cycle Counter using Inline Assembly (CSR 0xC00)

This section details writing a C function to return the cycle counter by reading CSR 0xC00 using inline assembly, and explains each constraint.

### Compilation and Testing:

### Compile:
```bash
riscv32-unknown-elf-gcc -march=rv32imc_zicsr -mabi=ilp32 -O0   rdcycle.c -o rdcycle.elf
```

### Run in spike:
```bash
spike -d --isa=rv32im /home/deeptanshu/Desktop/vsdcodeflow/opt/riscv/bin/pk rdcycle.elf
```

### Expected Output:
```
Cycles taken: <some number>
```

### Inline Assembly Breakdown:
The inline assembly statement used is:
```c
__asm__ volatile (
    "rdcycle %0"
    : "=r" (cycles)
    :
    :
);
```

- **Assembly Template**: `"rdcycle %0"`
  - `rdcycle`: Reads the CYCLE CSR (0xC00) into a register.
  - `%0`: Placeholder for the output operand (`cycles`).
- **Output Constraint**: `"=r" (cycles)`
  - `=`: Indicates the operand is write-only.
  - `r`: Specifies a general-purpose register. The compiler allocates a register to store the result.
  - `(cycles)`: Binds the result to the C variable `cycles` (a `uint32_t`). After execution, the value in the allocated register is copied to `cycles`.
  - **Purpose**: Ensures the cycle count is stored in the `cycles` variable.
- **Input Constraints**: (empty)
  - **Explanation**: `rdcycle` does not require input operands as it directly reads the CSR.
  - **Purpose**: Indicates the instruction operates independently of C-level inputs.
- **Clobbered Registers**: (empty)
  - **Explanation**: No additional registers are modified (clobbered) by `rdcycle` beyond the output register.
  - **Purpose**: Informs the compiler that no registers need to be saved/restored, simplifying code generation.
- **`__asm__ volatile`**:
  - `__asm__`: GCC keyword for inline assembly.
  - `volatile`: Prevents the compiler from optimizing away the assembly.
  - **Purpose**: Ensures the `rdcycle` instruction is executed exactly as written.

### Output:
![image](https://github.com/user-attachments/assets/1686a324-f376-4c3f-9a21-8b407875a938)


## 10. GPIO Register Toggling and Compiler Optimization Prevention

This section outlines a bare-metal C snippet to toggle a GPIO register at `0x10012000` and explains how to prevent the compiler from optimizing away the store operations.

### Compilation and Testing:

### Compile:
```bash
riscv32-unknown-elf-gcc -nostartfiles -nostdlib -T linker.ld   -march=rv32imac -mabi=ilp32 start.S uart.c -o uart.elf
```

### Run in QEMU:
```bash
qemu-system-riscv32 -nographic -machine virt -bios none -kernel uart.elf
```

### Expected Output:
```
10101010...
```

### Bare-Metal C Snippet to Toggle GPIO:
```c
// gpio_toggle.c
#define UART ((volatile char *)0x10000000)

void delay() {
    for (volatile int i = 0; i < 100000; ++i); // crude delay
}

int main() {
    for (int i = 0; i < 10; ++i) {
        *UART = '1';
        delay();
        *UART = '0';
        delay();
    }
    return 0;
}

```
Using `volatile` ensures that every read and write to `GPIO_REGISTER` is performed and not optimized away by the compiler, critical for interacting with hardware registers.

### Output:
![image](https://github.com/user-attachments/assets/9ee4cc2b-0cf9-4012-8026-e980954f578d)


## 11. Minimal Linker Script with Specific Section Placement

This section provides a minimal linker script that places the `.text` section at `0x00000000` and the `.data` section at `0x10000000` for RV32IMC.

### Linker Script (link.ld):
```ld
/* link.ld - Minimal linker script for custom memory layout */
ENTRY(_start)

MEMORY {
  ROM (rx)  : ORIGIN = 0x80000000, LENGTH = 128K
  RAM (rwx) : ORIGIN = 0x80020000, LENGTH = 128K
}

SECTIONS {
  .text : {
    *(.text*)
    *(.rodata*)
  } > ROM

  .data : {
    *(.data*)
  } > RAM

  .bss : {
    *(.bss*)
    *(COMMON)
  } > RAM

  .stack (NOLOAD) : ALIGN(4) {
    _stack_start = .;
    . += 0x1000;  /* 4 KB stack */
    _stack_top = .;
  } > RAM
}

```

### Compile Command:
```bash
riscv32-unknown-elf-gcc -march=rv32imac -mabi=ilp32   -nostdlib -nostartfiles -T linker.ld   start.S uart.c -o uart.elf
```
Ensure `_start.s` contains the `_start` label, sets up the stack, and jumps to `main`.

### Output:
![image](https://github.com/user-attachments/assets/30938deb-9067-4c49-b01e-ee8addc45afc)


## 12. Understanding start.S in Bare-Metal RISC-V

This section explains the role of `start.S` (C runtime zero) in a bare-metal RISC-V program and its typical tasks.

### What crt0.S Typically Does:
`crt0.S` is the first assembly file executed after a RISC-V processor resets. It sets up the execution environment before `main()` is called, crucial when a full C library like `newlib` or `glibc` is not used.

| Step | Task |
|------|------|
| 1    | Set up the stack pointer (sp) |
| 2    | Initialize .bss (zero out uninitialized data) |
| 3    | Optionally copy .data from flash to RAM |
| 4    | Call the main() function |
| 5    | Handle main() return (often enters an infinite loop or triggers an exit routine) |

### starter Script (start.S):
```S
.section .text
.globl _start
_start:
    la sp, _stack_top      # Initialize stack
    call main              # Call main()
1:  j 1b                   # Loop forever

```

## 13. Enabling Machine-Timer Interrupt (MTIP) and Simple Handler

This section demonstrates how to enable the Machine-Timer Interrupt (MTIP) and write a simple handler in C/assembly.

### Compilation and Testing:
Save Files: `timer.c`, `trap.s`, `start.s`, and `linker1.ld` were saved.
### starter Script (start.S):
```S
.section .text
.global _start
_start:
    la sp, _stack_top
    call main
1:  j 1b
                  # Loop forever

```
### Trap Script (trap.S):
```S
.attribute arch, "rv32i2p1_zicsr2p0"
.section .text
.globl trap_entry

trap_entry:
    addi sp, sp, -16
    sw ra, 12(sp)

    # Check if it's a machine timer interrupt
    lw t0, 0x342  # mcause
    li t1, 0x80000007
    bne t0, t1, trap_exit

    # Re-arm timer
    la t2, mtime
    lw t3, 0(t2)
    addi t3, t3, 100
    la t4, mtimecmp
    sw t3, 0(t4)

    # Output '.' via UART
    li t5, 0x10000000
    li t6, '.'
    sb t6, 0(t5)

trap_exit:
    lw ra, 12(sp)
    addi sp, sp, 16
    mret

.section .data
mtimecmp: .word 0x02004000
mtime:    .word 0x0200BFF8

```

### Linker1 Script (linker1.ld):
```ld
ENTRY(_start)

SECTIONS {
  . = 0x80000000;

  .text : { *(.text*) *(.rodata*) }
  .data : { *(.data*) }
  .bss  : { *(.bss*) *(COMMON) }

  . = ALIGN(16);
  .stack : { . = . + 0x1000; }
  _stack_top = .;
}

```
### Timer (Timer.c):
```c
#include <stdint.h>

#define MTIMECMP ((volatile uint64_t*) 0x02004000)
#define MTIME    ((volatile uint64_t*) 0x0200BFF8)

extern void trap_entry(void);

void enable_timer_interrupt() {
    *MTIMECMP = *MTIME + 1000000;

    // Enable timer interrupt
    asm volatile("csrs mie, %0" :: "r"(1 << 7));
    // Enable global machine interrupts
    asm volatile("csrs mstatus, %0" :: "r"(1 << 3));
}

int main() {
    // Set mtvec to trap handler
    asm volatile("csrw mtvec, %0" :: "r"(trap_entry));

    // Enable timer interrupt
    enable_timer_interrupt();

    // Print initial message
    const char *msg = "Timer enabled ";
    volatile char *uart = (char *)0x10000000;
    while (*msg) *uart++ = *msg++;

    // Loop forever — interrupt will print dots
    while (1);
}


```

### Compile:
```bash
riscv32-unknown-elf-gcc -nostdlib -T linker.ld -o timer.elf   start.S trap.S timer.c
```

### Run in QEMU:
```bash
qemu-system-riscv32 -nographic -machine virt -bios none -kernel timer.elf
```

### Expected Output:
```
S A Timer enabled .MTIP .MTIP ... (MTIP every ~1s, dots continue)
```

### Output:
![image](https://github.com/user-attachments/assets/369f6753-4872-4108-b45e-ad9d7257cbd9)
![Screenshot from 2025-06-08 17-32-36](https://github.com/user-attachments/assets/3e1d3eee-f690-44f7-836f-882a3691bbda)



## 14. Explanation of the 'A' (Atomic) Extension in RV32IMAC

This section explains the 'A' (Atomic) extension in the RISC-V RV32IMAC instruction set architecture, the instructions it adds, and their usefulness.

### Overview of the 'A' Extension:
- **Purpose**: Provides instructions for atomic memory operations, crucial for synchronization in concurrent environments like multi-core processors or operating systems.
- **Base ISA**: RV32IMAC includes the 32-bit base integer (I), multiply-divide (M), atomic (A), and compressed (C) extensions. The 'A' extension adds atomic instructions to RV32I.
- **Atomicity**: Atomic operations ensure a sequence of memory accesses (e.g., read-modify-write) is performed as a single, uninterruptible unit, preventing race conditions when multiple hardware threads (harts) access shared memory.
- **Context**: Useful for implementing locks, semaphores, or shared data structures in bare-metal or OS code in the lab environment.

### Instructions Added by the 'A' Extension:
The 'A' extension introduces two main categories: Load-Reserved/Store-Conditional (LR/SC) and Atomic Memory Operations (AMO).

#### Load-Reserved/Store-Conditional (LR/SC):
- `lr.w rd, (rs1)` (Load-Reserved Word): Loads a 32-bit word from `rs1` into `rd` and establishes a reservation on the memory address.
- `sc.w rd, rs2, (rs1)` (Store-Conditional Word): Attempts to store `rs2` to `rs1`. Succeeds (writes 0 to `rd`) only if the reservation is valid. Fails (writes non-zero to `rd`) if another hart modified the address.
- **Mechanism**: LR/SC forms a pair for atomic read-modify-write. The reservation ensures the location remains unchanged between `lr.w` and `sc.w`. If another hart writes to the reserved address, `sc.w` fails.
- **Constraints**: Must operate on naturally aligned 32-bit addresses. Code between `lr.w` and `sc.w` should be minimal.

#### Atomic Memory Operations (AMO):
These instructions perform a read-modify-write operation atomically in a single step, returning the original value in `rd`:
- `amoadd.w rd, rs2, (rs1)`: Atomic Add
- `amoswap.w rd, rs2, (rs1)`: Atomic Swap
- `amoand.w rd, rs2, (rs1)`: Atomic AND
- `amoor.w rd, rs2, (rs1)`: Atomic OR
- `amoxor.w rd, rs2, (rs1)`: Atomic XOR
- `amomin.w rd, rs2, (rs1)`: Atomic Minimum
- `amomax.w rd, rs2, (rs1)`: Atomic Maximum
- `amominu.w rd, rs2, (rs1)`: Atomic Unsigned Minimum
- `amomaxu.w rd, rs2, (rs1)`: Atomic Unsigned Maximum
- **Constraints**: Operate on 32-bit aligned addresses and are performed atomically by hardware.

### Why These Instructions Are Useful:
- **Synchronization**: Essential for implementing synchronization primitives like locks, mutexes, and semaphores in multi-core systems. For example, `lr.w/sc.w` can implement a spinlock.
- **Data Consistency**: Prevent race conditions when multiple harts access shared variables (e.g., counters, queues). `amoadd.w` can atomically increment a shared counter.
- **Efficiency**: AMOs perform complex operations in one instruction, reducing software overhead. LR/SC allows flexible atomic operations with minimal hardware complexity.
- **Scalability**: In multi-core SoCs, atomic instructions ensure scalable synchronization without relying on global bus locks, which degrade performance.
- **Use Cases**: Operating systems (thread scheduling, resource allocation), bare-metal (coordinating harts), atomic counters, lock-free data structures, interrupt-safe updates.

## 15. Two-Thread Mutex Example using LR/SC on RV32

This section provides a two-thread mutex example using Load-Reserved/Store-Conditional (LR/SC) on RV32.
```
### Atomic_lock (Atomic_lock.c):
```c
#include <stdint.h>

volatile int lock = 0;

void lock_acquire() {
    int tmp;
    do {
        asm volatile(
            "lr.w %0, (%1)\n"
            "sc.w %0, %2, (%1)\n"
            : "=&r"(tmp)
            : "r"(&lock), "r"(1)
            : "memory"
        );
    } while (tmp != 0);

    volatile char *uart = (char *)0x10000000;
    *uart = 'L';  // print 'L' on UART on lock acquire
}

void lock_release() {
    asm volatile("sw zero, 0(%0)" : : "r"(&lock) : "memory");
}

void main() {
    volatile char *uart = (char *)0x10000000;
    const char *msg = "Starting test\n";
    for (const char *p = msg; *p; p++) {
        *uart = *p;
    }

    for (int i = 0; i < 10; i++) {
        lock_acquire();
        lock_release();
    }

    while (1) {
        *uart = '.';
        for (volatile int i = 0; i < 100000; i++);  // delay
    }
} 


```

### Compilation and Testing:

### Compile:
```bash
riscv32-unknown-elf-gcc -nostartfiles -nostdlib -T linker.ld   -march=rv32imac -mabi=ilp32 start.S atomic_lock.c -o atomic.elf

```
### Run in QEMU:
```bash
qemu-system-riscv32 -nographic -machine virt -bios none -kernel atomic.elf
```

### Expected Output:
```
LLLLLLLLLLLLLLL................
```

### Output:
![image](https://github.com/user-attachments/assets/88051b2d-72e4-4081-8fdb-5b6dbb7c7c58)


# 16. Retargeting _write for printf to UART

To retarget `printf` to send bytes to a memory-mapped UART at `0x10000000` (with status register at `0x10000005`, bit 5 for TX ready) in a bare-metal RISC-V environment, implement a custom `_write` function. This function is a low-level I/O hook used by the C standard library (e.g., newlib) to handle output.

## The printf Output Chain
1. `printf`: Formats strings and variables into raw bytes.
2. `fputc`/`fputs`: Passes characters to file streams (e.g., `stdout`).
3. `_write`: Performs the physical output to hardware.

## Role of _write
Newlib provides a weak `_write` implementation. By defining your own `_write`, you override it to direct `printf` output to the UART.

## _write Signature
```c
int _write(int file, char *ptr, int len);
```
- `file`: File descriptor (1 for `stdout`, 2 for `stderr`).
- `ptr`: Buffer containing bytes to send.
- `len`: Number of bytes to send.
- Returns: Number of bytes written or -1 on error.

## Implementation
```c
#include <stdint.h>
#include <unistd.h>

#define UART_ADDR ((volatile uint8_t*)0x10000000)

ssize_t _write(int fd, const void *buf, size_t count) {
    const char *cbuf = (const char *)buf;
    for (size_t i = 0; i < count; i++) {
        UART_ADDR[0] = cbuf[i];
    }
    return count;
}

int main() {
    const char *msg = "Hello RISC-V\n";
    write(1, msg, 12);
    while (1); // infinite loop to keep running
    return 0;
}

```
### Output:
![image](https://github.com/user-attachments/assets/0ae42240-818c-4ccb-b558-6a7ee6c94248)

## Note
For full `printf` support, `_read` and `_sbrk` implementations may also be needed.

# 17. Verifying RV32 Endianness with a Union Trick

RV32 is little-endian by default in most RISC-V implementations (e.g., QEMU `virt`). This section verifies byte ordering using a union trick in C.

## Code (main.c)
```c
#include <stdint.h>
#include <stdio.h>
#include <unistd.h>

// UART MMIO addresses (QEMU virt platform)
volatile uint32_t *const UART_DR = (volatile uint32_t *)0x10000000;
volatile uint32_t *const UART_SR = (volatile uint32_t *)0x10000005;
#define UART_SR_TX_READY (1 << 5)

// Redirect printf output to UART
ssize_t _write(int file, const void *ptr, size_t len) {
    if (file == STDOUT_FILENO || file == STDERR_FILENO) {
        const char *buf = (const char *)ptr;
        for (size_t i = 0; i < len; ++i) {
            while (!(*UART_SR & UART_SR_TX_READY));
            *UART_DR = buf[i];
        }
        return len;
    }
    return -1;
}

int main() {
    union {
        uint32_t value;
        uint8_t bytes[4];
    } endian_test;

    endian_test.value = 0x01020304;

    printf("Byte order:\n");
    for (int i = 0; i < 4; i++) {
        printf("Byte %d: 0x%02x\n", i, endian_test.bytes[i]);
    }

    if (endian_test.bytes[0] == 0x04) {
        printf("Detected: Little Endian\n");
    } else {
        printf("Detected: Big Endian\n");
    }

    return 0;
}

```

## Compile and Run
1. Save `main.c`, `startup.s`:
   ```assembly
   .section .text
   .global _start
   _start:
       j main
   ```
   and `linker.ld`:
   ```ld
  ENTRY(_start)

SECTIONS {
    . = 0x80000000;

    .text : {
        *(.text*)
    } :rx

    .rodata : {
        *(.rodata*)
    } :r

    .data : {
        *(.data*)
    } :rw

    .bss : {
        *(.bss*) *(COMMON)
    } :rw
}

   ```
2. Compile:
   ```bash
riscv32-unknown-elf-gcc main.o start.o -o output.elf -T linker.ld -nostartfiles -nostdlib -march=rv32imac -mabi=ilp32 -lc -lgcc

   ```
3. Run in QEMU:
   ```bash
   qemu-system-riscv32 -M virt -kernel output.elf -nographic -bios none
   ```

## Expected Output
```
Byte order:
Byte 0: 0x04
Byte 1: 0x03
Byte 2: 0x02
Byte 3: 0x01
Little-Endian
```
![image](https://github.com/user-attachments/assets/1ff2eb15-13dd-4c35-b491-ca3be15f294e)



