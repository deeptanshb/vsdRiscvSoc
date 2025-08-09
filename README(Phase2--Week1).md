## 🧾 **Title**

> **Phase 2--Week 1: RISC-V Toolchain Tasks & System Validation using Spike and Proxy Kernel**

---

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
```
## Output:
<img width="801" height="98" alt="image" src="https://github.com/user-attachments/assets/d6d3557d-a65a-49d4-8430-cabd4cec103d" />

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


