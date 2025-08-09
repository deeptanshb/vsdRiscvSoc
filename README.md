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

| Instruction     | Opcode   | rd   | rs1  | rs2  | funct3 | funct7   | Binary (32-bit)           | Description         |
|----------------|----------|------|------|------|--------|----------|---------------------------|---------------------|
| add x5, x6, x7  | 0110011  | x5   | x6   | x7   | 000    | 0000000  | 0000000 00111 00110 000 00101 0110011 | x5 = x6 + x7        |
| sub x8, x9, x10 | 0110011  | x8   | x9   | x10  | 000    | 0100000  | 0100000 01010 01001 000 01000 0110011 | x8 = x9 - x10       |
| and x11,x12,x13 | 0110011  | x11  | x12  | x13  | 111    | 0000000  | 0000000 01101 01100 111 01011 0110011 | x11 = x12 & x13     |
| or  x14,x15,x16 | 0110011  | x14  | x15  | x16  | 110    | 0000000  | 0000000 10000 01111 110 01110 0110011 | x14 = x15 \| x16    |
| sll x17,x18,x19 | 0110011  | x17  | x18  | x19  | 001    | 0000000  | 0000000 10011 10010 001 10001 0110011 | x17 = x18 << x19    |

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





