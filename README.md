## 🧾 **Title**
## 🎯 **Objective**

The objective of this phase is to extend the RISC-V toolchain workflow by setting up and validating the full bare-metal execution flow using:

- ✅ The **Spike RISC-V ISA simulator**
- ✅ The **Proxy Kernel (pk)** as a minimal bootloader and execution environment
- ✅ Cross-compilation of a C program into a RISC-V binary
- ✅ Execution of compiled RISC-V programs on Spike
- ✅ System-specific uniqueness verification using user/host-based hashing
- ✅ Handling toolchain/environment setup issues (QEMU, NVIDIA DKMS, PATHs, etc.)

This week focuses on low-level build and execution testing, emulator validation, proxy kernel linking, and debugging toolchain inconsistencies.


# ✅ Phase 2: Environment Preparation & Toolchain Setup

## ✅ Task 0: Environment Preparation (Linux)
If you're on **Ubuntu 22.04**, use your native system.
```bash
sudo apt-get update
```
## Output:
<img width="768" height="517" alt="image" src="https://github.com/user-attachments/assets/721ea715-bcba-4f59-a5c3-b9bc2776cb46" />

## 🛠️ Task 1 — Install Base Developer Tools

```bash
sudo apt-get install -y git vim autoconf automake autotools-dev curl \
libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex \
texinfo gperf libtool patchutils bc zlib1g-dev libexpat1-dev gtkwave
```
## Output:
<img width="804" height="517" alt="image" src="https://github.com/user-attachments/assets/51815a06-f019-493b-97f1-d79af8346fd1" />

## 📁  Task 2 — Create Workspace
```bash
cd
pwd=$PWD
mkdir -p riscv_toolchain
cd riscv_toolchain
```
## 📦 Task 3 — Download Prebuilt RISC‑V GCC Toolchain
```bash
wget "https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14.tar.gz"
tar -xvzf riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14.tar.gz
```
## Output:
<img width="804" height="517" alt="image" src="https://github.com/user-attachments/assets/c7eed53f-73b9-46d4-9d95-a0b2bc3630b3" />

## 🧭 Task 4 — Add Toolchain to PATH
Temporary (current session):
```bash
export PATH=$pwd/riscv_toolchain/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14/bin:$PATH
```
Permanant Session:
```bash
echo 'export PATH=$HOME/riscv_toolchain/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
## Output:
<img width="804" height="188" alt="image" src="https://github.com/user-attachments/assets/0266182e-4c3d-4546-9b1b-8a090def3f77" />

## 🌳 Task 5 — Install Device Tree Compiler
```bash
sudo apt-get install -y device-tree-compiler
```
## Output:
<img width="804" height="197" alt="image" src="https://github.com/user-attachments/assets/60da748a-d4aa-4815-a44a-15ede9ee7c04" />

## 🏗️ Task 6 — Build Spike (ISA Simulator)
```bash
cd $pwd/riscv_toolchain
git clone https://github.com/riscv/riscv-isa-sim.git
cd riscv-isa-sim
mkdir build && cd build
../configure --prefix=$pwd/riscv_toolchain/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14
make -j$(nproc)
sudo make install
```
## ⚙️ Task 7 — Build the Proxy Kernel
```bash
cd $pwd/riscv_toolchain
git clone https://github.com/riscv/riscv-pk.git
cd riscv-pk
mkdir build && cd build
../configure --prefix=$HOME/riscv_toolchain/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14 \
--host=riscv64-unknown-elf
make -j$(nproc)
sudo make install
```
## ✅ Task 8 — Final Sanity Check
```bash
which pk
spike pk --help
spike --help | head -n 1
```
## Output:
<img width="801" height="98" alt="image" src="https://github.com/user-attachments/assets/d6d3557d-a65a-49d4-8430-cabd4cec103d" />
<img width="784" height="92" alt="Screenshot from 2025-08-10 12-18-02" src="https://github.com/user-attachments/assets/65ecea5e-93f0-44c9-8b32-c6974dfc3096" />


## 🧠 Task 9 — Create unique_test.c
```bash
#define _GNU_SOURCE

#include <stdint.h>
#include <stdio.h>

#ifndef USERNAME
#define USERNAME "unknown_user"
#warning "USERNAME not defined. Using default: unknown_user"
#endif

#ifndef HOSTNAME
#define HOSTNAME "unknown_host"
#warning "HOSTNAME not defined. Using default: unknown_host"
#endif

static uint64_t fnv1a64(const char *s) {
    const uint64_t FNV_OFFSET = 1469598103934665603ULL;
    const uint64_t FNV_PRIME  = 1099511628211ULL;
    uint64_t h = FNV_OFFSET;
    for (const unsigned char *p = (const unsigned char*)s; *p; ++p) {
        h ^= (uint64_t)(*p);
        h *= FNV_PRIME;
    }
    return h;
}

int main(void) {
    const char *user = USERNAME;
    const char *host = HOSTNAME;

    char buf[256];
    int n = snprintf(buf, sizeof(buf), "%s@%s", user, host);
    if (n <= 0) return 1;

    uint64_t uid = fnv1a64(buf);

    printf("RISC-V Uniqueness Check\n");
    printf("User: %s\n", user);
    printf("Host: %s\n", host);
    printf("UniqueID: 0x%016llx\n", (unsigned long long)uid);

#ifdef __VERSION__
    unsigned long long vlen = (unsigned long long)sizeof(__VERSION__) - 1;
    printf("GCC_VLEN: %llu\n", vlen);
#endif

    return 0;
}
```
## Output:
<img width="801" height="507" alt="image" src="https://github.com/user-attachments/assets/a99283df-6992-483c-9dfd-2186a3d48116" />

## 🛠️ Task 10 — Compile the Program
```bash
riscv64-unknown-elf-gcc -O2 -Wall -march=rv64imac -mabi=lp64 \
-DUSERNAME="\"$(id -un)\"" -DHOSTNAME="\"$(hostname -s)\"" \
unique_test.c -o unique_test
```
## Output:
<img width="804" height="197" alt="Screenshot from 2025-08-02 15-40-53" src="https://github.com/user-attachments/assets/be501b50-dab6-47f8-b70b-9b36d184e5d3" />

## ▶️ Task 11 — Run Using Spike + Proxy Kernel
```bash
spike pk ./unique_test
```
## Output:
<img width="816" height="223" alt="image" src="https://github.com/user-attachments/assets/420a0a0c-1aa6-4cfa-a5b0-bef7c2c30738" />

# ✅ RISC-V Phase2--Week 2: Local Setup Check and ISA Instruction Decoding

---

## 🎯 Objective

The goal of this task is to:
- ✅ Compile and run multiple C programs using a **local RISC-V toolchain**
- ✅ Embed machine-unique identity information into the program output using a header (`unique.h`)
- ✅ Generate and inspect assembly and disassembled code
- ✅ Manually decode 5 RISC-V **integer instructions**
- ✅ Verify execution using **Spike** simulator and demonstrate output uniqueness

---

## 🛠️ Step-by-Step Process

---

### 🔹 1. Setup Identity Variables (Run in Linux Terminal)

```bash
export U=$(id -un)
export H=$(hostname -s)
export M=$(cat /etc/machine-id | head -c 16)
export T=$(date -u +%Y-%m-%dT%H:%M:%SZ)
export E=$(date +%s)
```

🧠 **Explanation**:  
These environment variables store your user, machine, and timestamp info. They’re injected into the program during compilation to make each build uniquely yours.

---

### 🔹 2. Add `unique.h` Header File

This file generates a **ProofID** and **RunID** with your identity + timestamps using the `fnv1a64()` hash function.

Include it in your `.c` files like:
```c
#ifndef UNIQUE_H
#define UNIQUE_H

#include <stdio.h>
#include <stdint.h>
#include <time.h>

#ifndef USERNAME
#define USERNAME "unknown_user"
#endif

#ifndef HOSTNAME
#define HOSTNAME "unknown_host"
#endif

#ifndef MACHINE_ID
#define MACHINE_ID "unknown_machine"
#endif

#ifndef BUILD_UTC
#define BUILD_UTC "unknown_time"
#endif

#ifndef BUILD_EPOCH
#define BUILD_EPOCH 0
#endif

static uint64_t fnv1a64(const char *s) {
    const uint64_t OFF = 1469598103934665603ULL;
    const uint64_t PRIME = 1099511628211ULL;
    uint64_t h = OFF;

    for (const unsigned char *p = (const unsigned char*)s; *p; ++p) {
        h ^= *p;
        h *= PRIME;
    }

    return h;
}

static void uniq_print_header(const char *program_name) {
    time_t now = time(NULL);
    char buf[512];
    int n = snprintf(buf, sizeof(buf), "%s|%s|%s|%s|%ld|%s|%s",
        USERNAME, HOSTNAME, MACHINE_ID, BUILD_UTC,
        (long)BUILD_EPOCH, __VERSION__, program_name);
    (void)n;

    uint64_t proof = fnv1a64(buf);

    char rbuf[600];
    snprintf(rbuf, sizeof(rbuf), "%s|run_epoch=%ld", buf, (long)now);
    uint64_t runid = fnv1a64(rbuf);

    printf("=== RISC-V Proof Header ===\n");
    printf("User : %s\n", USERNAME);
    printf("Host : %s\n", HOSTNAME);
    printf("MachineID : %s\n", MACHINE_ID);
    printf("BuildUTC : %s\n", BUILD_UTC);
    printf("BuildEpoch : %ld\n", (long)BUILD_EPOCH);
    printf("GCC : %s\n", __VERSION__);
    printf("PointerBits: %d\n", (int)(8 * (int)sizeof(void*)));
    printf("Program : %s\n", program_name);
    printf("ProofID : 0x%016llx\n", (unsigned long long)proof);
    printf("RunID : 0x%016llx\n", (unsigned long long)runid);
    printf("===========================\n");
}

#endif
```
### 🔹 3. Write C Programs

Each program starts by calling:

```c
uniq_print_header("program_name");
```

Programs include:

`factorial.c`
```c
#include "unique.h"

static unsigned long long fact(unsigned n){ return (n<2)?1ULL:n*fact(n-1); }

int main(void){
    uniq_print_header("factorial");
    unsigned n = 12;
    printf("n=%u, n!=%llu\n", n, fact(n));
    return 0;
}
```
`bitops.c`
```c
#include "unique.h"

int main(void){
    uniq_print_header("bitops");
    unsigned x=0xA5A5A5A5u, y=0x0F0F1234u;
    printf("x&y=0x%08X\n", x&y);
    printf("x|y=0x%08X\n", x|y);
    printf("x^y=0x%08X\n", x^y);
    printf("x<<3=0x%08X\n", x<<3);
    printf("y>>2=0x%08X\n", y>>2);
    return 0;
}
```
`max_array.c`
```c
#include "unique.h"

int main(void){
    uniq_print_header("max_array");
    int a[] = {42,-7,19,88,3,88,5,-100,37};
    int n = sizeof(a)/sizeof(a[0]), max=a[0];
    for(int i=1;i<n;i++) if(a[i]>max) max=a[i];
    printf("Array length=%d, Max=%d\n", n, max);
    return 0;
}
```
`bubble_sort.c`
```c
#include "unique.h"

void bubble(int *a,int n){ for(int i=0;i<n-1;i++) for(int j=0;j<n-1-i;j++) if(a[j]>a[j+1]){int t=a[j];a[j]=a[j+1];a[j+1]=t;} }

int main(void){
    uniq_print_header("bubble_sort");
    int a[]={9,4,1,7,3,8,2,6,5}, n=sizeof(a)/sizeof(a[0]);
    bubble(a,n);
    printf("Sorted:"); for(int i=0;i<n;i++) printf(" %d",a[i]); puts("");
    return 0;
}
```
---

### 🔹 4. Compile Each Program

`factorial.c`
```bash
riscv64-unknown-elf-gcc -O0 -g -march=rv64imac -mabi=lp64 \
-DUSERNAME="\"$U\"" -DHOSTNAME="\"$H\"" -DMACHINE_ID="\"$M\"" \
-DBUILD_UTC="\"$T\"" -DBUILD_EPOCH=$E \
factorial.c -o factorial
```
`bitops.c`
```bash
riscv64-unknown-elf-gcc -O0 -g -march=rv64imac -mabi=lp64 \
-DUSERNAME="\"$U\"" -DHOSTNAME="\"$H\"" -DMACHINE_ID="\"$M\"" \
-DBUILD_UTC="\"$T\"" -DBUILD_EPOCH=$E \
bitops.c -o bitops
```
`max_array.c`
```bash
riscv64-unknown-elf-gcc -O0 -g -march=rv64imac -mabi=lp64 \
-DUSERNAME="\"$U\"" -DHOSTNAME="\"$H\"" -DMACHINE_ID="\"$M\"" \
-DBUILD_UTC="\"$T\"" -DBUILD_EPOCH=$E \
max_array.c -o max_array
```
`bubble_sort.c`
```bash
riscv64-unknown-elf-gcc -O0 -g -march=rv64imac -mabi=lp64 \
-DUSERNAME="\"$U\"" -DHOSTNAME="\"$H\"" -DMACHINE_ID="\"$M\"" \
-DBUILD_UTC="\"$T\"" -DBUILD_EPOCH=$E \
bubble_sort.c -o bubble_sort
```
🧠 **Explanation**:  
This builds the program for the **RISC-V 64-bit architecture**, and embeds your machine info using the `-D` flags.

---
### 🔹 5. Run with Spike Simulator

`factorial.c`
```bash
spike pk ./factorial
```
### Output:
<img width="800" height="477" alt="factorial" src="https://github.com/user-attachments/assets/42e75d1f-d4c0-4a33-8027-2f2187c1353b" />

`bitops.c`
```bash
spike pk ./bitops
```
### Output:
<img width="799" height="453" alt="bitops" src="https://github.com/user-attachments/assets/531c0d78-938d-4d28-aa23-714eb0220eb6" />

`max_array.c`
```bash
spike pk ./max_array
```
### Output:
<img width="799" height="453" alt="max_array" src="https://github.com/user-attachments/assets/cc333c05-5408-4e61-9e39-973cd5602477" />

`bubble_sort.c`
```bash
spike pk ./bubble_sort
```
### Output:
<img width="803" height="453" alt="bubble_sort" src="https://github.com/user-attachments/assets/12ba172a-27cd-4543-bd71-017fec24c073" />

---

### 🔹 6. Generate Assembly and Disassembly

#### ✅ Assembly (.s):
`factorial`
```bash
riscv64-unknown-elf-gcc -O0 -S factorial.c -o factorial.s
```
`max_array`
```bash
riscv64-unknown-elf-gcc -O0 -S max_array.c -o max_array.s
```
`bitops`
```bash
riscv64-unknown-elf-gcc -O0 -S bitops.c -o bitops.s
```
`bubble_sort`
```bash
riscv64-unknown-elf-gcc -O0 -S bubble_sort.c -o bubble_sort.s
```
### Output:
<img width="803" height="191" alt="Screenshot from 2025-08-04 19-14-55" src="https://github.com/user-attachments/assets/5193468c-274f-45c2-9aa7-97a09c634c6b" />

#### ✅ Disassemble main:

`factorial`
```bash
riscv64-unknown-elf-objdump -d ./factorial | sed -n '/<main>:/,/^$/p' | tee factorial_main_objdump.txt
```
### Output:
<img width="1756" height="613" alt="Screenshot from 2025-08-04 19-21-05" src="https://github.com/user-attachments/assets/8ce478f2-fb78-4afc-a125-2d822f7ab20c" />

`max_array`
```bash
riscv64-unknown-elf-objdump -d ./max_array | sed -n '/<main>:/,/^$/p' | tee max_array_main_objdump.txt
```
### Output:
<img width="1756" height="994" alt="Screenshot from 2025-08-04 19-30-07" src="https://github.com/user-attachments/assets/ef320aff-eb72-493d-9c1e-4d0d9a23f211" />

`bitops`
```bash
riscv64-unknown-elf-objdump -d ./bitops | sed -n '/<main>:/,/^$/p' | tee bitops_main_objdump.txt
```
### Output:
<img width="1856" height="994" alt="Screenshot from 2025-08-04 19-31-15" src="https://github.com/user-attachments/assets/0b743898-0fd8-430b-968c-10eafa8998a0" />

`bubble_sort`
```bash
riscv64-unknown-elf-objdump -d ./bubble_sort | sed -n '/<main>:/,/^$/p' | tee bubble_sort_main_objdump.txt
```
### Output:
<img width="1856" height="994" alt="Screenshot from 2025-08-04 19-32-03" src="https://github.com/user-attachments/assets/0a90e40b-fe65-42a3-9ab4-d910f12f883b" />

---

### 🔹 7. Manual Instruction Decoding

Manually decode at least **5 integer instructions** from `.s` or `.objdump` output.

🧠 Decode into:
- Opcode, funct3, funct7
- Register fields (rd, rs1, rs2)
- Full 32-bit binary
- Simple description

---

## 📘 Instruction Decoding Table

| Instruction       | Opcode   | rd   | rs1  | rs2  | funct3 | funct7   | Binary (32-bit)                 | Description                  |
|-------------------|----------|------|------|------|--------|----------|----------------------------------|------------------------------|
| addi sp, sp, -64  | 0010011  | x2   | x2   | imm  | 000    | —        | imm[11:0]=111111000000 00010 000 00010 0010011 | sp = sp - 64                 |
| sd ra, 56(sp)     | 0100011  | —    | x2   | x1   | 011    | —        | imm[11:5]=0000000 x1=00001 x2=00010 funct3=011 imm[4:0]=111000 | Store ra into mem[sp+56]     |
| ld a5, -24(s0)    | 0000011  | x15  | x8   | imm  | 011    | —        | imm[11:0]=11111101000 rd=01111 rs1=01000 funct3=011 | a5 = mem[s0-24]               |
| sll a5, a5, 0x2   | 0110011  | x15  | x15  | x2   | 001    | 0000000  | funct7=0000000 rs2=00010 rs1=01111 funct3=001 rd=01111 opcode=0110011 | a5 = a5 << 2                  |
| add a5, a5, a4    | 0110011  | x15  | x15  | x14  | 000    | 0000000  | funct7=0000000 rs2=01110 rs1=01111 funct3=000 rd=01111 opcode=0110011 | a5 = a5 + a4                  |


---

## 📎 Notes

- All outputs contain unique **ProofID** and **RunID**.
- Screenshots of terminal and disassembly must be included.
- `.s` and `.objdump` must match your compiled `.elf` file.

---

## ✅ Tool Checks (for README)

### Spike available:
```bash
which spike
```
### Output:
<img width="801" height="124" alt="Screenshot from 2025-08-02 16-51-13" src="https://github.com/user-attachments/assets/6f5bf46d-8773-435c-a99a-acfcda404d30" />

---

## 🏁 Conclusion

This task demonstrates your ability to:
- Set up a RISC-V toolchain locally
- Compile and run real C programs
- Understand RISC-V machine instructions
- Document the entire process with identity proof





