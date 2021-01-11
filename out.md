### ./kernel/x86_64-unknown-none.json
```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "disable-redzone": true,
    "features": "-mmx,+sse",
    "panic-strategy": "unwind",
    "pre-link-args": {
        "ld.lld": [
            "/opt/cross/lib/gcc/x86_64-elf/10.2.0/libgcc.a", "--allow-multiple-definition", "-T/home/pitust/code/an_os/link.ld"
        ]
    },
    "eliminate-frame-pointer": false
}
```
### ./kernel/.cargo/config.toml
```toml

[unstable]
build-std = ["core", "alloc"]

[build]
target = "x86_64-unknown-none.json"
# todo: make my own runner
# [target.'cfg(target_os = "none")']
# runner = "bootimage runner"
```
### ./kernel/package.json
```json
{
  "name": "an_os",
  "version": "1.0.0",
  "description": "i decided i need no fast drive # Sun Sep 20 09:24:24 AM IST 2020 Working on the exec loader since i got the toolchain working. # Sun Sep 20 09:38:28 AM IST 2020 Exec loader now works # Sun Sep 20 01:19:13 PM IST 2020 Got /init running # Mon Sep 21 07:30:43 PM IST 2020 so i decided that a VM is so slow and i decided to preempt and load ELFs. It was really slow, that's why. In retrospect, it was stupid to make a Vm, that was sooooo slow. # Mon Sep 21 10:30:46 PM IST 2020 oh god preemptive multitasking is so confusing. Like this:",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "git+ssh://git@github.com/pitust/an_os.git"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/pitust/an_os/issues"
  },
  "homepage": "https://github.com/pitust/an_os#readme",
  "dependencies": {
    "@types/node": "^14.14.6"
  }
}

```
### ./kernel/sym-city.ts
```typescript
import { execSync } from "child_process";
import { readFileSync, unlinkSync, writeFileSync } from "fs";

// GET DA SYMBOLZ
execSync('objdump --dwarf=decodedline build/kernel.elf >data.txt', { stdio: 'inherit' })
let o = readFileSync('data.txt').toString().trim();
unlinkSync('data.txt');
let cfnm = 'NONE';
let data = {};
for (let l of o.split('\n')) {
    if (l.startsWith('CU: ')) cfnm = l.slice(4, -1).trim();
    else if (l.endsWith(':')) cfnm = l.slice(0, -1).trim();
    else if (l.trim() && cfnm != 'NONE') {
        let [_, lineno, addr] = l.split(/\s/).map(e => e.trim()).filter(e => e);
        if (+addr > 0x200000) {
            data[cfnm] = data[cfnm] || [];
            data[cfnm].push({ addr: +addr, line: +lineno });
        }
    }
}
writeFileSync('build/ksymmap.json', JSON.stringify(data));
execSync('serde_conv -i build/ksymmap.json  -F postcard -o build/ksymmap.pcrd');
execSync('serde_conv -i build/ksymmap.json  -F cpost -o build/ksymmap.epcrd');
```
### ./kernel/asm/boot.s
```null
section .multiboot_header
header_start:
    dd 0xe85250d6                ; magic number (multiboot 2)
    dd 0                         ; architecture 0 (protected mode i386)
    dd header_end - header_start ; header length
    ; checksum
    dd -(0xe85250d6 + 0 + (header_end - header_start))

    ; insert optional multiboot tags here

    ; required end tag
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:

section .bss
align 4096
p4_table:
    resb 4096
p3_table:
    resb 4096
p2_table:
    resb 4096
stack_bottom:
    ; 512K of stack should be good for a while
    resb 4096 * 64
stack_top:

section .rodata
gdt64:
    dq 0 ; zero entry
    dq (1<<43) | (1<<44) | (1<<47) | (1<<53)
.pointer:
    dw $ - gdt64 - 1
    dq gdt64



global _start
extern kmain
section .text
bits 32
_start:
    ;now enable SSE and the like
    mov eax, cr0
    and ax, 0xFFFB		;clear coprocessor emulation CR0.EM
    or ax, 0x2			;set coprocessor monitoring  CR0.MP
    mov cr0, eax
    mov eax, cr4
    or ax, 3 << 9		;set CR4.OSFXSR and CR4.OSXMMEXCPT at the same time
    mov cr4, eax

    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    mov edi, ebx
    call set_up_page_tables
    call enable_paging
    lgdt [gdt64.pointer]

    jmp 8:long_mode_start
long_mode_start:
    call kmain
.end_loop:
    nop
    nop
    nop
    jmp .end_loop



set_up_page_tables:
    ; map first P4 entry to P3 table
    mov eax, p3_table
    or eax, 0b11 ; present + writable
    mov [p4_table], eax

    ; map first P3 entry to P2 table
    mov eax, p2_table
    or eax, 0b11 ; present + writable
    mov [p3_table], eax

    mov ecx, 0
.map_p2_table:
    ; map ecx-th P2 entry to a huge page that starts at address 2MiB*ecx
    mov eax, 0x200000  ; 2MiB
    mul ecx            ; start address of ecx-th page
    or eax, 0b10000011 ; present + writable + huge
    mov [p2_table + ecx * 8], eax ; map ecx-th entry

    inc ecx            ; increase counter
    cmp ecx, 512       ; if counter == 512, the whole P2 table is mapped
    jne .map_p2_table  ; else map the next entry

    ret



enable_paging:
    ; load P4 to cr3 register (cpu uses this to access the P4 table)
    mov eax, p4_table
    mov cr3, eax

    ; enable PAE-flag in cr4 (Physical Address Extension)
    mov eax, cr4
    or eax, 1 << 5
    mov cr4, eax

    ; set the long mode bit in the EFER MSR (model specific register)
    mov ecx, 0xC0000080
    rdmsr
    or eax, 1 << 8
    wrmsr

    ; enable paging in the cr0 register
    mov eax, cr0
    or eax, 1 << 31
    mov cr0, eax

    ret

```
### ./kernel/faster_rlibc/.cargo/config.toml
```toml

[unstable]
build-std = ["core", "compiler_builtins"]

[build]
target = "../target.json"

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```
### ./kernel/faster_rlibc/Cargo.toml
```toml
[package]
name = "faster_rlibc"
version = "0.1.0"
authors = ["pitust <stelmaszek.piotrpilot@gmail.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[build-dependencies]
cc = "1.0.59"

```
### ./kernel/faster_rlibc/src/lib.rs
```rust
#![no_std]
#![no_builtins]
#![feature(llvm_asm)]
#![feature(asm)]
extern crate core;


pub unsafe fn memcpy(dest: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    let mut i = 0;
    while i < n {
        *dest.offset(i as isize) = *src.offset(i as isize);
        i += 1;
    }
    return dest;
}
#[no_mangle]
pub unsafe extern "C" fn fastermemcpy(dest: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    if n % 8 != 0 {
        unreachable!();
    }
    asm!("cld; rep movsq; cld", in("rcx") (n / 8), in("rsi") (src), in("rdi") (dest));
    return dest;
}
#[no_mangle]
pub unsafe extern "C" fn fastermemset(dest: *mut u8, x: u64, n: usize) -> *mut u8 {
    if n % 8 != 0 {
        unreachable!();
    }
    asm!("cld; rep stosq; cld", in("rcx") (n / 8), in("rax") (x), in("rdi") (dest));
    return dest;
}


pub unsafe extern "C" fn memmove(dest: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    if src < dest as *const u8 {
        // copy from end
        let mut i = n;
        while i != 0 {
            i -= 1;
            *dest.offset(i as isize) = *src.offset(i as isize);
        }
    } else {
        // copy from beginning
        let mut i = 0;
        while i < n {
            *dest.offset(i as isize) = *src.offset(i as isize);
            i += 1;
        }
    }
    return dest;
}


pub unsafe extern "C" fn memset(s: *mut u8, c: i32, n: usize) -> *mut u8 {
    let mut i = 0;
    while i < n {
        *s.offset(i as isize) = c as u8;
        i += 1;
    }
    return s;
}


pub unsafe extern "C" fn memcmp(s1: *const u8, s2: *const u8, n: usize) -> i32 {
    let mut i = 0;
    while i < n {
        let a = *s1.offset(i as isize);
        let b = *s2.offset(i as isize);
        if a != b {
            return a as i32 - b as i32;
        }
        i += 1;
    }
    return 0;
}

```
### ./kernel/kmacros/Cargo.toml
```toml
[package]
name = "kmacros"
version = "0.1.0"
authors = ["pitust <stelmaszek.piotrpilot@gmail.com>"]
edition = "2018"
type = 'proc-macro'
# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
syn = { version = "1.0.39", features = ["full", "fold"] }
quote = "1.0"
proc-macro2 = "1.0"

[lib]
proc-macro = true
```
### ./kernel/kmacros/src/lib.rs
```rust
#![feature(type_name_of_val)]
extern crate proc_macro2;
extern crate quote;
extern crate syn;
use std::{
    fs::{self, DirEntry},
    io,
    path::Path,
};

use proc_macro::TokenStream;
use quote::quote;
use syn::ItemFn;
use syn::{parse, parse_macro_input, parse_str, Block, ExprMatch};

fn getfilez(dir: &Path, cb: &mut dyn FnMut(&DirEntry)) -> io::Result<()> {
    assert!(dir.exists());
    if dir.is_dir() {
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.is_dir() {
                getfilez(&path, cb)?;
            } else {
                cb(&entry);
            }
        }
    }
    Ok(())
}
// fn getdirz(dir: &Path, cb: &dyn Fn(&DirEntry)) -> io::Result<()> {
//     if dir.is_dir() {
//         for entry in fs::read_dir(dir)? {
//             let entry = entry?;
//             let path = entry.path();
//             if path.is_dir() {
//                 getfilez(&path, cb)?;
//                 cb(&entry);
//             }
//         }
//     }
//     Ok(())
// }

#[proc_macro_attribute]
pub fn handle_read(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let mut v = vec![];
    getfilez(Path::new("rootfs"), &mut |d| {
        let d: &DirEntry = d;
        let p = d.path();
        let p = p
            .as_os_str()
            .to_str()
            .unwrap()
            .split("rootfs/")
            .nth(1)
            .unwrap();
        v.push(p.to_string());
    })
    .unwrap();

    let inp = parse_macro_input!(item as ItemFn);
    let attrs = inp.attrs.clone();
    let sig = inp.sig.clone();
    let vis = inp.vis.clone();
    let mut o = format!("match path {{ ");
    for k in &v {
        o = format!(
            "{old} {src:?} | {src2:?} => {{ include_bytes!({file:?}) }}, ",
            old = o,
            src = k,
            src2 = format!("/{}", k),
            file = format!("../rootfs/{}", k)
        );
    }
    o = format!(
        "{} _ => panic!(\"File not found (used {{}}), have: {{}} !!!\", path, {:?}) }}",
        o,
        format!("{:?}", v)
    );

    let blk = parse_str::<ExprMatch>(&o).expect(&format!("eYYY: {}", o));
    let expanded = quote! {
        #vis #sig {
            #blk
        }
    };
    let x: proc_macro2::TokenStream = proc_macro2::TokenStream::from(expanded);
    let mut p: String = String::from("");
    for idx in 0..attrs.len() {
        let attr = attrs[idx].clone();
        let z: proc_macro2::TokenStream = quote! { #attr };
        p += &z.to_string();
    }
    p += &x.to_string();
    p.parse().expect("Should parse")
}

```
### ./kernel/safety-here/Cargo.toml
```toml
[package]
name = "safety-here"
version = "0.1.0"
authors = ["pitust <piotr@stelmaszek.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
[build-dependencies]
cc="1.0"
```
### ./kernel/safety-here/src/lib.rs
```rust
#![no_std]
#![feature(global_asm)]
global_asm!(include_str!("main.s"));

#[derive(Debug, Copy, Clone)]
#[repr(C)]
pub struct Jmpbuf {
    pub rbx: u64,
    pub rbp: u64,
    pub r12: u64,
    pub r13: u64,
    pub r14: u64,
    pub r15: u64,
    pub rsp: u64,
    pub rip: u64,
    pub rsi: u64,
}
impl Jmpbuf {
    pub fn new() -> Jmpbuf {
        Jmpbuf {
            rbx: 0,
            rbp: 0,
            r12: 0,
            r13: 0,
            r14: 0,
            r15: 0,
            rsp: 0,
            rip: 0,
            rsi: 0,
        }
    }
}
pub type PushJmpbuf = extern "C" fn(&mut Jmpbuf);
extern {
    pub fn setjmp(buf: &mut Jmpbuf) -> i32;
    pub fn longjmp(buf: &Jmpbuf, val: i32) -> !;
    pub fn changecontext(fcn: PushJmpbuf, ref_to_a_buffer: &mut Jmpbuf);
}
```
### ./kernel/safety-here/src/main.s
```null
// This and x64_mach.s are from newlib, found code at https://github.com/mirror/newlib-cygwin/blob/17918cc6a6e6471162177a1125c6208ecce8a72e/newlib/libc/machine/x86_64/.

/*
* ====================================================
* Copyright (C) 2007 by Ellips BV. All rights reserved.
*
* Permission to use, copy, modify, and distribute this
* software is freely granted, provided that this notice
* is preserved.
* ====================================================
*/

/*
**  jmp_buf:
**   rbx rbp r12 r13 r14 r15 rsp rip
**   0   8   16  24  32  40  48  56
*/

#include "x64mach.s"

.global setjmp
.global longjmp

setjmp:
    movq    %rbx,  0 (%rdi)
    movq    %rbp,  8 (%rdi)
    movq    %r12, 16 (%rdi)
    movq    %r13, 24 (%rdi)
    movq    %r14, 32 (%rdi)
    movq    %r15, 40 (%rdi)
    leaq    8 (%rsp), %rax
    movq    %rax, 48 (%rdi)
    movq    (%rsp), %rax
    movq    %rax, 56 (%rdi)
    movq    $0, %rax
    ret

longjmp:
    movq    %rsi, %rax        /* Return value */

    movq     8 (%rdi), %rbp

    movq    48 (%rdi), %rsp
    pushq   56 (%rdi)
    movq     0 (%rdi), %rbx
    movq    16 (%rdi), %r12
    movq    24 (%rdi), %r13
    movq    32 (%rdi), %r14
    movq    40 (%rdi), %r15

ret
```
### ./kernel/safety-here/src/swap.c
```null
// no std libs here lol. but C must stay. (actually, this is because of rusts's rules, breaking my code. whatever.)

// this will memcpy it over. so nice. 
typedef void (*push_jmpbuf_t)(void *);
extern int setjmp(void* buf);
extern __attribute__((noreturn)) void longjmp(void* buf, int code);
void changecontext(push_jmpbuf_t fcn, void* ref_to_a_buffer) {
    if (setjmp(ref_to_a_buffer) == 1) {
        return;
    }
    fcn(ref_to_a_buffer);
    longjmp(ref_to_a_buffer, 1);
}
```
### ./kernel/safety-here/src/x64_mach.s
```null
/*
 ** This file is distributed WITHOUT ANY WARRANTY; without even the implied
 ** warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 */

#ifndef __USER_LABEL_PREFIX__
#define __USER_LABEL_PREFIX__ _
#endif

#define __REG_PREFIX__ %

/* ANSI concatenation macros.  */

#define CONCAT1(a, b) CONCAT2(a, b)
#define CONCAT2(a, b) a##b

/* Use the right prefix for global labels.  */

#define SYM(x) CONCAT1(__USER_LABEL_PREFIX__, x)

/* Use the right prefix for registers.  */

#define REG(x) CONCAT1(__REG_PREFIX__, x)

#define rax REG(rax)
#define rbx REG(rbx)
#define rcx REG(rcx)
#define rdx REG(rdx)
#define rsi REG(rsi)
#define rdi REG(rdi)
#define rbp REG(rbp)
#define rsp REG(rsp)

#define r8  REG(r8)
#define r9  REG(r9)
#define r10 REG(r10)
#define r11 REG(r11)
#define r12 REG(r12)
#define r13 REG(r13)
#define r14 REG(r14)
#define r15 REG(r15)

#define eax REG(eax)
#define ebx REG(ebx)
#define ecx REG(ecx)
#define edx REG(edx)
#define esi REG(esi)
#define edi REG(edi)
#define ebp REG(ebp)
#define esp REG(esp)

#define st0 REG(st)
#define st1 REG(st(1))
#define st2 REG(st(2))
#define st3 REG(st(3))
#define st4 REG(st(4))
#define st5 REG(st(5))
#define st6 REG(st(6))
#define st7 REG(st(7))

#define ax REG(ax)
#define bx REG(bx)
#define cx REG(cx)
#define dx REG(dx)

#define ah REG(ah)
#define bh REG(bh)
#define ch REG(ch)
#define dh REG(dh)

#define al REG(al)
#define bl REG(bl)
#define cl REG(cl)
#define dl REG(dl)

#define sil REG(sil)

#define mm1 REG(mm1)
#define mm2 REG(mm2)
#define mm3 REG(mm3)
#define mm4 REG(mm4)
#define mm5 REG(mm5)
#define mm6 REG(mm6)
#define mm7 REG(mm7)

#define xmm0 REG(xmm0)
#define xmm1 REG(xmm1)
#define xmm2 REG(xmm2)
#define xmm3 REG(xmm3)
#define xmm4 REG(xmm4)
#define xmm5 REG(xmm5)
#define xmm6 REG(xmm6)
#define xmm7 REG(xmm7)

#define cr0 REG(cr0)
#define cr1 REG(cr1)
#define cr2 REG(cr2)
#define cr3 REG(cr3)
#define cr4 REG(cr4)

#ifdef _I386MACH_NEED_SOTYPE_FUNCTION
#define SOTYPE_FUNCTION(sym) .type SYM(sym),@function
#else
#define SOTYPE_FUNCTION(sym)
#endif

#ifndef _I386MACH_DISABLE_HW_INTERRUPTS
#define        __CLI
#define        __STI
#else
#define __CLI  cli
#define __STI  sti
#endif
```
### ./kernel/src/pci.rs
```rust
use crate::{dbg, print, println};
use alloc::string::{String, ToString};
use alloc::vec::Vec;
use x86_64::instructions::port::Port;
pub const CONFIG_ADDRESS: u16 = 0xCF8;
pub const CONFIG_DATA: u16 = 0xCFC;
// pub const PCI_TABLE: &str = include_str!("../pci.txt");
// Ported from C, original at https://wiki.osdev.org/PCI
fn pci_read16(bus: u8, slot: u8, func: u8, offset: u8) -> u16 {
    let mut addrp: Port<u32> = Port::new(CONFIG_ADDRESS);
    let mut datap: Port<u16> = Port::new(CONFIG_DATA);
    let lbus = bus as u32;
    let lslot = slot as u32;
    let lfunc = func as u32;

    /* create configuration address as per Figure 1 */
    let address = (lbus << 16) | (lslot << 11) | (lfunc << 8) | (offset as u32) | (0x80000000);

    /* write out the address */
    unsafe {
        addrp.write(address);
    }
    /* read in the data */
    unsafe { datap.read() }
}
fn pci_read32(bus: u8, slot: u8, func: u8, offset: u8) -> u32 {
    (pci_read16(bus, slot, func, offset) as u32)
        | ((pci_read16(bus, slot, func, offset + 2) as u32) << 16)
}
fn pci_read8(bus: u8, slot: u8, func: u8, offset: u8) -> u8 {
    pci_read16(bus, slot, func, offset) as u8
}
fn to_int(c: &str) -> Option<u16> {
    let mut r: u16 = 0;
    for ccx in c.chars() {
        r = r * 16;
        let mut cc = ccx as u8;
        if cc < 48 || cc >= 58 {
            if cc > 97 && cc < 103 {
                cc -= 97;
                cc += 48;
                cc += 10;
            } else {
                return None;
            }
        }
        r += (cc - 48) as u16;
    }
    return Some(r);
}
pub fn idlookup(vendor: u16, dev: u16) -> Option<String> {
    // let vec = PCI_TABLE.split('\n');
    // let mut iscv = false;
    // for x in vec {
    //     if x.len() < 4 {
    //         continue;
    //     }
    //     let lelr = to_int(x.clone().split_at(4).0);
    //     if lelr == Some(vendor) {
    //         println!("Found co. for {}", dev);
    //         iscv = true;
    //     } else if lelr.is_some() {
    //         if iscv {
    //             return None;
    //         }
    //         iscv = false;
    //     } else if iscv {
    //         if x.len() < 8 {
    //             continue;
    //         }
    //         let lelr2 = to_int(x.clone().split_at(5).0.split_at(1).1);
    //         if lelr2.is_some() && lelr2 == Some(dev) {
    //             return Some(x.clone().split_at(7).1.to_string());
    //         }
    //     }
    // }
    return None;
}

pub fn testing() {
    println!("Enumerating PCI");
    for bus_id in 0..255u8 {
        for slot in 0..32u8 {
            let vendor = pci_read16(bus_id, slot, 0, 0);
            let devid = pci_read16(bus_id, slot, 0, 2);
            if vendor != 0xffff && vendor != 0 {
                println!("{}:{}  {:#04x}:{:#04x}", bus_id, slot, vendor, devid);
                if let Some(x) = idlookup(vendor, devid) {
                    println!("  name = {}", x);
                }
                // let some_ram = crate::memory::alloc::ALLOCATOR.get().allocate_first_fit(alloc::alloc::Layout::from_size_align(32, 8).unwrap()).unwrap();
                println!("  header type = {}", pci_read8(bus_id, slot, 0, 0xD));
                println!("  subsystem = {}", pci_read16(bus_id, slot, 0, 0x2e));
                println!("  subsystem vendor = {}", pci_read16(bus_id, slot, 0, 0x2c));
                for i in 0..6 {
                    let barx = pci_read32(bus_id, slot, 0, 0x10 + (i * 4));
                    if barx & 1 == 1 {
                        // iospace
                        println!("  bar{} = io:{:#06x}", i, barx & 0xffffffc);
                    } else {
                        // memory
                        let bartype = (barx >> 1) & 3;
                        println!("  bar{}.raw = {:#06x}", i, barx);
                        if bartype == 0 {
                            println!("  bar{} = mem32:{:#06x}", i, barx & 0xFFFFFFF0);
                        }
                        if bartype == 2 {
                            println!(
                                "  bar{} = mem64:{:#06x}",
                                i,
                                (((barx as u64) & 0xFFFFFFF0u64)
                                    + (((pci_read32(bus_id, slot, 0, 14 + (i * 4)) & 0xFFFFFFFF)
                                        as u64)
                                        << 32))
                            );
                        }
                    }
                }
            }
        }
    }
}

```
### ./kernel/src/unwind.rs
```rust
use crate::prelude::*;
use core::ffi::c_void;
use gimli::{DebugLineOffset, LittleEndian};
use x86_64::VirtAddr;
use xmas_elf::sections::SectionData;
use xmas_elf::{self, symbol_table::Entry};
extern "C" {
    pub fn __register_frame(__frame: *mut c_void, __size: usize);
}
#[no_mangle]
unsafe extern "C" fn strlen(mut s: *mut u8) -> usize {
    let mut i = 0;
    while *s != 0 {
        i += 1;
        s = s.offset(1);
    }
    i
}
#[no_mangle]
unsafe extern "C" fn abort() -> ! {
    panic!("C abort");
}
pub fn basic_unmangle(s: alloc::string::String) -> alloc::string::String {
    if !s.starts_with("_Z") {
        return s;
    }
    let old = cpp_demangle::Symbol::new(s)
        .unwrap()
        .to_string()
        .replace("..", "::")
        .replace("$LT$", "<")
        .replace("$GT$", ">")
        .replace("$u20$", " ")
        .replace("$u7b$", "{")
        .replace("$u7d$", "}");
    let mut vec: Vec<&str> = old.split("::").collect();
    vec.pop();
    return vec.join("::");
}
ezy_static! { SYMBOL_TABLE, BTreeMap<u64, String>, BTreeMap::<u64, String>::new() }
pub fn line_tweaks(debug_line: &[u8]) {
    let d = gimli::DebugLine::new(debug_line, LittleEndian);
    let data = d.program(DebugLineOffset(0), 8, None, None).unwrap();
    println!("{:?}", data.header());
}
pub fn register_module(kernel: &[u8]) {
    let k = xmas_elf::ElfFile::new(kernel).unwrap();
    let symbols = SYMBOL_TABLE.get();
    for s in k.section_iter() {
        match s.get_name(&k) {
            Ok(".eh_frame") => {
                // yay!
                // addr is s.address()
                // len is s.size()
                dprintln!("Got framez");
                unsafe {
                    __register_frame(s.address() as *mut c_void, s.size() as usize);
                }
            }
            Ok(".symtab") => match s.get_data(&k).unwrap() {
                SectionData::SymbolTable64(st) => {
                    for e in st {
                        let name = e.get_name(&k).unwrap();
                        let addr = e.value();
                        symbols.insert(addr, name.to_string());
                    }
                }
                _ => {}
            },
            Ok(".debug_line") => match s.get_data(&k).unwrap() {
                SectionData::Undefined(raw_data) => {
                    line_tweaks(raw_data);
                }
                _ => {}
            },
            Ok(s) => {
                println!("S: {}", s);
            }
            _ => {}
        }
    }
}
pub fn backtrace() {
    let symbols = SYMBOL_TABLE.get();
    unsafe {
        trace(&mut |f| {
            let mut addr = ((f.ip() as u64) >> 4) << 4;
            if addr > 0xf00000000 {
                loop {
                    if addr == 0 {
                        break;
                    }
                    match symbols.get(&addr) {
                        Some(_) => {
                            break;
                        }
                        None => {
                            addr -= 1;
                        }
                    }
                }
            }
            ksymmap::addr_fmt(
                f.ip() as u64,
                match symbols.get(&addr) {
                    Some(sym) => Some(basic_unmangle(sym.clone())),
                    None => None,
                },
            );

            true
        });
    }
}

pub enum Frame {
    Raw(*mut uw::_Unwind_Context),
    Cloned {
        ip: *mut c_void,
        sp: *mut c_void,
        symbol_address: *mut c_void,
    },
}

// With a raw libunwind pointer it should only ever be access in a readonly
// threadsafe fashion, so it's `Sync`. When sending to other threads via `Clone`
// we always switch to a version which doesn't retain interior pointers, so we
// should be `Send` as well.
unsafe impl Send for Frame {}
unsafe impl Sync for Frame {}

impl Frame {
    pub fn ip(&self) -> *mut c_void {
        let ctx = match *self {
            Frame::Raw(ctx) => ctx,
            Frame::Cloned { ip, .. } => return ip,
        };
        unsafe { uw::_Unwind_GetIP(ctx) as *mut c_void }
    }

    pub fn sp(&self) -> *mut c_void {
        match *self {
            Frame::Raw(ctx) => unsafe { uw::get_sp(ctx) as *mut c_void },
            Frame::Cloned { sp, .. } => sp,
        }
    }

    pub fn symbol_address(&self) -> *mut c_void {
        if let Frame::Cloned { symbol_address, .. } = *self {
            return symbol_address;
        }
        unsafe { uw::_Unwind_FindEnclosingFunction(self.ip()) }
    }

    pub fn module_base_address(&self) -> Option<*mut c_void> {
        None
    }
}

impl Clone for Frame {
    fn clone(&self) -> Frame {
        Frame::Cloned {
            ip: self.ip(),
            sp: self.sp(),
            symbol_address: self.symbol_address(),
        }
    }
}

#[inline(always)]
pub unsafe fn trace(mut cb: &mut dyn FnMut(&Frame) -> bool) {
    uw::_Unwind_Backtrace(trace_fn, &mut cb as *mut _ as *mut _);

    extern "C" fn trace_fn(
        ctx: *mut uw::_Unwind_Context,
        arg: *mut c_void,
    ) -> uw::_Unwind_Reason_Code {
        let cb = unsafe { &mut *(arg as *mut &mut dyn FnMut(&Frame) -> bool) };
        let cx = Frame::Raw(ctx);

        let keep_going = cb(&cx);

        if keep_going {
            uw::_URC_NO_REASON
        } else {
            uw::_URC_FAILURE
        }
    }
}

/// Unwind library interface used for backtraces
///
/// Note that dead code is allowed as here are just bindings
/// iOS doesn't use all of them it but adding more
/// platform-specific configs pollutes the code too much
#[allow(non_camel_case_types)]
#[allow(non_snake_case)]
#[allow(dead_code)]
mod uw {
    pub use self::_Unwind_Reason_Code::*;

    use core::ffi::c_void;

    #[repr(C)]
    pub enum _Unwind_Reason_Code {
        _URC_NO_REASON = 0,
        _URC_FOREIGN_EXCEPTION_CAUGHT = 1,
        _URC_FATAL_PHASE2_ERROR = 2,
        _URC_FATAL_PHASE1_ERROR = 3,
        _URC_NORMAL_STOP = 4,
        _URC_END_OF_STACK = 5,
        _URC_HANDLER_FOUND = 6,
        _URC_INSTALL_CONTEXT = 7,
        _URC_CONTINUE_UNWIND = 8,
        _URC_FAILURE = 9, // used only by ARM EABI
    }

    pub enum _Unwind_Context {}

    pub type _Unwind_Trace_Fn =
        extern "C" fn(ctx: *mut _Unwind_Context, arg: *mut c_void) -> _Unwind_Reason_Code;

    extern "C" {
        pub fn _Unwind_Backtrace(
            trace: _Unwind_Trace_Fn,
            trace_argument: *mut c_void,
        ) -> _Unwind_Reason_Code;

        // available since GCC 4.2.0, should be fine for our purpose
        #[cfg(all(
            not(all(target_os = "android", target_arch = "arm")),
            not(all(target_os = "freebsd", target_arch = "arm")),
            not(all(target_os = "linux", target_arch = "arm"))
        ))]
        pub fn _Unwind_GetIP(ctx: *mut _Unwind_Context) -> usize;

        #[cfg(all(
            not(all(target_os = "android", target_arch = "arm")),
            not(all(target_os = "freebsd", target_arch = "arm")),
            not(all(target_os = "linux", target_arch = "arm"))
        ))]
        pub fn _Unwind_FindEnclosingFunction(pc: *mut c_void) -> *mut c_void;

        #[cfg(all(
            not(all(target_os = "android", target_arch = "arm")),
            not(all(target_os = "freebsd", target_arch = "arm")),
            not(all(target_os = "linux", target_arch = "arm"))
        ))]
        // This function is a misnomer: rather than getting this frame's
        // Canonical Frame Address (aka the caller frame's SP) it
        // returns this frame's SP.
        //
        // https://github.com/libunwind/libunwind/blob/d32956507cf29d9b1a98a8bce53c78623908f4fe/src/unwind/GetCFA.c#L28-L35
        #[link_name = "_Unwind_GetCFA"]
        pub fn get_sp(ctx: *mut _Unwind_Context) -> usize;
    }
}

```
### ./kernel/src/devices/mice.rs
```rust
pub trait Mouse {
    fn get_x(&self) -> u32;
    fn get_y(&self) -> u32;
}

```
### ./kernel/src/devices/mice_ps2.rs
```rust
use crate::prelude::*;
#[derive(Copy, Clone)]
struct PS2MouseInternals {
    x: u32,
    y: u32,
}
_ezy_static! { PS2_MOUSE_INTERNALS_INSTANCE, PS2MouseInternals, PS2MouseInternals { x: 0, y: 0} }

#[derive(Debug)]
pub struct PS2Mouse;
impl PS2Mouse {
    pub fn init(&self) {}
}
impl crate::devices::mice::Mouse for PS2Mouse {
    fn get_x(&self) -> u32 {
        return PS2_MOUSE_INTERNALS_INSTANCE.get().x;
    }
    fn get_y(&self) -> u32 {
        return PS2_MOUSE_INTERNALS_INSTANCE.get().y;
    }
}
static STATE: AtomicU8 = AtomicU8::new(0);
lazy_static! {
    static ref MOUSE_DATA: Mutex<Vec<u8>> = Mutex::new(vec![0, 0, 0]);
}
pub fn handle_mouse_interrupt() {
    let da = MOUSE_DATA.get();
    match STATE.load(Ordering::Relaxed) {
        0 => {
            STATE.store(1, Ordering::Relaxed);
            da[0] = unsafe { inb(0x60) };
        }
        1 => {
            STATE.store(2, Ordering::Relaxed);
            da[1] = unsafe { inb(0x60) };
        }
        2 => {
            STATE.store(0, Ordering::Relaxed);
            da[2] = unsafe { inb(0x60) };
            let mut ii = PS2_MOUSE_INTERNALS_INSTANCE.get();
            ii.x = (ii.x as i32 + (da[0] as i8) as i32) as u32;
            ii.y = (ii.y as i32 + (da[1] as i8) as i32) as u32;
        }
        _ => unreachable!(),
    }
}

```
### ./kernel/src/devices/mod.rs
```rust
pub mod mice;
pub mod mice_ps2;

```
### ./kernel/src/drive/cpio.rs
```rust
use crate::{dbg, print, println};
use alloc::{
    string::{String, ToString},
    vec,
    vec::Vec,
};
#[derive(Debug, Clone)]
pub struct CPIOEntry {
    pub magic: u64,
    pub dev: u64,
    pub ino: u64,
    pub mode: u64,
    pub uid: u64,
    pub gid: u64,
    pub nlink: u64,
    pub rdev: u64,
    pub mtime: u64,
    pub namesize: u64,
    pub filesize: u64,
    pub name: String,
    pub data: Vec<u8>,
}

pub fn chars2int(c: &[u8]) -> u64 {
    let mut res = 0u64;
    for i in c {
        if *i < 48 {
            res *= 8;
            continue;
        }
        res *= 8;
        res += (*i as u64) - 48;
    }
    res
}

pub fn parse_one(drv: &mut crate::drive::Offreader) -> Result<CPIOEntry, String> {
    let magic = chars2int(drv.read_consume(6)?.as_slice());
    let dev = chars2int(drv.read_consume(6)?.as_slice());
    let ino = chars2int(drv.read_consume(6)?.as_slice());
    let mode = chars2int(drv.read_consume(6)?.as_slice());
    let uid = chars2int(drv.read_consume(6)?.as_slice());
    let gid = chars2int(drv.read_consume(6)?.as_slice());
    let nlink = chars2int(drv.read_consume(6)?.as_slice());
    let rdev = chars2int(drv.read_consume(6)?.as_slice());
    let mtime = chars2int(drv.read_consume(11)?.as_slice());
    let nd = drv.read_consume(6)?;
    let nds = nd.as_slice();
    let namesize = chars2int(nds);
    let filesize = chars2int(drv.read_consume(11)?.as_slice());
    let name = String::from_utf8(
        drv.read_consume(namesize)?
            .split_at((namesize - 1) as usize)
            .0
            .to_vec(),
    )
    .unwrap();
    let data = drv.read_consume(filesize)?;
    if name == "TRAILER!!!" {
        return Err("EOF".to_string());
    }
    Ok(CPIOEntry {
        magic,
        dev,
        ino,
        mode,
        uid,
        gid,
        nlink,
        rdev,
        mtime,
        namesize,
        filesize,
        name,
        data,
    })
}
pub fn parse(drv: &mut crate::drive::Offreader) -> Result<Vec<CPIOEntry>, String> {
    let mut rv = vec![];
    loop {
        let x = parse_one(drv);
        match x {
            Ok(o) => {
                rv.push(o);
            }
            Err(e) => {
                if rv.len() == 0 {
                    return Err(e);
                }
                return Ok(rv);
            }
        }
    }
}

```
### ./kernel/src/drive/fat.rs
```rust
use crate::prelude::*;

pub fn try_fat() {}

```
### ./kernel/src/drive/ext2.rs
```rust
use crate::prelude::*;

use bitflags::bitflags;
use drive::RODev;

#[repr(C)]
#[derive(Copy, Clone)]
pub struct BlkGrpDesc {
    blkumap: u32,
    inoumap: u32,
    inotbl: u32,
}

pub struct SuperBlock {
    pub total_number_of_inodes_in_file_system: u32,
    pub total_number_of_blocks_in_file_system: u32,
    pub number_of_blocks_reserved_for_superuser: u32,
    pub total_number_of_unallocated_blocks: u32,
    pub total_number_of_unallocated_inodes: u32,
    pub block_number_of_the_block_containing_the_superblock: u32,
    pub log2_blocksize: u32,
    pub log2_fragment_size: u32,
    pub block_per_group: u32,
    pub number_of_fragments_in_each_block_group: u32,
    pub inode_per_group: u32,
    pub last_mount_time: u32,
    pub last_written_time: u32,
    pub number_of_times_the_volume_has_been_mounted_since_its_last_consistency_check: u16,
    pub number_of_mounts_allowed_before_a_consistency_check_must_be_done: u16,
    pub ext2_signature: u16,
    pub file_system_state: u16,
    pub what_to_do_when_an_error_is_detected: u16,
    pub minor_portion_of_version: u16,
    pub posix_time_of_last_consistency_check: u32,
    pub interval: u32,
    pub operating_system_id_from_which_the_filesystem_on_this_volume_was_created: u32,
    pub major_portion_of_version: u32,
    pub user_id_that_can_use_reserved_blocks: u16,
    pub group_id_that_can_use_reserved_blocks: u16,
    pub first_non_reserved: u32,
    pub inode_sz: u16,
}

pub fn handle_super_block(data: &[u8]) -> SuperBlock {
    let total_number_of_inodes_in_file_system =
        unsafe { *(data.split_at(0).1.split_at(4).0.as_ptr() as *const u32) };
    let total_number_of_blocks_in_file_system =
        unsafe { *(data.split_at(4).1.split_at(4).0.as_ptr() as *const u32) };
    let number_of_blocks_reserved_for_superuser =
        unsafe { *(data.split_at(8).1.split_at(4).0.as_ptr() as *const u32) };
    let total_number_of_unallocated_blocks =
        unsafe { *(data.split_at(12).1.split_at(4).0.as_ptr() as *const u32) };
    let total_number_of_unallocated_inodes =
        unsafe { *(data.split_at(16).1.split_at(4).0.as_ptr() as *const u32) };
    let block_number_of_the_block_containing_the_superblock =
        unsafe { *(data.split_at(20).1.split_at(4).0.as_ptr() as *const u32) };
    let log2_blocksize = unsafe { *(data.split_at(24).1.split_at(4).0.as_ptr() as *const u32) };
    let log2_fragment_size = unsafe { *(data.split_at(28).1.split_at(4).0.as_ptr() as *const u32) };
    let number_of_blocks_in_each_block_group =
        unsafe { *(data.split_at(32).1.split_at(4).0.as_ptr() as *const u32) };
    let number_of_fragments_in_each_block_group =
        unsafe { *(data.split_at(36).1.split_at(4).0.as_ptr() as *const u32) };
    let number_of_inodes_in_each_block_group =
        unsafe { *(data.split_at(40).1.split_at(4).0.as_ptr() as *const u32) };
    let last_mount_time = unsafe { *(data.split_at(44).1.split_at(4).0.as_ptr() as *const u32) };
    let last_written_time = unsafe { *(data.split_at(48).1.split_at(4).0.as_ptr() as *const u32) };
    let number_of_times_the_volume_has_been_mounted_since_its_last_consistency_check =
        unsafe { *(data.split_at(52).1.split_at(2).0.as_ptr() as *const u16) };
    let number_of_mounts_allowed_before_a_consistency_check_must_be_done =
        unsafe { *(data.split_at(54).1.split_at(2).0.as_ptr() as *const u16) };
    let ext2_signature = unsafe { *(data.split_at(56).1.split_at(2).0.as_ptr() as *const u16) };
    let file_system_state = unsafe { *(data.split_at(58).1.split_at(2).0.as_ptr() as *const u16) };
    let what_to_do_when_an_error_is_detected =
        unsafe { *(data.split_at(60).1.split_at(2).0.as_ptr() as *const u16) };
    let minor_portion_of_version =
        unsafe { *(data.split_at(62).1.split_at(2).0.as_ptr() as *const u16) };
    let posix_time_of_last_consistency_check =
        unsafe { *(data.split_at(64).1.split_at(4).0.as_ptr() as *const u32) };
    let interval = unsafe { *(data.split_at(68).1.split_at(4).0.as_ptr() as *const u32) };
    let operating_system_id_from_which_the_filesystem_on_this_volume_was_created =
        unsafe { *(data.split_at(72).1.split_at(4).0.as_ptr() as *const u32) };
    let major_portion_of_version =
        unsafe { *(data.split_at(76).1.split_at(4).0.as_ptr() as *const u32) };
    let user_id_that_can_use_reserved_blocks =
        unsafe { *(data.split_at(80).1.split_at(2).0.as_ptr() as *const u16) };
    let group_id_that_can_use_reserved_blocks =
        unsafe { *(data.split_at(82).1.split_at(2).0.as_ptr() as *const u16) };
    let first_non_reserved = unsafe { *(data.split_at(84).1.split_at(2).0.as_ptr() as *const u32) };
    let inode_sz = unsafe { *(data.split_at(88).1.split_at(2).0.as_ptr() as *const u16) };

    let sup = SuperBlock {
        total_number_of_inodes_in_file_system,
        total_number_of_blocks_in_file_system,
        number_of_blocks_reserved_for_superuser,
        total_number_of_unallocated_blocks,
        total_number_of_unallocated_inodes,
        block_number_of_the_block_containing_the_superblock,
        log2_blocksize,
        log2_fragment_size,
        block_per_group: number_of_blocks_in_each_block_group,
        number_of_fragments_in_each_block_group,
        inode_per_group: number_of_inodes_in_each_block_group,
        last_mount_time,
        last_written_time,
        number_of_times_the_volume_has_been_mounted_since_its_last_consistency_check,
        number_of_mounts_allowed_before_a_consistency_check_must_be_done,
        ext2_signature,
        file_system_state,
        what_to_do_when_an_error_is_detected,
        minor_portion_of_version,
        posix_time_of_last_consistency_check,
        interval,
        operating_system_id_from_which_the_filesystem_on_this_volume_was_created,
        major_portion_of_version,
        user_id_that_can_use_reserved_blocks,
        group_id_that_can_use_reserved_blocks,
        first_non_reserved,
        inode_sz,
    };
    // dbg!(total_number_of_inodes_in_file_system);
    // dbg!(total_number_of_blocks_in_file_system);
    // dbg!(number_of_blocks_reserved_for_superuser);
    // dbg!(total_number_of_unallocated_blocks);
    // dbg!(total_number_of_unallocated_inodes);
    // dbg!(block_number_of_the_block_containing_the_superblock);
    // dbg!(log2_blocksize);
    // dbg!(log2_fragment_size);
    // dbg!(number_of_blocks_in_each_block_group);
    // dbg!(number_of_fragments_in_each_block_group);
    // dbg!(number_of_inodes_in_each_block_group);
    // dbg!(last_mount_time);
    // dbg!(last_written_time);
    // dbg!(number_of_times_the_volume_has_been_mounted_since_its_last_consistency_check);
    // dbg!(number_of_mounts_allowed_before_a_consistency_check_must_be_done);
    if ext2_signature != 0xef53 {
        panic!("Ext2 mount failed: not ext2");
    }
    // dbg!(file_system_state);
    // dbg!(what_to_do_when_an_error_is_detected);
    // dbg!(minor_portion_of_version);
    // dbg!(posix_time_of_last_consistency_check);
    // dbg!(interval);
    // dbg!(operating_system_id_from_which_the_filesystem_on_this_volume_was_created);
    if major_portion_of_version < 1 {
        panic!(
            "Ext2 mount failed: wrong major {}",
            major_portion_of_version
        );
    }
    // dbg!(user_id_that_can_use_reserved_blocks);
    // dbg!(group_id_that_can_use_reserved_blocks);
    return sup;
}
bitflags! {
    pub struct Ext2InodeAttr: u16 {
        const OTHER_EXECUTE = 0x001;
        const OTHER_WRITE = 0x002;
        const OTHER_READ = 0x004;
        const GROUP_EXECUTE = 0x008;
        const GROUP_WRITE = 0x010;
        const GROUP_READ = 0x020;
        const USER_EXECUTE = 0x040;
        const USER_WRITE = 0x080;
        const USER_READ = 0x100;
        const STICKY = 0x200;
        const SETGID = 0x400;
        const SETUID = 0x800;

        const FIFO = 0x1000;
        const CHARACTERDEV = 0x2000;
        const DIRECTORY = 0x4000;
        const BLOCKDEV = 0x6000;
        const REGULARFILE = 0x8000;
        const SYMLINK = 0xA000;
        const SOCKET = 0xC000;

    }
}
impl Ext2InodeAttr {
    pub fn to_str(&self) -> String {
        let mut s = "".to_string();
        if self.contains(Ext2InodeAttr::DIRECTORY) {
            s += "d";
        } else if self.contains(Ext2InodeAttr::SYMLINK) {
            s += "l";
        } else if self.contains(Ext2InodeAttr::CHARACTERDEV) {
            s += "c";
        } else if self.contains(Ext2InodeAttr::BLOCKDEV) {
            s += "b";
        } else if self.contains(Ext2InodeAttr::REGULARFILE) {
            s += "-";
        } else {
            s += "?";
        }
        if self.contains(Ext2InodeAttr::USER_READ) {
            s += "r";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::USER_WRITE) {
            s += "w";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::USER_EXECUTE) {
            s += "x";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::GROUP_READ) {
            s += "r";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::GROUP_WRITE) {
            s += "w";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::GROUP_EXECUTE) {
            s += "x";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::OTHER_READ) {
            s += "r";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::OTHER_WRITE) {
            s += "w";
        } else {
            s += "-";
        }
        if self.contains(Ext2InodeAttr::OTHER_EXECUTE) {
            s += "x";
        } else {
            s += "-";
        }

        s
    }
}
#[repr(C)]
#[derive(Copy, Clone, Debug)]
struct Inode {
    perms: u16,
    uid: u16,
    size: u32,
    atime: u32,
    ctime: u32,
    mtime: u32,
    dtime: u32,
    gid: u16,
    hlinkc: u16,
    seccused: u32,
    flags: u32,
    sysspec: u32,
    blkptr0: u32,
    blkptr1: u32,
    blkptr2: u32,
    blkptr3: u32,
    blkptr4: u32,
    blkptr5: u32,
    blkptr6: u32,
    blkptr7: u32,
    blkptr8: u32,
    blkptr9: u32,
    blkptr10: u32,
    blkptr11: u32,
    singlblkptr: u32,
    dblblkptr: u32,
    triplblkptr: u32,
}

pub fn readarea(addr: u64, len: u64, dev: &mut Box<dyn RODev>) -> Vec<u8> {
    let lba = (addr / 512) as u32;
    let extraoff = addr - ((lba as u64) * 512);
    let totallen = (len + extraoff + 511) / 512;
    let mut d: Vec<u8> = (0..totallen)
        .flat_map(|f| dev.read_from((f as u32) + lba).unwrap())
        .collect();
    let mut n = d.split_off(extraoff as usize);
    drop(n.split_off(len as usize));
    drop(d);
    n
}

pub fn get_blk_grp_desc(blkgrp: u64, dev: &mut Box<dyn RODev>) -> BlkGrpDesc {
    // BlkGrpDesc
    // let addr = (0x800 + 32 /);
    let a = readarea(0x800 + (blkgrp * 0x20), 32, dev);
    let cln = unsafe { *(a.as_ptr() as *const BlkGrpDesc) }.clone();
    drop(a);
    cln
}
fn read_blk_tbl(dev: &mut Box<dyn RODev>, sb: &SuperBlock, dat: Vec<u8>) -> Vec<u8> {
    let slc = unsafe { core::slice::from_raw_parts(dat.as_ptr() as *const u32, dat.len() >> 2) };
    let mut d: Vec<(u64, u64)> = slc
        .iter()
        .filter(|a| **a != 0)
        .map(|f| {
            (
                (f << (10 + sb.log2_blocksize)) as u64,
                1 << (10 + sb.log2_blocksize) as u64,
            )
        })
        .collect();
    let t = dev.vector_read_ranges(&mut d);
    // println!("{:?}", t);
    t
}
fn read_from_inode(ino: Inode, dev: &mut Box<dyn RODev>, sb: &SuperBlock) -> Vec<u8> {
    if ino.triplblkptr != 0 {
        let dat = readarea(
            (ino.triplblkptr as u64) << (10 + sb.log2_blocksize),
            1 << (10 + sb.log2_blocksize),
            dev,
        );
        let dat = read_blk_tbl(dev, sb, dat);
        let dat = read_blk_tbl(dev, sb, dat);
        return read_blk_tbl(dev, sb, dat);
    }
    if ino.dblblkptr != 0 {
        let dat = readarea(
            (ino.dblblkptr as u64) << (10 + sb.log2_blocksize),
            1 << (10 + sb.log2_blocksize),
            dev,
        );
        let dat = read_blk_tbl(dev, sb, dat);
        return read_blk_tbl(dev, sb, dat);
    }
    if ino.singlblkptr != 0 {
        let dat = readarea(
            (ino.singlblkptr as u64) << (10 + sb.log2_blocksize),
            1 << (10 + sb.log2_blocksize),
            dev,
        );
        println!("{:?}", dat);
        return read_blk_tbl(dev, sb, dat);
    }
    [
        ino.blkptr0 as u64,
        ino.blkptr1 as u64,
        ino.blkptr2 as u64,
        ino.blkptr3 as u64,
        ino.blkptr4 as u64,
        ino.blkptr5 as u64,
        ino.blkptr6 as u64,
        ino.blkptr7 as u64,
        ino.blkptr8 as u64,
        ino.blkptr9 as u64,
        ino.blkptr10 as u64,
        ino.blkptr11 as u64,
    ]
    .iter()
    .flat_map(|f| {
        if *f != 0 {
            readarea(
                f << (10 + sb.log2_blocksize),
                1 << (10 + sb.log2_blocksize),
                dev,
            )
        } else {
            vec![]
        }
    })
    .collect()
}

pub fn readdir(dev: &mut Box<dyn RODev>, inode: u32, sb: &SuperBlock) -> BTreeMap<String, u32> {
    let group = inode / sb.inode_per_group;
    let desc = get_blk_grp_desc(group as u64, dev);
    let inode_table = (desc.inotbl as u64) << (sb.log2_blocksize + 10);
    let data = readarea(inode_table + (0x80 * (inode as u64 - 1)), 0x80, dev);
    let ino = unsafe { *(data.as_ptr() as *const Inode) }.clone();
    drop(data);
    let f = Ext2InodeAttr::from_bits(ino.perms).unwrap();
    assert!(f.contains(Ext2InodeAttr::DIRECTORY));
    let mut v = BTreeMap::new();
    let d = read_from_inode(ino, dev, sb);
    let mut doff = 0;
    loop {
        if doff == 1024 {
            break;
        }
        let d: Vec<u8> = d
            .split_at(doff)
            .1
            .split_at(0x108)
            .0
            .iter()
            .map(|f| *f)
            .collect();
        let ino2 = unsafe { *(d.as_ptr() as *mut u32) };
        if ino2 == 0 {
            break;
        }
        let len = unsafe { *(d.as_ptr().offset(4) as *mut u16) };
        doff += len as usize;
        let fnmlen = unsafe { *(d.as_ptr().offset(6) as *mut u8) } as usize;
        let d = String::from_utf8(
            d.split_at(8)
                .1
                .split_at(fnmlen)
                .0
                .iter()
                .map(|f| *f)
                .collect::<Vec<u8>>(),
        )
        .unwrap();
        v.insert(d, ino2);
    }
    drop(d);
    v
}
pub fn cat(dev: &mut Box<dyn RODev>, inode: u32, sb: &SuperBlock) -> Vec<u8> {
    let group = inode / sb.inode_per_group;
    let desc = get_blk_grp_desc(group as u64, dev);
    let inode_table = (desc.inotbl as u64) << (sb.log2_blocksize + 10);
    let data = readarea(inode_table + (0x80 * (inode as u64 - 1)), 0x80, dev);
    let ino = unsafe { *(data.as_ptr() as *const Inode) }.clone();
    drop(data);

    let f = Ext2InodeAttr::from_bits(ino.perms).unwrap();
    assert!(f.contains(Ext2InodeAttr::REGULARFILE) || f.contains(Ext2InodeAttr::SYMLINK));
    let mut z = read_from_inode(ino, dev, sb);
    
    z.truncate(ino.size as usize);
    z
}
pub fn stat(dev: &mut Box<dyn RODev>, inode: u32, sb: &SuperBlock) -> Ext2InodeAttr {
    let group = inode / sb.inode_per_group;
    let desc = get_blk_grp_desc(group as u64, dev);
    let inode_table = (desc.inotbl as u64) << (sb.log2_blocksize + 10);
    let data = readarea(inode_table + (0x80 * (inode as u64 - 1)), 0x80, dev);
    let ino = unsafe { *(data.as_ptr() as *const Inode) }.clone();
    drop(data);
    Ext2InodeAttr::from_bits(ino.perms).unwrap()
}
pub fn tree(dev: &mut Box<dyn RODev>, inode: u32, sb: &SuperBlock, s: String) {
    let p = readdir(dev, inode, sb);
    let d: Vec<(&String, &u32)> = p.iter().collect();
    let mut c = 0;
    let dlen = d.len();
    for (nm, ino) in d {
        if nm.starts_with(".") {
            continue;
        }
        let flags = stat(dev, *ino, sb);
        if c + 1 == dlen {
            println!("{}\\- {} ({})", s.clone(), nm, flags.to_str());
        } else {
            println!("{}+- {} ({})", s.clone(), nm, flags.to_str());
        }

        if flags.contains(Ext2InodeAttr::DIRECTORY) && !nm.starts_with(".") {
            tree(dev, *ino, sb, s.clone() + " ");
        }
        c += 1;
    }
}
pub fn handle_rodev_with_ext2(mut dev: Box<dyn RODev>) {
    let b: Vec<u8> = vec![dev.read_from(2).unwrap(), dev.read_from(3).unwrap()]
        .into_iter()
        .flat_map(|f| f)
        .collect();
    let sup = handle_super_block(b.as_slice());
    tree(&mut dev, 2, &sup, " ".to_string());
}

pub fn traverse_fs_tree(dev: &mut Box<dyn RODev>, sb: &SuperBlock, path_elems: Vec<String>) -> u32 {
    let mut inode = 2;
    for e in path_elems {
        inode = *readdir(dev, inode, sb).get(&e).unwrap();
    }

    inode
}

```
### ./kernel/src/drive/gpt.rs
```rust
use crate::prelude::*;

use drive::{RODev, ReadOp};

pub struct GPTPart {
    name: String,
    drive: Box<dyn RODev>,
    guid: String,
    slba: u32,
    sz: u32,
}
impl GPTPart {
    pub fn name(&self) -> String {
        return self.name.clone();
    }
    pub fn guid(&self) -> String {
        return self.guid.clone();
    }
    pub fn slba(&self) -> u32 {
        return self.slba;
    }
    pub fn sz(&self) -> u32 {
        return self.sz;
    }
    fn map_va(&self, a: u64) -> Result<u64, String> {
        if (self.sz * 512) as u64 <= a {
            return Err(format!("GPT fault: OOB read ({} > {})", a, self.sz * 512));
        }
        Ok(a + (self.slba * 512) as u64)
    }
}
impl RODev for GPTPart {
    fn read_from(&mut self, lba: u32) -> Result<Vec<u8>, String> {
        if self.sz <= lba {
            return Err(format!("GPT fault: OOB read ({} > {})", lba, self.sz));
        }
        return self.drive.read_from(lba + self.slba);
    }
    fn read_unaligned(&mut self, addr: u64, len: u64) -> Result<Vec<u8>, String> {
        self.drive.read_unaligned(self.map_va(addr)?, len)
    }
    fn vector_read_ranges(&mut self, ops: &mut [(u64, u64)]) -> Vec<u8> {
        let mut ops: Vec<(u64, u64)> = ops.into_iter().map(|p| (self.map_va(p.0).unwrap(), p.1)).collect();
        let q = self.drive.vector_read_ranges(&mut ops);
        if q.len() == 0 {
            panic!("RF");
        }
        q
    }
}
pub trait GetGPTPartitions {
    fn get_gpt_partitions(
        &mut self,
        clone_rodev: Box<dyn Fn<(), Output = Box<dyn RODev>>>,
    ) -> Vec<GPTPart>;
}
impl GetGPTPartitions for dyn RODev {
    fn get_gpt_partitions(
        &mut self,
        clone_rodev: Box<dyn Fn<(), Output = Box<dyn RODev>>>,
    ) -> Vec<GPTPart> {
        let mut p = vec![];
        let a = self.read_from(1).unwrap();
        let efi_magic = String::from_utf8(a.split_at(8).0.to_vec()).unwrap();
        if efi_magic != "EFI PART" {
            panic!("Invalid efi drive");
        }
        let startlba = unsafe { *(a.as_ptr().offset(0x48) as *mut u64) };
        let partc = unsafe { *(a.as_ptr().offset(0x50) as *mut u32) };
        let mut data = vec![];
        for i in 0..(((partc as u32) + 3) / 4) {
            let dat = self.read_from(i + (startlba as u32)).unwrap();
            for de in dat {
                data.push(de);
            }
        }
        // 0x0	16	Partition Type GUID (zero means unused entry)
        // 0x10	16	Unique Partition GUID
        // 0x20	8	StartingLBA
        // 0x28	8	EndingLBA
        // 0x30	8	Attributes
        // 0x38	72	Partition Name

        for part in 0..(partc) {
            let startoff = 0x80 * part;
            let data = data.split_at(startoff as usize).1;
            let parttype = prntguuid(data.split_at(16).0);
            let slba = unsafe { *(data.split_at(0x20).1.as_ptr() as *const u64) };
            let elba = unsafe { *(data.split_at(0x28).1.as_ptr() as *const u64) };
            let partid = prntguuid(data.split_at(16).1.split_at(16).0);
            let partname =
                String::from_utf8(data.split_at(0x38).1.split_at(72).0.to_vec()).unwrap();
            if parttype == "00000000-0000-0000-000000000000" {
                continue;
            }
            p.push(GPTPart {
                name: partname,
                guid: partid,
                drive: clone_rodev(),
                slba: slba as u32,
                sz: (elba - slba + 1) as u32,
            })
        }
        p
    }
}

pub fn test0() {
    let mut d = drive::SickCustomDev {};
    let d2: &mut dyn RODev = &mut d;
    let p = d2.get_gpt_partitions(box (|| box (drive::SickCustomDev {})));
    for pa in p {
        println!("{}", pa.name());
        println!("{}", pa.guid());
        println!("{}", pa.slba());

        drive::ext2::handle_rodev_with_ext2(box pa);
    }
}
pub fn prntguuid(g: &[u8]) -> String {
    // 8 4 4 12

    let h = hex::encode(g.split_at(16).0);
    let (s1, h) = h.split_at(8);
    let (s2, h) = h.split_at(4);
    let (s3, h) = h.split_at(4);
    let (s4, _h) = h.split_at(12);
    return "".to_string() + s1 + "-" + s2 + "-" + s3 + "-" + s4;
}

```
### ./kernel/src/drive/mod.rs
```rust
use crate::prelude::*;
use queue::ArrayQueue;
use core::convert::TryInto;
use x86_64::{
    instructions::port::{Port, PortReadOnly, PortWriteOnly},
    VirtAddr,
};
pub mod cpio;
pub mod ext2;
pub mod fat;
pub mod gpt;
#[derive(Debug)]
pub struct Offreader {
    drive: Drive,
    offlba: u32,
    queue: ArrayQueue<u8>,
}

#[derive(Debug, Clone)]
pub struct ReadOp {
    pub data: Vec<u8>,
    pub from: u64,
}

pub trait RODev {
    fn read_from(&mut self, lba: u32) -> Result<Vec<u8>, String>;
    fn read_unaligned(&mut self, addr: u64, len: u64) -> Result<Vec<u8>, String> {
        let mut q = self.read_from((addr / 512).try_into().unwrap())?;
        for i in 0..((len + 511) / 512) {
            q.extend(self.read_from(((addr / 512) + 1 + i).try_into().unwrap())?);
        }

        let mut q = q.split_off((addr % 512).try_into().unwrap());
        q.truncate(len as usize);
        Ok(q)
    }
    fn vector_read_ranges(&mut self, ops: &mut [(u64, u64)]) -> Vec<u8> {
        let mut p = vec![];
        for op in ops {
            p.extend(self.read_unaligned(op.0, op.1).unwrap());
        }
        p
    }
}

#[derive(Debug, Clone)]
pub struct Drive {
    data: Port<u16>,
    error: PortReadOnly<u8>,
    features: PortWriteOnly<u8>,
    sec_count: Port<u8>,
    lba_low: Port<u8>,
    lba_mid: Port<u8>,
    lba_high: Port<u8>,
    dhr: Port<u8>,
    status: PortReadOnly<u8>,
    command: PortWriteOnly<u8>,
    is_slave: bool,
}
impl Drive {
    pub unsafe fn new(is_slave: bool, base: u16, _base2: u16) -> Drive {
        Drive {
            data: Port::new(base),
            error: PortReadOnly::new(base + 1),
            features: PortWriteOnly::new(base + 1),
            sec_count: Port::new(base + 2),
            lba_low: Port::new(base + 3),
            lba_mid: Port::new(base + 4),
            lba_high: Port::new(base + 5),
            dhr: Port::new(base + 6),
            status: PortReadOnly::new(base + 7),
            command: PortWriteOnly::new(base + 7),
            // alt_status: PortReadOnly::new(base2),
            is_slave,
        }
    }
    pub fn get_offreader(&self) -> Offreader {
        Offreader {
            drive: self.clone(),
            offlba: 0,
            queue: ArrayQueue::new(1024),
        }
    }
}

impl RODev for Drive {
    fn read_from(&mut self, lba: u32) -> Result<Vec<u8>, String> {
        let mut vec = Vec::new();
        vec.reserve_exact(512);
        unsafe { vec.set_len(512) };
        unsafe {
            self.dhr.write(
                (0xE0 | ((lba >> 24) & 0x0F) | {
                    if self.is_slave {
                        0x10
                    } else {
                        0x00
                    }
                }) as u8,
            );
            self.features.write(0x00);
            self.sec_count.write(1);
            self.lba_low.write(lba as u8);
            self.lba_mid.write((lba >> 8) as u8);
            self.lba_high.write((lba >> 16) as u8);
            self.command.write(0x20);
        }
        while unsafe {
            let x: u8 = self.status.read();
            x & 0x8 != 0x8
        } {
            if unsafe { self.status.read() } & 1 == 1 {
                return Err(
                    "Drive read error: ".to_string() + &unsafe { self.error.read() }.to_string()
                );
            }
        }
        for i in 0..256 {
            unsafe {
                let val = self.data.read();
                let hi = (val >> 8) as u8;
                let lo = (val & 0xff) as u8;
                vec[i * 2 + 0] = lo;
                vec[i * 2 + 1] = hi;
            }
        }
        Ok(vec)
    }
}

pub struct SickCustomDev {}
#[repr(C)]
#[derive(Debug, Clone)]
struct EDRPType {
    addr: u64,
    len_or_count: u64,
    off: u64,
    isread: u64,
}

static mut EDRP: EDRPType = EDRPType {
    addr: 0,
    len_or_count: 0,
    off: 0,
    isread: 0,
};
impl SickCustomDev {
    fn edrp_do_read() {
        let val = unsafe { &EDRP as *const EDRPType as u64 };
        assert!(
            val < (1 << 32),
            "EDRP is above 4GiB (is the kernel too large?)"
        );
        let val = val as u32;
        // do the read!
        unsafe {
            x86::io::outl(0xff00, val);
        }
    }
    fn edrp_do_vectored() {
        let val = unsafe { &EDRP as *const EDRPType as u64 };
        assert!(
            val < (1 << 32),
            "EDRP is above 4GiB (is the kernel too large?)"
        );
        let val = val as u32;
        // do the read!
        unsafe {
            x86::io::outl(0xff04, val);
        }
    }
}
impl RODev for SickCustomDev {
    fn read_from(&mut self, lba: u32) -> Result<Vec<u8>, String> {
        let edv = (lba as u64) * 512;
        let re = unsafe { &mut EDRP };
        let p: Vec<u8> = vec![0; 512];
        re.addr = memory::convpc(p.as_ptr());
        re.isread = 1;
        re.len_or_count = 512;
        re.off = edv;
        SickCustomDev::edrp_do_read();
        Ok(p)
    }
    fn read_unaligned(&mut self, addr: u64, len: u64) -> Result<Vec<u8>, String> {
        let edv = addr;
        let re = unsafe { &mut EDRP };
        let p: Vec<u8> = [0u8].repeat(len as usize);
        re.addr = memory::convpc(p.as_ptr());
        re.isread = 1;
        re.len_or_count = len;
        re.off = edv;
        if addr == 0x80c7200 {
            loop {}
        }
        SickCustomDev::edrp_do_read();
        Ok(p)
    }
    fn vector_read_ranges(&mut self, ops: &mut [(u64, u64)]) -> Vec<u8> {
        let edrp = unsafe { &mut EDRP };
        let mem = [0u8].repeat(ops.iter().map(|a| a.1).sum::<u64>() as usize);
        let mut sztot = 0;
        let mut arrz: Vec<u64> = vec![];
        let mut kepe: Vec<Box<EDRPType>> = vec![];
        for op in ops {
            let addr = memory::convpc(unsafe { mem.as_ptr().offset(sztot) });
            let q = box EDRPType {
                addr,
                len_or_count: op.1,
                off: op.0,
                isread: 1,
            };
            arrz.push(memory::convpc(q.as_ref() as *const EDRPType));
            kepe.push(q);
            sztot += op.1 as isize;
        }
        //phptr<EDRP>[]
        edrp.addr = memory::convpc(arrz.as_ptr());
        edrp.len_or_count = arrz.len() as u64;
        SickCustomDev::edrp_do_vectored();
        drop(kepe);

        mem
    }
    // fn vector_read_ranges(&mut self, ops: &mut [(u64, u64)]) -> Vec<u8> {
    //     let mem = [0u8].repeat(ops.iter().map(|a| a.1).sum::<u64>() as usize);
    //     let mut q = vec![];
    //     for op in ops {
    //         q.push(box EDRPType {
    //             addr: memory::translate(VirtAddr::new(op.data.as_ptr() as u64))
    //                 .unwrap()
    //                 .as_u64(),
    //             len_or_count: op.data.len() as u64,
    //             off: op.from,
    //             isread: 1,
    //         })
    //     }
    //     let mut dat = [0u64].repeat(q.len());
    // //     let mut i = 0;
    // //     println!("{:?}", q);
    // //     for z in q {
    // //         dat[i] = memory::translate(VirtAddr::new(z.as_ref() as *const EDRPType as u64))
    // //             .unwrap()
    // //             .as_u64();
    // //         println!(" + {:#x?}", dat[i]);
    // //         i += 1;
    // //     }
    //     let val = unsafe { &mut EDRP };
    //     val.addr = memory::translate(VirtAddr::new(dat.as_ptr() as u64))
    //         .unwrap()
    //         .as_u64();
    //     val.len_or_count = dat.len() as u64;
    //     SickCustomDev::edrp_do_vectored();
    //     return mem;
    // }
}

impl Offreader {
    pub fn offset(&self, off: u32) -> Result<Offreader, String> {
        let mut nor = Offreader {
            drive: self.drive.clone(),
            offlba: self.offlba,
            queue: self.queue.clone(),
        };
        for _i in 0..off {
            if nor.queue.count == 0 {
                nor.read_sector()?
            }
            nor.queue.pop();
        }
        Ok(nor)
    }
    fn read_sector(&mut self) -> Result<(), String> {
        let data = self.drive.read_from(self.offlba)?;
        self.offlba += 1;
        for i in 0..512 {
            self.queue.push(data[i]);
        }
        Ok(())
    }
    pub fn read_consume(&mut self, n: u64) -> Result<Vec<u8>, String> {
        let mut o = vec![];
        for _i in 0..n {
            if self.queue.count == 0 {
                self.read_sector()?;
            }
            let e = self.queue.pop();
            o.push(*e);
        }
        Ok(o)
    }
}

```
### ./kernel/src/events/mod.rs
```rust
use crate::prelude::*;
use alloc::rc::Rc;

#[derive(Debug, Clone, Eq, PartialEq)]
pub struct EventListenerID {
    id: u32,
    name: String,
}
#[derive(Clone, Ord, PartialOrd, Eq, PartialEq, Copy)]
pub struct EventListenerIDInterface {
    id: u32,
}
impl EventListenerIDInterface {
    pub fn get_name(&self) -> String {
        match EVENT_LISTENERS.get().get(&EventListenerID {
            id: self.id,
            name: "<unknown>".to_string(),
        }) {
            Some(listener) => listener.id.name.clone(),
            None => "<unknown>".to_string(),
        }
    }
    pub fn emit(&self, event_name: String) {
        let listeners = EVENT_LISTENERS.get();
        let listener = listeners
            .get(&EventListenerID {
                id: self.id,
                name: "<unknown>".to_string(),
            })
            .expect("Invalid EventListenerIDInterface");
        match listener.events.get(&event_name) {
            Some(listeners) => {
                for listener in listeners {
                    listener();
                }
            }
            None => {}
        }
    }
    pub fn listen(&self, event_name: String, fcn: Box<dyn SyncFn()>) {
        let listeners = EVENT_LISTENERS.get();
        let listener = listeners
            .get_mut(&EventListenerID {
                id: self.id,
                name: "<unknown>".to_string(),
            })
            .expect("Invalid EventListenerIDInterface");
        match listener.events.get_mut(&event_name) {
            Some(listeners) => {
                listeners.push_back(fcn);
            }
            None => {
                let mut list = LinkedList::new();
                list.push_back(fcn);
                listener.events.insert(event_name, list);
            }
        }
    }
}
impl core::fmt::Debug for EventListenerIDInterface {
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        let ll = EVENT_LISTENERS.get();
        let v = ll.get(&EventListenerID {
            id: 0,
            name: "<unknown>".to_string(),
        });
        match v {
            Some(v) => write!(f, "[EventEmitter: {} (id: {})]", v.id.name, v.id.id),
            None => write!(f, "[EventEmitter: <invalid> (id: {})]", self.id),
        }
    }
}
impl core::cmp::PartialOrd for EventListenerID {
    fn partial_cmp(&self, other: &Self) -> Option<core::cmp::Ordering> {
        return self.id.partial_cmp(&other.id);
    }
}
impl core::cmp::Ord for EventListenerID {
    fn cmp(&self, other: &Self) -> core::cmp::Ordering {
        return self.id.cmp(&other.id);
    }
}

pub trait SyncFn<T>: Sync + Send + Fn<T> {}
pub struct EventListener {
    events: BTreeMap<String, LinkedList<Box<dyn SyncFn()>>>,
    id: EventListenerID,
}

ezy_static! { EVENT_LISTENERS, BTreeMap<EventListenerID, EventListener>, BTreeMap::new() }
ezy_static! { EVENT_NAME_TO_ID, BTreeMap<String, u32>, BTreeMap::new() }
counter!(ListenerId);
pub fn create_listener(name: String, public: bool) -> EventListenerIDInterface {
    let id = ListenerId.inc();
    let id = id as u32;
    let idstruct = EventListenerID {
        id,
        name: name.clone(),
    };
    let l = EventListener {
        events: BTreeMap::new(),
        id: idstruct.clone(),
    };
    EVENT_LISTENERS.get().insert(idstruct, l);
    if public {
        EVENT_NAME_TO_ID.get().insert(name, id);
    }
    EventListenerIDInterface { id }
}
pub fn get_id_by_name(name: String) -> Option<EventListenerIDInterface> {
    match EVENT_NAME_TO_ID.get().get(&name) {
        Some(val) => Some(EventListenerIDInterface { id: *val }),
        None => None,
    }
}

```
### ./kernel/src/exiting.rs
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

pub fn exit_qemu(exit_code: QemuExitCode) -> ! {
    unsafe {
        x86::io::outl(0xf4, exit_code as u32);
    }
    halt();
}

pub fn halt() -> ! {
    loop {
        x86_64::instructions::hlt();
    }
}

pub fn exit() -> ! {
    if super::constants::should_fini_exit() {
        exit_qemu(QemuExitCode::Success);
    }
    if super::constants::should_fini_wait() {
        halt();
    }
    panic!("The constants are invalid")
}
pub fn exit_fail() -> ! {
    if super::constants::should_fini_exit() {
        exit_qemu(QemuExitCode::Failed);
    }
    if super::constants::should_fini_wait() {
        halt();
    }
    panic!("The constants are invalid")
}

```
### ./kernel/src/init/mod.rs
```rust
use core::ffi::c_void;

use x86_64::{
    instructions::{
        segmentation::set_cs,
        segmentation::{load_ds, load_fs, load_gs, load_ss},
        tables::load_tss,
    },
    registers::{
        model_specific::Star,
        model_specific::{Efer, EferFlags, LStar, SFMask},
        rflags::RFlags,
    },
};

use crate::prelude::*;
// handle init

fn run_task<T: Fn()>(t: &str, f: T) {
    println!("[kinit] task {}", t);
    f();
}
fn gdt() {
    interrupts::GDT.0.load();
    unsafe {
        set_cs(interrupts::GDT.1.code_selector);
        load_ds(interrupts::GDT.1.data_selector);
        load_fs(interrupts::GDT.1.data_selector);
        load_gs(interrupts::GDT.1.data_selector);
        load_ss(interrupts::GDT.1.data_selector);
        load_tss(interrupts::GDT.1.tss_selector);
    }
}
pub fn init(boot_info: &'static multiboot2::BootInformation) {
    println!("[kinit] Setting up Oh Es");
    println!("[kinit] [mman] initializing...");
    unsafe {
        *memory::FRAME_ALLOC.get() = Some(memory::BootInfoFrameAllocator::init(
            &boot_info.memory_map_tag().unwrap(),
        ));
    }
    println!("[kinit] [mman] we have frame allocation!");

    memory::allocator::init_heap().expect("Heap init failed");
    println!("[kinit] [mman] heap ready.");
    run_task("idt", || {
        interrupts::init_idt();
    });
    run_task("pit", || {
        interrupts::init_timer(1);
    });
    run_task("gdt", gdt);
    let b = box 3;
    println!("{}", b);
    run_task("io.general", || {
        io::proper_init_for_iodevs(boot_info);
    });
    run_task("status", || {
        println!("We have a liftoff! Internal kernel mman done.");
        let c = x86::cpuid::CpuId::new();
        println!(
            "Booting Oh Es on {}",
            c.get_vendor_info().unwrap().as_string()
        );
        let r1 = x86::cpuid::cpuid!(0x80000002);
        let r2 = x86::cpuid::cpuid!(0x80000003);
        let r3 = x86::cpuid::cpuid!(0x80000004);
        let mut bytes = vec![];
        bytes.push(r1.eax);
        bytes.push(r1.ebx);
        bytes.push(r1.ecx);
        bytes.push(r1.edx);
        bytes.push(r2.eax);
        bytes.push(r2.ebx);
        bytes.push(r2.ecx);
        bytes.push(r2.edx);
        bytes.push(r3.eax);
        bytes.push(r3.ebx);
        bytes.push(r3.ecx);
        bytes.push(r3.edx);
        let d = unsafe { core::slice::from_raw_parts(bytes.as_mut_ptr() as *mut u8, 3 * 4 * 4) };
        println!(" + Brand is {}", String::from_utf8(d.to_vec()).unwrap());

        match c.get_hypervisor_info() {
            Some(hi) => {
                println!(" + We are on {:?}", hi.identify());
            }
            None => {}
        };
    });
    run_task("task_queue.init", || {
        preempt::TASK_QUEUE.get();
    });

    // Set up syscalls
    run_task("regs", || {
        run_task("regs.efer", || unsafe {
            Efer::update(|a| {
                *a |= EferFlags::SYSTEM_CALL_EXTENSIONS | EferFlags::NO_EXECUTE_ENABLE;
            });
        });
        run_task("regs.lstar", || {
            LStar::write(VirtAddr::from_ptr(
                crate::userland::new_syscall_trampoline as *const u8,
            ));
        });
        run_task("regs.sfmask", || {
            SFMask::write(RFlags::INTERRUPT_FLAG);
        });
        run_task("regs.star", || {
            Star::write(
                interrupts::GDT.1.usercode,
                interrupts::GDT.1.userdata,
                interrupts::GDT.1.code_selector,
                interrupts::GDT.1.data_selector,
            )
            .unwrap();
        });
    });
    run_task("io.device.kbdint", || {
        task::keyboard::KEY_QUEUE.init_once(|| crossbeam_queue::ArrayQueue::new(100));
    });

    run_task("unwind", || {

        unsafe {
            unwind::__register_frame((&memory::es) as *const u8 as *mut u8 as *mut c_void, (&memory::esz) as *const u8 as u64 as usize);
        }
    });
    run_task("ksvc", || {
        ksvc::ksvc_init();
    });
    run_task("enable_int", || {
        x86_64::instructions::interrupts::enable();
    });
}

```
### ./kernel/src/io/device/debugcon.rs
```rust
use crate::prelude::*;
use io::device::IODevice;

pub struct DebugCon {
    pub port: u16,
}
impl IODevice for DebugCon {
    fn write_str(&mut self, s: &str) {
        for c in s.chars() {
            unsafe {
                outb(self.port, c as u8);
            }
        }
    }

    fn read_chr(&mut self) -> Option<char> {
        None
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device/kbdint_input.rs
```rust
use crate::prelude::*;
use io::device::IODevice;

pub struct KbdInt {}
impl IODevice for KbdInt {
    fn write_str(&mut self, _s: &str) {}

    fn read_chr(&mut self) -> Option<char> {
        match task::keyboard::KEY_QUEUE.get().unwrap().pop() {
            Ok(k) => match k {
                pc_keyboard::DecodedKey::RawKey(_k) => None,
                pc_keyboard::DecodedKey::Unicode(c) => Some(c),
            },
            Err(_) => None,
        }
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device/multiboot_text.rs
```rust
use crate::prelude::*;
use font8x8::UnicodeFonts;
use io::device::IODevice;
use multiboot2::FramebufferField;

pub struct MultibootText {
    addr: u64,
    w: u32,
    h: u32,
    x_pos: u32,
}
#[repr(C)]
pub struct VGATextCell {
    chr: u8,
    cc: u8,
}
impl MultibootText {
    pub fn new(tag: multiboot2::FramebufferTag<'static>) -> MultibootText {
        match tag.buffer_type {
            multiboot2::FramebufferType::Indexed { palette: _ } => {
                panic!("No support for indexed FB")
            }
            multiboot2::FramebufferType::RGB {
                red: _,
                green: _,
                blue: _,
            } => panic!("Create a MultibootVGA please"),
            multiboot2::FramebufferType::Text => {
                // Text FB
            }
        };
        MultibootText {
            addr: tag.address,
            w: tag.width,
            h: tag.height,
            x_pos: 0,
        }
    }
    pub fn put_char_at(&mut self, x_pos: u32, chr: char) -> usize {
        unsafe {
            (*(self.addr as *mut VGATextCell).offset((x_pos + ((self.h - 1) * self.w)) as isize))
                .chr = chr as u8;
            (*(self.addr as *mut VGATextCell).offset((x_pos + ((self.h - 1) * self.w)) as isize))
                .cc = 0xf;
        }
        1
    }
    pub fn set_cursor_pos(&self, x: u16, y: u16) {
        let pos = y * (self.w as u16) + x;

        unsafe {
            outb(0x3D4, 0x0F);
            outb(0x3D5, (pos & 0xFF) as u8);
            outb(0x3D4, 0x0E);
            outb(0x3D5, ((pos >> 8) & 0xFF) as u8);
        }
    }
    pub fn putc(&mut self, chr: char) {
        let mut off = self.x_pos;
        off = match chr {
            '\n' => {
                self.new_line();
                0
            }
            '\x20'..'\x7f' => off + self.put_char_at(off, chr) as u32,
            '\x08' => {
                self.put_char_at(off - 1, ' ');
                off - 1
            }
            _ => off,
        };
        self.x_pos = off;
        self.update_loc();
    }
    pub fn new_line(&mut self) {
        self.scroll_up()
    }
    pub fn scroll_up(&mut self) {
        let o = self.addr as *mut u8;

        self.x_pos = 0;
        unsafe {
            faster_rlibc::fastermemcpy(
                o,
                o.offset((self.w * 2) as isize),
                (self.w * 2 * (self.h - 1)) as usize,
            );
            faster_rlibc::fastermemset(
                o.offset((self.w * 2 * (self.h - 1)) as isize),
                0,
                (self.w * 2) as usize,
            );
        }
        self.update_loc();
    }
    fn update_loc(&self) {
        self.set_cursor_pos(self.x_pos as u16, (self.h - 1) as u16);
    }
}
impl IODevice for MultibootText {
    fn write_str(&mut self, s: &str) {
        for c in s.chars() {
            self.putc(c);
        }
    }

    fn read_chr(&mut self) -> Option<char> {
        None
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device/multiboot_vga.rs
```rust
use crate::prelude::*;
use font8x8::UnicodeFonts;
use io::device::IODevice;
use multiboot2::FramebufferField;

pub struct MultibootVGA {
    addr: u64,
    w: u32,
    h: u32,
    x_pos: u32,
    y_pos: u32,
    font: fontdue::Font,
}
impl MultibootVGA {
    pub fn new(tag: multiboot2::FramebufferTag<'static>) -> MultibootVGA {
        match tag.buffer_type {
            multiboot2::FramebufferType::Indexed { palette: _ } => {
                panic!("No support for indexed FB")
            }
            multiboot2::FramebufferType::RGB {
                red: _,
                green: _,
                blue: _,
            } => {}
            multiboot2::FramebufferType::Text => {
                // Text FB
                panic!("Create a MultibootText instead");
            }
        };
        let font = include_bytes!("../../../fonts/roboto.ttf") as &[u8];
        // Parse it into the font type.
        let font = fontdue::Font::from_bytes(font, fontdue::FontSettings::default()).unwrap();

        MultibootVGA {
            addr: tag.address,
            w: tag.width,
            h: tag.height,
            x_pos: 0,
            y_pos: 0,
            font,
        }
    }
    pub fn put_pxl_at(&mut self, x: u32, y: u32, is_on: bool) {
        let off = 4 * (self.w * y + x);

        unsafe {
            if is_on {
                *(self.addr as *mut u8).offset(off as isize).offset(2) =
                    io::CR.load(Ordering::Relaxed) as u8;
                *(self.addr as *mut u8).offset(off as isize).offset(1) =
                    io::CG.load(Ordering::Relaxed) as u8;
                *(self.addr as *mut u8).offset(off as isize).offset(0) =
                    io::CB.load(Ordering::Relaxed) as u8;
            } else {
                *(self.addr as *mut u32).offset(off as isize) = 0;
            }
        };
    }
    pub fn put_char_at(&mut self, x_pos: u32, chr: char) -> usize {
        let (m, r) = self.font.rasterize(chr, 8.0);
        for x in 0..m.width {
            for y in 0..m.height {
                let v = r[y * m.width + x];
                self.put_pxl_at(x as u32 + x_pos, y as u32 + self.h - 8, v < 128);
                if x as u32 + x_pos > self.w {
                    self.new_line()
                }
            }
        }
        m.width
    }
    pub fn putc(&mut self, chr: char) {
        let mut off = self.x_pos;
        off = match chr {
            '\n' => {
                self.new_line();
                0
            }
            '\x20'..'\x7f' => off + self.put_char_at(off, chr) as u32,
            _ => off,
        };
        self.x_pos = off
    }
    pub fn new_line(&mut self) {
        self.scroll_up()
    }
    pub fn scroll_up(&mut self) {
        let o = self.addr as *mut u8;

        self.y_pos -= 16;
        self.x_pos = 0;
        unsafe {
            faster_rlibc::fastermemcpy(
                o,
                o.offset((32 * self.w) as isize),
                (self.w * 4 * (self.h - 16)) as usize,
            );
        }
    }
}
impl IODevice for MultibootVGA {
    fn write_str(&mut self, s: &str) {
        for c in s.chars() {
            // TODO: multiboot VGA
            self.put_char_at(0, c);
        }
    }

    fn read_chr(&mut self) -> Option<char> {
        None
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device/serial.rs
```rust
use crate::prelude::*;
use io::device::IODevice;

pub struct Serial {
    se: u16,
}
impl Serial {
    pub fn new(port: u16) -> Serial {
        unsafe {
            outb(port + 1, 0x00); // Disable all interrupts
            outb(port + 3, 0x80); // Enable DLAB (set baud rate divisor)
            outb(port + 0, 0x03); // Set divisor to 3 (lo byte) 38400 baud
            outb(port + 1, 0x00); //                  (hi byte)
            outb(port + 3, 0x03); // 8 bits, no parity, one stop bit
            outb(port + 2, 0xC7); // Enable FIFO, clear them, with 14-byte threshold
            outb(port + 4, 0x0B); // IRQs enabled, RTS/DSR set
        }

        Serial { se: port }
    }
}
impl IODevice for Serial {
    fn write_str(&mut self, s: &str) {
        for c in s.as_bytes() {
            while unsafe { inb(self.se + 5) & 0x20 } == 0 {}
            unsafe {
                outb(self.se, *c);
            }
        }
    }

    fn read_chr(&mut self) -> Option<char> {
        if unsafe { inb(self.se + 5) & 1 } != 0 {
            Some(unsafe { inb(self.se) } as char)
        } else {
            None
        }
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device/virt.rs
```rust
use crate::prelude::*;
pub struct Repe {
    pub s: String
}
impl io::device::IODevice for Repe {
    fn write_str(&mut self, s: &str) {
    }

    fn read_chr(&mut self) -> Option<char> {
        if self.s.len() != 0 {
            let c = self.s.chars().next().unwrap();
            self.s = self.s.split_off(1);
            Some(c)
        } else {
            None
        }
    }

    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/device.rs
```rust
use crate::prelude::*;

pub mod debugcon;
pub mod kbdint_input;
pub mod multiboot_text;
pub mod multiboot_vga;
pub mod serial;
pub mod virt;

pub trait IODevice: Send + Sync {
    fn write_str(&mut self, _s: &str) {}
    fn write_bytes(&mut self, _s: &[u8]) {}
    fn read_chr(&mut self) -> Option<char> {
        None
    }
    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

pub struct Mutlidev {
    dev: Vec<Box<dyn IODevice>>,
}
impl Mutlidev {
    pub fn new(d: Vec<Box<dyn IODevice>>) -> Mutlidev {
        Mutlidev { dev: d }
    }
    pub fn empty() -> Mutlidev {
        Mutlidev::new(vec![])
    }
}
impl IODevice for Mutlidev {
    fn write_str(&mut self, s: &str) {
        for i in self.dev.as_mut_slice() {
            i.write_str(s);
        }
    }

    fn read_chr(&mut self) -> Option<char> {
        for i in self.dev.as_mut_slice() {
            match i.read_chr() {
                Some(c) => return Some(c),
                None => {}
            }
        }
        None
    }

    // you can't seek a multidev
    fn seek(&mut self, _to: usize) -> Option<()> {
        None
    }
}

```
### ./kernel/src/io/mod.rs
```rust
use crate::constants;
use crate::shittymutex::Mutex;
use alloc::borrow::ToOwned;
use alloc::boxed::Box;
use alloc::string::{String, ToString};
use core::{
    cell::Cell,
    fmt::{Arguments, Result, Write},
};
use core::{file, line, stringify};
use core::{
    future::Future,
    sync::atomic::{AtomicUsize, Ordering},
    task::Poll,
};
use font8x8::UnicodeFonts;
use lazy_static::lazy_static;

use self::device::IODevice;

pub static X_POS: AtomicUsize = AtomicUsize::new(0);
pub static Y_POS: AtomicUsize = AtomicUsize::new(0);
pub static CR: AtomicUsize = AtomicUsize::new(255);
pub static CG: AtomicUsize = AtomicUsize::new(255);
pub static CB: AtomicUsize = AtomicUsize::new(255);

pub mod device;

pub enum MaybeInitDevice {
    GotMman(
        alloc::vec::Vec<Box<dyn device::IODevice>>,
        alloc::vec::Vec<Box<dyn device::IODevice>>,
    ),
    NoMman,
}

impl MaybeInitDevice {
    pub fn force_maybeinitdev(&mut self) -> &mut Self {
        self
    }
}

pub struct Printer;
pub struct DbgPrinter;
lazy_static! {
    pub static ref IO_DEVS: Mutex<MaybeInitDevice> = Mutex::new(MaybeInitDevice::NoMman);
}

pub fn proper_init_for_iodevs(mbstruct: &'static multiboot2::BootInformation) {
    // we got mman now! let's get i/o subsystem fixed
    // first, parse out kcmdline
    let kcmdline = mbstruct.command_line_tag().unwrap().command_line();
    let mut devs: alloc::vec::Vec<Box<dyn device::IODevice>> = alloc::vec![];
    let mut ddevs: alloc::vec::Vec<Box<dyn device::IODevice>> = alloc::vec![];
    let mut nid = false;
    for op in kcmdline.split(" ") {
        if op.starts_with("debugcon=") {
            let debugcon_addr = op.split_at(8).1.parse().unwrap();
            devs.push(box device::debugcon::DebugCon {
                port: debugcon_addr,
            });
            if nid {
                ddevs.push(box device::debugcon::DebugCon {
                    port: debugcon_addr,
                });
            }
        }
        if op.starts_with("serial=") {
            let debugcon_addr = op.split_at(7).1.parse().unwrap();
            devs.push(box device::serial::Serial::new(debugcon_addr));
            if nid {
                ddevs.push(box device::serial::Serial::new(debugcon_addr));
            }
        }
        if op.starts_with("input-txt=") {
            let dat = op.split_at(10).1;
            devs.push(box device::virt::Repe { s: dat.to_string() + "\n" });
        }
        if op == "default_serial" {
            devs.push(box device::serial::Serial::new(0x3F8));
            if nid {
                ddevs.push(box device::debugcon::DebugCon { port: 0x402 });
            }
        }
        if op == "default_debugcon" {
            devs.push(box device::debugcon::DebugCon { port: 0x402 });
            if nid {
                ddevs.push(box device::debugcon::DebugCon { port: 0x402 });
            }
        }
        if op == "textvga" {
            devs.push(box device::multiboot_text::MultibootText::new(
                mbstruct.framebuffer_tag().unwrap(),
            ));
            if nid {
                ddevs.push(box device::multiboot_text::MultibootText::new(
                    mbstruct.framebuffer_tag().unwrap(),
                ));
            }
        }
        if op == "graphicz" {
            devs.push(box device::multiboot_vga::MultibootVGA::new(
                mbstruct.framebuffer_tag().unwrap(),
            ));
            if nid {
                ddevs.push(box device::multiboot_vga::MultibootVGA::new(
                    mbstruct.framebuffer_tag().unwrap(),
                ));
            }
        }
        if op == "kbdint" {
            devs.push(box device::kbdint_input::KbdInt {});
            if nid {
                ddevs.push(box device::kbdint_input::KbdInt {});
            }
        }
        if op == "debug:" {
            nid = true;
        } else {
            nid = false;
        }
    }
    *IO_DEVS.get() = MaybeInitDevice::GotMman(devs, ddevs);
    Printer
        .write_fmt(format_args!("Done kernel commandline: {}\n", kcmdline))
        .unwrap();
    unsafe {
        log::set_logger_racy(&KLogImpl).unwrap();
    }
    crate::println!("{}", log::max_level());
    log::set_max_level(LevelFilter::Trace);
    log::info!("Test");
}

impl Printer {
    pub fn set_color(&mut self, r: u8, g: u8, b: u8) {
        CR.store(r as usize, Ordering::SeqCst);
        CG.store(g as usize, Ordering::SeqCst);
        CB.store(b as usize, Ordering::SeqCst);
        // TODO: move this to io::device::serial

        // SERIAL
        //     .get()
        //     .write_fmt(format_args!("\x1b[38;2;{};{};{}m", r, g, b))
        //     .expect("Printing to serial failed");
    }
    pub fn clear_screen(&mut self) {
        panic!("Clear Screen in legacy io printer");
    }
    pub fn scroll_up(&mut self) {
        // TODO: remove cuz autoscroll
        panic!("Scroll Up in legacy io printer");
    }

    pub fn newline(&mut self) {
        panic!("newline in legacy io printer");
    }
}

impl Write for Printer {
    fn write_str(&mut self, s: &str) -> Result {
        match IO_DEVS.get().force_maybeinitdev() {
            MaybeInitDevice::GotMman(mmaned, dbgdevs) => {
                for mm in mmaned {
                    mm.write_str(s);
                }
            }
            MaybeInitDevice::NoMman => {
                if crate::constants::should_debug_log() {
                    for c in s.chars() {
                        unsafe {
                            x86::io::outb(0x402, c as u8);
                        }
                    }
                }
            }
        }
        Ok(())
    }
}
impl Write for DbgPrinter {
    fn write_str(&mut self, s: &str) -> Result {
        match IO_DEVS.get().force_maybeinitdev() {
            MaybeInitDevice::GotMman(_mmaned, dbgdevs) => {
                for mm in dbgdevs {
                    mm.write_str(s);
                }
            }
            MaybeInitDevice::NoMman => {
                if crate::constants::should_debug_log() {
                    for c in s.chars() {
                        unsafe {
                            x86::io::outb(0x402, c as u8);
                        }
                    }
                }
            }
        }
        Ok(())
    }
}

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::print_out(format_args!($($arg)*)));
}
#[macro_export]
macro_rules! dprint {
    ($($arg:tt)*) => ($crate::io::dprint_out(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}

#[macro_export]
macro_rules! dprintln {
    () => ($crate::dprint!("\n"));
    ($($arg:tt)*) => ($crate::dprint!("{}\n", format_args!($($arg)*)));
}

#[macro_export]
macro_rules! dbg {
    () => {
        $crate::println!("[{}:{}]", file!(), line!());
    };
    ($val:expr) => {
        // Use of `match` here is intentional because it affects the lifetimes
        // of temporaries - https://stackoverflow.com/a/48732525/1063961
        match $val {
            tmp => {
                $crate::println!("[{}:{}] {} = {:#?}",
                    file!(), line!(), stringify!($val), &tmp);
                tmp
            }
        }
    };
    // Trailing comma with single argument is ignored
    ($val:expr,) => { $crate::dbg!($val) };
    ($($val:expr),+ $(,)?) => {
        ($($crate::dbg!($val)),+,)
    };
}

#[derive(Debug, PartialEq, Eq, Ord, PartialOrd, Copy, Clone)]
enum EHS {
    None,
    Bracket,
    Value,
}
pub struct ReadLineFuture<
    Ta: FnMut<(), Output = Option<String>> + Sized,
    Tb: FnMut<(), Output = Option<String>> + Sized,
    Tc: FnMut<(String,), Output = Option<String>> + Sized,
> {
    text: alloc::boxed::Box<String>,
    finito: bool,
    arrow_up: Ta,
    arrow_down: Tb,
    complete: Tc,
    escape_handling_step: EHS,
}
impl<
        Ta: FnMut<(), Output = Option<String>> + Sized + core::marker::Unpin,
        Tb: FnMut<(), Output = Option<String>> + Sized + core::marker::Unpin,
        Tc: FnMut<(String,), Output = Option<String>> + Sized + core::marker::Unpin,
    > ReadLineFuture<Ta, Tb, Tc>
{
    pub fn handle_c(&mut self, k: char) -> Poll<String> {
        if self.escape_handling_step == EHS::Bracket {
            // assert_eq!(k, '[', "Wrong escape sent over serial");
            self.escape_handling_step = EHS::Value;
            return Poll::Pending;
        }
        if self.escape_handling_step == EHS::Value {
            let result = match k {
                'A' => {
                    // arrow up
                    (self.arrow_up)()
                }
                'B' => {
                    // arrow down
                    (self.arrow_down)()
                }
                _ => None,
            };

            match result {
                Some(str) => {
                    let mut el = self.text.as_ref().to_owned();
                    while el.len() != 0 {
                        el.remove(el.len() - 1);
                        print!("\x08 \x08");
                    }
                    *self.text = str.clone();
                    print!("{}", self.text);
                }
                None => {}
            }
            self.escape_handling_step = EHS::None;
            return Poll::Pending;
        }
        if k == '\r' || k == '\n' {
            println!();
            self.finito = true;
            return Poll::Ready(String::from(self.text.as_str()));
        } else if k == '\x7f' || k == '\x08' {
            let mut el = self.text.as_ref().to_owned();
            if el.len() == 0 {
                return Poll::Pending;
            }
            el.remove(el.len() - 1);
            print!("\x08 \x08");
            *self.text = el;
            return Poll::Pending;
        } else if k == '\x1b' {
            // TODO: fix this
            self.escape_handling_step = EHS::Bracket;
            // let c = SERIAL.get().receive() as char;
            // let result = match c {
            //     'A' => {
            //         // arrow up
            //         (self_ref.arrow_up)()
            //     }
            //     'B' => {
            //         // arrow down
            //         (self_ref.arrow_down)()
            //     }
            //     _ => None,
            // };

            // match result {
            //     Some(str) => {
            //         let mut el = self_ref.text.as_ref().to_owned();
            //         while el.len() != 0 {
            //             el.remove(el.len() - 1);
            //             print!("\x08");
            //         }
            //         *self_ref.text = str.clone();
            //         print!("{}", self_ref.text);
            //     }
            //     None => {}
            // }
            return Poll::Pending;
        } else if k == '\t' {
            let result = (self.complete)(self.text.to_string());
            match result {
                Some(str) => {
                    let mut el = self.text.as_ref().to_owned();
                    while el.len() != 0 {
                        el.remove(el.len() - 1);
                        print!("\x08");
                    }
                    *self.text = str.clone();
                    print!("{}", self.text);
                }
                None => {}
            }
        } else {
            print!("{}", k);
            *self.text = self.text.as_ref().to_owned() + &String::from(k);
        }
        return Poll::Pending;
    }
}
impl<
        Ta: FnMut<(), Output = Option<String>> + Sized + core::marker::Unpin,
        Tb: FnMut<(), Output = Option<String>> + Sized + core::marker::Unpin,
        Tc: FnMut<(String,), Output = Option<String>> + Sized + core::marker::Unpin,
    > Future for ReadLineFuture<Ta, Tb, Tc>
{
    type Output = String;

    fn poll(
        self: core::pin::Pin<&mut Self>,
        _: &mut core::task::Context<'_>,
    ) -> core::task::Poll<<Self as core::future::Future>::Output> {
        let self_ref = self.get_mut();
        if self_ref.finito {
            return Poll::Ready(String::from(self_ref.text.as_str()));
        }
        let k = (|| match IO_DEVS.get().force_maybeinitdev() {
            MaybeInitDevice::GotMman(d, d2) => {
                for k in d {
                    match k.read_chr() {
                        Some(k) => {
                            return Some(k);
                        }
                        None => {}
                    }
                }
                None
            }
            MaybeInitDevice::NoMman => None,
        })();
        // println!("{:?}", k);
        match k {
            Some(k) => match self_ref.handle_c(k) {
                Poll::Ready(r) => {
                    return Poll::Ready(r);
                }
                Poll::Pending => {}
            },
            None => {}
        }

        return Poll::Pending;
    }
}
pub fn read_line<
    Ta: FnMut<(), Output = Option<String>> + Sized,
    Tb: FnMut<(), Output = Option<String>> + Sized,
    Tc: FnMut<(String,), Output = Option<String>> + Sized,
>(
    arrow_up: Ta,
    arrow_down: Tb,
    complete: Tc,
) -> ReadLineFuture<Ta, Tb, Tc> {
    ReadLineFuture {
        text: alloc::boxed::Box::from(String::from("")),
        finito: false,
        arrow_up,
        arrow_down,
        complete,
        escape_handling_step: EHS::None,
    }
}

#[macro_export]
macro_rules! input {
    () => {
        $crate::io::read_line(
            || {
                return None;
            },
            || {
                return None;
            },
            |s: String| {
                return None;
            },
        );
    };
    ($arrowup: expr, $arrowdown: expr, $complete: expr) => {
        $crate::io::read_line($arrowup, $arrowdown, $complete);
    };
}

#[doc(hidden)]
pub fn print_out(args: Arguments) {
    Printer.write_fmt(args).expect("Write failed");
}

#[doc(hidden)]
pub fn dprint_out(args: Arguments) {
    DbgPrinter.write_fmt(args).expect("Write failed");
}

use log::{Level, LevelFilter, Metadata, Record};

struct KLogImpl;

impl log::Log for KLogImpl {
    fn enabled(&self, _metadata: &Metadata) -> bool {
        true
    }

    fn log(&self, record: &Record) {
        let o = match record.level() {
            Level::Error => "\x1b[32mERROR",
            Level::Warn => "\x1b[33mWARN",
            Level::Info => "\x1b[38mINFO",
            Level::Debug => "\x1b[30mDEBUG",
            Level::Trace => "\x1b[30;2mTRACE",
        };
        println!("{} {}\x1b[0m", o, record.args());
    }

    fn flush(&self) {}
}

#[no_mangle]
pub fn out(s: &str) {
    // print!("{}", s);
}

```
### ./kernel/src/ksymmap.rs
```rust
use crate::prelude::*;
use serde::{Deserialize, Serialize};
// use serde_derive::{Deserialize, Serialize};
use serde_json::Result;

#[derive(Serialize, Deserialize)]
struct Symbol {
    addr: u64,
    line: i32,
}
enum SymFmt {
    JSON,
    Postcard,
    EfficentPostcard,
}
ezy_static! { SYMTAB, Option<BTreeMap<u64, String>>, None }
pub fn load_ksymmap(name: String, symmap: &[u8]) {
    let mut sfmt: Option<SymFmt> = None;
    if name.ends_with(".pcrd") {
        sfmt = Some(SymFmt::Postcard);
    }
    if name.ends_with(".cpost") {
        sfmt = Some(SymFmt::EfficentPostcard);
    }
    if name.ends_with(".json") {
        sfmt = Some(SymFmt::JSON);
    }
    if sfmt.is_none() {
        println!("Unknown ksymmap format");
        return;
    }
    let deser: BTreeMap<String, Vec<Symbol>> = match sfmt.unwrap() {
        SymFmt::JSON => serde_json::from_slice(&symmap).unwrap(),
        SymFmt::Postcard => postcard::from_bytes(&symmap).unwrap(),
        SymFmt::EfficentPostcard => {
            *SYMTAB.get() = postcard::from_bytes(&symmap).unwrap();
            return;
        }
    };
    let mut symtab = BTreeMap::<u64, String>::new();
    for (k, v) in deser {
        for sym in v {
            symtab.insert(sym.addr, k.clone() + &":" + &sym.line.to_string());
        }
    }
    *SYMTAB.get() = Some(symtab);
}
pub fn addr2sym(addr: u64) -> Option<String> {
    let p = &*SYMTAB.get();
    match p {
        Some(stab) => {
            let mut addr = addr;
            if addr > 0xf00000000 {
                loop {
                    if addr < 0xf00000000 {
                        break;
                    }
                    match stab.get(&addr) {
                        Some(r) => {
                            return Some(r.clone());
                        }
                        None => {
                            addr -= 1;
                        }
                    }
                }
            }
        }
        None => {}
    }
    None
}
pub fn addr_fmt(addr: u64, fcnname: Option<String>) {
    let loc = addr2sym(addr);
    let part = if loc.is_some() {
        loc.unwrap()
    } else {
        "???".to_string()
    };
    let part2 = if fcnname.is_some() {
        fcnname.unwrap()
    } else {
        "???".to_string()
    };
    println!(" at {:p} {} in {}", addr as *const u8, part, part2);
}

```
### ./kernel/src/memory/allocator.rs
```rust
use crate::{dprintln, print, println};
// use core::alloc::GlobalAlloc;
use core::ptr::null_mut;
use core::{
    alloc::{GlobalAlloc, Layout},
    ptr::NonNull,
};
use linked_list_allocator::{Heap, LockedHeap};
use x86_64::structures::paging::mapper::MapToError;
use x86_64::structures::paging::{FrameAllocator, Mapper, Page, PageTableFlags, Size4KiB};
use x86_64::VirtAddr;
pub struct WrapperAlloc {}
#[global_allocator]
pub static WRAPPED_ALLOC: WrapperAlloc = WrapperAlloc {};
impl WrapperAlloc {
    pub unsafe fn do_alloc(&self, layout: Layout) -> *mut u8 {
        // dprintln!("{:?} {:?}", layout, layout.align_to(16).unwrap());
        x86_64::instructions::interrupts::without_interrupts(|| {
            ralloc::Allocator.alloc(layout.align_to(8).unwrap())
        })
    }
    pub unsafe fn do_dealloc(&self, ptr: *mut u8, layout: Layout) {
        x86_64::instructions::interrupts::without_interrupts(|| {
            ralloc::Allocator.dealloc(ptr, layout.align_to(8).unwrap())
        })
    }
}
unsafe impl core::alloc::GlobalAlloc for WrapperAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        return self.do_alloc(layout);
    }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        return self.do_dealloc(ptr, layout);
    }
}
pub static ALLOCATOR: crate::shittymutex::Mutex<Heap> =
    crate::shittymutex::Mutex::new(Heap::empty());
pub const HEAP_START: usize = 0x100000000;
pub const HEAP_SIZE: usize = 4 * 1024;
pub const COW_PAGE: PageTableFlags = PageTableFlags::BIT_10;
pub const STACK_PAGE: PageTableFlags = PageTableFlags::BIT_11;
pub static CUR_ADDR: core::sync::atomic::AtomicU64 =
    core::sync::atomic::AtomicU64::new((HEAP_START + HEAP_SIZE) as u64);
pub static CUR_ADDR_PUB: core::sync::atomic::AtomicU64 =
    core::sync::atomic::AtomicU64::new((HEAP_START + HEAP_SIZE) as u64);

pub fn expand_by(size: u64) {
    CUR_ADDR_PUB.fetch_add(size, core::sync::atomic::Ordering::Relaxed);
    let size = ((size + 4095) / 4096) * 4096;
    let num = CUR_ADDR.fetch_add(size, core::sync::atomic::Ordering::Relaxed);
    expand_ram(num, size).expect("Failed expanding RAM");
}
pub fn init_heap() -> Result<(), MapToError<Size4KiB>> {
    let mut frame_alloc = crate::memory::FRAME_ALLOC
        .get()
        .expect("A frame allocator was not made yet");

    let page_range = {
        let heap_start = VirtAddr::new(HEAP_START as u64);
        let heap_end = heap_start + HEAP_SIZE - 1u64;
        let heap_start_page = Page::containing_address(heap_start);
        let heap_end_page = Page::containing_address(heap_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };

    for page in page_range {
        let frame = frame_alloc
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        unsafe {
            super::get_mapper()
                .map_to(page, frame, flags, &mut frame_alloc)?
                .flush();
        };
    }

    unsafe {
        ALLOCATOR.get().init(HEAP_START, HEAP_SIZE);
    }

    Ok(())
}

fn expand_ram(from: u64, size: u64) -> Result<(), MapToError<Size4KiB>> {
    let mut frame_alloc = crate::memory::FRAME_ALLOC
        .get()
        .expect("A frame allocator was not made yet");
    let page_range = {
        let heap_start = VirtAddr::new(from as u64) + 1u64;
        let heap_end = heap_start + size - 4u64;
        let heap_start_page = Page::containing_address(heap_start);
        let heap_end_page = Page::containing_address(heap_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };
    for page in page_range {
        let frame = frame_alloc
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        unsafe {
            super::get_mapper()
                .map_to(page, frame, flags, &mut frame_alloc)?
                .flush();
        };
    }

    unsafe {
        ALLOCATOR.get().extend(size as usize);
    }

    Ok(())
}

```
### ./kernel/src/preempt/mod.rs
```rust
// preemtive multitasking
use crate::prelude::*;
use safety_here::Jmpbuf;
use x86_64::{
    instructions::tables::{lgdt, load_tss},
    registers::control::Cr3,
    structures::{
        gdt::GlobalDescriptorTable, paging::mapper::MapToError, paging::FrameAllocator,
        paging::Mapper, paging::Page, paging::PageTableFlags, paging::PhysFrame, paging::Size4KiB,
        tss::TaskStateSegment,
    },
    VirtAddr,
};
static STACKS: AtomicU64 = AtomicU64::new(0x18000u64);
pub fn stack_alloc(stack_size: u64) -> Result<*const u8, MapToError<Size4KiB>> {
    let mut frame_alloc = crate::memory::FRAME_ALLOC
        .get()
        .expect("A frame allocator was not made yet");
    // frame_alloc.show();
    let base = STACKS.fetch_add(stack_size, Ordering::Relaxed);
    let page_range = {
        let stack_start = VirtAddr::new(base as u64);
        let stack_end = stack_start + stack_size - 1u64;
        let heap_start_page = Page::containing_address(stack_start);
        let heap_end_page = Page::containing_address(stack_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };

    for page in page_range {
        let frame = frame_alloc
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        println!("MAP: {:#?} -> {:#?}", page, frame);
        unsafe {
            memory::get_mapper()
                .map_to(page, frame, flags, &mut frame_alloc)?
                .flush();
        };
    }
    Ok(base as *const u8)
}
static TASK_QUEUE_CUR: AtomicUsize = AtomicUsize::new(0);
// privesc would be CURRENT_TASK.get().pid |= 1. just sayin' you know.
#[derive(Copy, Clone, Debug)]
pub struct Task {
    pub state: Jmpbuf,
    pub rsp0: VirtAddr,
    pub rsp_ptr: VirtAddr,
    pub pid: u64,
    pub box1: Option<&'static [u8]>,
    pub box2: Option<&'static [u8]>,
    pub program_break: u64,
    pub is_listening: bool,
    pub is_done: bool,
}
ezy_static! { TASK_QUEUE, Vec<Task>, vec![Task { state: Jmpbuf::new(), rsp0: crate::interrupts::get_rsp0(), rsp_ptr: crate::userland::alloc_rsp_ptr("syscall-stack:/bin/init".to_string()), pid: 1, box1: None, box2: None, program_break: 0, is_listening: false, is_done: false }] }
ezy_static! { CURRENT_TASK, Task, Task { state: Jmpbuf::new(), rsp0: crate::interrupts::get_rsp0(), rsp_ptr: crate::userland::alloc_rsp_ptr("fake stack".to_string()), pid: 1, box1: None, box2: None, program_break: 0, is_listening: false, is_done: false } }
extern "C" fn get_next(buf: &mut Jmpbuf) {
    let tq = TASK_QUEUE.get();
    let mut ct = CURRENT_TASK.clone();
    ct.state = buf.clone();
    tq[TASK_QUEUE_CUR.fetch_add(1, Ordering::Relaxed)] = ct;
    TASK_QUEUE_CUR.store(
        TASK_QUEUE_CUR.load(Ordering::Relaxed) % tq.len(),
        Ordering::Relaxed,
    );
    let q = tq[TASK_QUEUE_CUR.load(Ordering::Relaxed) % tq.len()];
    *CURRENT_TASK.get() = q;
    crate::interrupts::set_rsp0(q.rsp0);
    crate::userland::set_rsp_ptr(q.rsp_ptr);
    *buf = q.state;
}
pub fn yield_task() -> () {
    x86_64::instructions::interrupts::without_interrupts(|| {
        let c = Cr3::read();
        // set ourselves up as a task.
        let mut bufl = Jmpbuf::new();

        unsafe {
            safety_here::changecontext(get_next, &mut bufl);
        }
        unsafe {
            Cr3::write(c.0, c.1);
        }
    });
}
// we allow this here; this is a setup call for a new task
#[allow(improper_ctypes_definitions)]
extern "C" fn setup_call(
    _1: u64,
    _2: u64,
    _3: u64,
    _4: u64,
    _5: u64,
    _6: u64,
    fcn: fn(arg: u64) -> (),
    arg: u64,
) -> ! {
    x86_64::instructions::interrupts::enable();
    fcn(arg);
    loop {
        yield_task();
    }
}

pub fn task_alloc<T: FnOnce<()>>(f: T, stknm: String) {
    fn run_task_ll<T: FnOnce<()>>(arg: u64) {
        let b = unsafe { Box::from_raw(arg as *mut T) };
        b();
    }
    let b = Box::new(f);
    let ptr = Box::leak(b) as *const T;
    jump_to_task(run_task_ll::<T>, ptr as u64, stknm);
}
fn jump_to_task(newfcn: fn(arg: u64) -> (), arg: u64, stknm: String) {
    const STACK_SIZE_IN_QWORDS: usize = 1024;
    let end_of_stack = STACK_SIZE_IN_QWORDS - 2;
    let mut stack: Box<[u64]> = box [0; STACK_SIZE_IN_QWORDS];
    let index: usize = end_of_stack - 1; // Represents the callee saved registers
    stack[end_of_stack] = newfcn as u64;
    stack[end_of_stack + 1] = arg;
    let stack_ptr = Box::into_raw(stack);
    let stack_ptr_as_usize = stack_ptr as *mut u64 as usize;
    let stack_ptr_start = stack_ptr_as_usize + (index * core::mem::size_of::<usize>());

    let mut b = safety_here::Jmpbuf::new();
    b.rbx = 0;
    b.rbp = 0;
    b.r12 = 0;
    b.r13 = 0;
    b.r14 = 0;
    b.r15 = 0;
    b.rsp = stack_ptr_start as u64;
    b.rip = setup_call as *const u8 as u64;
    b.rsi = newfcn as *const u8 as u64;
    x86_64::instructions::interrupts::without_interrupts(|| {
        TASK_QUEUE.get().push(Task {
            state: b,
            rsp0: crate::interrupts::alloc_rsp0(),
            rsp_ptr: crate::userland::alloc_rsp_ptr(stknm),
            pid: 1,
            box1: None,
            box2: None,
            program_break: 0,
            is_listening: false,
            is_done: false,
        });
    });
}

#[cfg(test)]
static VAL: AtomicU64 = AtomicU64::new(3);

#[test_case]
pub fn test_tasks() {
    testing::test_header("Yields");
    task_alloc(|| {
        VAL.store(0, Ordering::Release);
    });
    // now
    yield_task();
    if VAL.load(Ordering::Relaxed) != 3 {
        testing::test_ok();
    } else {
        testing::test_fail();
    }
}

```
### ./kernel/src/proc.rs
```rust


```
### ./kernel/src/queue.rs
```rust
use alloc::{boxed::Box, vec, vec::Vec};

#[derive(Debug, Clone)]
pub struct ArrayQueue<T> {
    vec: Vec<Option<T>>,
    head: usize,
    tail: usize,
    pub count: usize,
}
impl<T> ArrayQueue<T> {
    pub fn push(&mut self, x: T) {
        if self.count == self.vec.len() {
            panic!("queue overflown")
        }
        self.vec[self.head] = Some(x);
        self.head += 1;
        if self.head as usize == self.vec.len() {
            self.head = 0;
        }
        self.count += 1;
    }
    pub fn pop(&mut self) -> &T {
        if self.tail == self.head {
            panic!("Bad read");
        }
        self.count -= 1;
        let val = self.vec[self.tail].as_ref().unwrap();
        self.tail += 1;
        if self.tail as usize == self.vec.len() {
            self.tail = 0;
        }
        val
    }
    pub fn new(sz: usize) -> ArrayQueue<T> {
        let mut q = ArrayQueue {
            vec: vec![],
            head: 0,
            tail: 0,
            count: 0,
        };
        for _i in 0..sz {
            q.vec.push(None)
        }
        q
    }
}

```
### ./kernel/src/shell/mod.rs
```rust
use crate::{drive::cpio::CPIOEntry, prelude::*};
struct Z {
    pub x: usize,
}
const ADDR: u16 = 0x1f0;
ezy_static! { CPIO, alloc::vec::Vec<CPIOEntry>, crate::drive::cpio::parse(
    &mut unsafe { crate::drive::Drive::new(true, ADDR, 0x3f6) }.get_offreader(),
).unwrap() }
fn ls(cmd: String) {
    {
        let mut file = "/".to_string() + &cmd + "/";
        if file == "" {
            file = "/".to_string();
        }
        file = file.replace("/./", "/");
        file = file.replace("//", "/");
        file = file.replace("//", "/");
        file = file.replace("//", "/");

        println!("Listing of {}", file);

        let entries = CPIO.get();
        for e in &*entries {
            if ("/".to_string() + e.name.clone().as_str()).starts_with(&(file.clone())) {
                println!("  {} | sz={} | perms={}", e.name, e.filesize, e.mode);
            }
        }
    }
}
pub fn prompt() {
    println!("+-------------------+");
    println!("| OhEs Shell v0.2.0 |");
    println!("+-------------------+");
}
fn cat(file: String) {
    let entries = CPIO.get();
    let mut is_ok = false;
    for e in &*entries {
        if e.name == file || (e.name.clone() + "/") == file {
            println!("{}", String::from_utf8(e.data.clone()).unwrap());
            is_ok = true;
            break;
        }
    }
    if !is_ok {
        println!("Error: ENOENT");
    }
}
fn loadksymmap(file: String) {
    let entries = CPIO.get();
    let mut is_ok = false;
    for e in &*entries {
        if e.name == file || (e.name.clone() + "/") == file {
            ksymmap::load_ksymmap(e.name.clone(), e.data.clone().as_slice());
            is_ok = true;
            break;
        }
    }
    if !is_ok {
        println!("Error: ENOENT");
    }
}
fn min(a: usize, b: usize) -> usize {
    if a < b {
        a
    } else {
        b
    }
}
pub async fn shell() {
    println!("Shell!");
    prompt();
    let mut ch: Vec<String> = vec!["prompt".to_string()];

    let cmds_m = Mutex::new(vec!["ls", "cat", "exit", "prompt", "cls", "pci"]);
    let mut cmd_h = BTreeMap::<String, Box<dyn Fn<(), Output = ()>>>::new();
    let input: Mutex<String> = Mutex::new("<unk>".to_string());
    macro_rules! cmd {
        ( $code:expr ) => {
            cmds_m.get().push(stringify!($code));
            cmd_h.insert(
                stringify!($code).to_string(),
                Box::new(|| ($code)(input.get().clone())),
            );
        };
    }
    macro_rules! ecmd {
        ( $name:ident, $code:expr ) => {
            cmds_m.get().push(stringify!($name));
            cmd_h.insert(
                stringify!($name).to_string(),
                Box::new(|| {
                    $code;
                }),
            );
        };
    }
    cmd!(ls);
    cmd!(cat);
    cmd!(loadksymmap);
    ecmd!(cls, Printer.clear_screen());
    ecmd!(sup, Printer.scroll_up());
    ecmd!(prompt, prompt());
    ecmd!(panic, panic!("You asked for it..."));
    ecmd!(user, crate::userland::loaduser());
    ecmd!(gptt, drive::gpt::test0());
    ecmd!(pci, crate::pci::testing());

    loop {
        print!("\x1b[44m\x1b[30m ~ \x1b[0m\x1b[34m\u{e0b0}\x1b[0m ");
        let im = Mutex::new(Z { x: ch.len() });
        let result: String = input!(
            || {
                let mut i = im.get().x;
                if i == 0 {
                    return None;
                }
                i -= 1;
                im.get().x = i;
                Some(ch[i].clone())
            },
            || {
                let mut i = im.get().x;
                if i == ch.len() {
                    return None;
                }
                i += 1;
                im.get().x = i;
                if i == ch.len() {
                    return Some("".to_string());
                }
                Some(ch[i].clone())
            },
            |s: String| {
                let mut can_suggest: Option<String> = None;
                let cmds = cmds_m.get();
                for opt in cmds.clone().into_iter() {
                    if can_suggest.is_none() && opt.starts_with(&s) {
                        can_suggest = Some(opt.to_string().clone());
                    } else if can_suggest.is_some() && opt.starts_with(&s) {
                        let sug = can_suggest.clone().unwrap();
                        let lene = min(sug.len(), opt.len());
                        let sc = sug.clone();
                        let mut s_iter = sc.chars();
                        let mut o_iter = opt.clone().chars();
                        let mut mxi = 0;
                        for _i in 0..lene {
                            if s_iter.next() == o_iter.next() {
                                mxi += 1;
                            } else {
                                break;
                            }
                        }
                        can_suggest = Some(sc.split_at(mxi).0.to_string());
                    }
                }
                can_suggest
            }
        )
        .await;
        ch.push(result.clone());
        let cmd = result.split(' ').next().unwrap();
        *input.get() = result
            .clone()
            .split_at(cmd.len())
            .1
            .to_string()
            .trim()
            .to_string();

        match cmd {
            "exit" => {
                return;
            }
            "cat" => {}
            _ => match cmd_h.get(cmd) {
                Some(handler) => {
                    handler();
                }
                None => {
                    println!("Unknown command: {}", cmd);
                }
            },
        }
    }
}

```
### ./kernel/src/shittymutex.rs
```rust
use core::{
    cell::UnsafeCell,
    ops::{Deref, DerefMut},
};

// Shitty mutex

pub struct Mutex<T: ?Sized> {
    data: UnsafeCell<T>,
}

/// A guard to which the protected data can be accessed
///
/// When the guard falls out of scope it will release the lock.
#[derive(Debug)]
pub struct MutexGuard<'a, T: ?Sized + 'a> {
    data: &'a mut T,
}

// Same unsafe impls as `std::sync::Mutex`
unsafe impl<T: ?Sized + Send> Sync for Mutex<T> {}
unsafe impl<T: ?Sized + Send> Send for Mutex<T> {}

impl<T> Deref for Mutex<T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &mut *self.data.get() }
    }
}
impl<T> DerefMut for Mutex<T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.data.get() }
    }
}
impl<T> Mutex<T> {
    pub const fn new(data: T) -> Mutex<T> {
        Mutex {
            data: UnsafeCell::new(data),
        }
    }
    pub fn get(&self) -> &mut T {
        unsafe { &mut *self.data.get() }
    }
}

```
### ./kernel/src/task/keyboard.rs
```rust
use crate::{print, println};
use conquer_once::spin::OnceCell;
use crossbeam_queue::ArrayQueue;
pub static KEY_QUEUE: OnceCell<ArrayQueue<pc_keyboard::DecodedKey>> = OnceCell::uninit();

pub(crate) fn key_enque(scancode: pc_keyboard::DecodedKey) {
    if let Ok(queue) = KEY_QUEUE.try_get() {
        if let Err(_) = queue.push(scancode) {
            println!("WARNING: scancode queue full; dropping keyboard input");
        }
    } else {
        println!("WARNING: scancode queue uninitialized");
    }
}

```
### ./kernel/src/task/mod.rs
```rust
use alloc::boxed::Box;
use core::{
    future::Future,
    pin::Pin,
    task::{Context, Poll},
};

pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }
}
impl Task {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}

pub mod keyboard;
pub mod simple_executor;

```
### ./kernel/src/task/simple_executor.rs
```rust
use super::Task;
use alloc::collections::VecDeque;
use core::task::RawWakerVTable;
use core::task::{Context, Poll, RawWaker, Waker};
pub struct SimpleExecutor {
    task_queue: VecDeque<Task>,
}

impl SimpleExecutor {
    pub fn new() -> SimpleExecutor {
        SimpleExecutor {
            task_queue: VecDeque::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {}
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}

fn dummy_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        dummy_raw_waker()
    }

    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(0 as *const (), vtable)
}
fn dummy_waker() -> Waker {
    unsafe { Waker::from_raw(dummy_raw_waker()) }
}

```
### ./kernel/src/testing.rs
```rust
use crate::io;
use crate::print;
use crate::println;
pub fn test_ok() {
    io::Printer.set_color(0, 255, 0);
    println!("[ok]");
    io::Printer.set_color(255, 255, 255);
}
pub fn test_fail() {
    io::Printer.set_color(255, 0, 0);
    println!("[fail]");
    io::Printer.set_color(255, 255, 255);
    panic!("Test failed!");
}

pub fn test_header(test: &str) {
    print!("Test: {}... ", test);
}

```
### ./kernel/src/uw.rs
```rust

```
### ./kernel/src/ksvc/mod.rs
```rust
use crate::{drive::RODev, prelude::*};
use drive::gpt::GetGPTPartitions;
use serde_derive::*;
use x86_64::structures::paging::PageTableFlags;
ezy_static! { KSVC_TABLE, BTreeMap<String, Box<dyn Send + Sync + Fn<(), Output = ()>>>, BTreeMap::new() }

#[derive(Serialize, Deserialize)]
pub enum KSvcResult {
    Success,
    Failure(String),
}

#[derive(Serialize, Deserialize, PartialEq, Eq, PartialOrd, Ord)]
pub enum FSOp {
    Read,
    ReadDir,
    Stat,
}
#[derive(Serialize, Deserialize)]
pub enum FSResult {
    Data(Vec<u8>),
    Dirents(Vec<String>),
    Stats(u16),
    Failure(String),
}

pub fn ksvc_init() {
    let t = KSVC_TABLE.get();

    t.insert("log".to_string(), box || {
        let d: String = postcard::from_bytes(preempt::CURRENT_TASK.box1.unwrap()).unwrap();
        print!("{}", d);
        let x = postcard::to_allocvec(&KSvcResult::Success).unwrap();
        preempt::CURRENT_TASK.get().box1 = Some(x.leak());
    });
    t.insert("pmap".to_string(), box || {
        let d: u64 = postcard::from_bytes(preempt::CURRENT_TASK.box1.unwrap()).unwrap();
        dprint!(
            "[pmap] Mapping in {:#x?} to {:#x?}",
            d,
            d + 0xffffffffc0000000
        );
        memory::map_to(
            VirtAddr::from_ptr(d as *const u8),
            VirtAddr::from_ptr((d + 0xffffffffc0000000) as *const u8),
            PageTableFlags::USER_ACCESSIBLE | PageTableFlags::WRITABLE | PageTableFlags::NO_EXECUTE,
        );
    });
    t.insert("punmap".to_string(), box || {
        let d: u64 = postcard::from_bytes(preempt::CURRENT_TASK.box1.unwrap()).unwrap();
        dprint!("[punmap] UnMapping {:#x?}", d);
        memory::munmap(VirtAddr::from_ptr(d as *const u8));
    });
    t.insert("kfs".to_string(), box || {
        let d: (FSOp, String) = postcard::from_bytes(preempt::CURRENT_TASK.box1.unwrap()).unwrap();
        let mut drv: Box<dyn RODev> = box drive::SickCustomDev {};
        let gpt = drv.get_gpt_partitions(box || box drive::SickCustomDev {});
        let tbl: Vec<Box<(dyn RODev + 'static)>> =
            gpt.into_iter().map(|x| (box x) as Box<dyn RODev>).collect();
        let mut gpt0 = tbl.into_iter().next().unwrap();

        let path_elems: Vec<String> =
            d.1.split('/')
                .map(|p| p.to_string())
                .filter(|a| a != "")
                .collect();
        let bytes_of_superblock: Vec<u8> =
            vec![gpt0.read_from(2).unwrap(), gpt0.read_from(3).unwrap()]
                .into_iter()
                .flat_map(|f| f)
                .collect();
        let sb = drive::ext2::handle_super_block(bytes_of_superblock.as_slice());
        let inode_id = drive::ext2::traverse_fs_tree(&mut gpt0, &sb, path_elems);

        let r = match d.0 {
            FSOp::Read => FSResult::Data(drive::ext2::cat(&mut gpt0, inode_id, &sb)),
            FSOp::ReadDir => FSResult::Dirents(
                drive::ext2::readdir(&mut gpt0, inode_id, &sb)
                    .into_iter()
                    .map(|k| k.0)
                    .collect(),
            ),
            FSOp::Stat => FSResult::Stats(drive::ext2::stat(&mut gpt0, inode_id, &sb).bits()),
        };
        let x = postcard::to_allocvec(&r).unwrap();
        preempt::CURRENT_TASK.get().box1 = Some(x.leak());
    });
}
pub fn dofs() {
    let d: (FSOp, String) = postcard::from_bytes(preempt::CURRENT_TASK.box1.unwrap()).unwrap();
    if d.0 == FSOp::Read {
        let r = FSResult::Data(userland::readfs(&d.1).to_vec());
        let x = postcard::to_allocvec(&r).unwrap();
        preempt::CURRENT_TASK.get().box1 = Some(x.leak());
        return
    }
    let mut drv: Box<dyn RODev> = box drive::SickCustomDev {};
    let gpt = drv.get_gpt_partitions(box || box drive::SickCustomDev {});
    let tbl: Vec<Box<(dyn RODev + 'static)>> =
        gpt.into_iter().map(|x| (box x) as Box<dyn RODev>).collect();
    let mut gpt0 = tbl.into_iter().next().unwrap();

    let path_elems: Vec<String> =
        d.1.split('/')
            .map(|p| p.to_string())
            .filter(|a| a != "")
            .collect();
    let bytes_of_superblock: Vec<u8> = vec![gpt0.read_from(2).unwrap(), gpt0.read_from(3).unwrap()]
        .into_iter()
        .flat_map(|f| f)
        .collect();
    let sb = drive::ext2::handle_super_block(bytes_of_superblock.as_slice());
    let inode_id = drive::ext2::traverse_fs_tree(&mut gpt0, &sb, path_elems);

    let r = match d.0 {
        FSOp::Read => FSResult::Data(userland::readfs(&d.1).to_vec()),
        FSOp::ReadDir => FSResult::Dirents(
            drive::ext2::readdir(&mut gpt0, inode_id, &sb)
                .into_iter()
                .map(|k| k.0)
                .collect(),
        ),
        FSOp::Stat => FSResult::Stats(drive::ext2::stat(&mut gpt0, inode_id, &sb).bits()),
    };
    let x = postcard::to_allocvec(&r).unwrap();
    preempt::CURRENT_TASK.get().box1 = Some(x.leak());
}

```
### ./kernel/src/constants.rs
```rust

pub fn is_test() -> bool {
    #[cfg(test)]
    return true;
    #[cfg(not(test))]
    return false;
}

pub fn should_conio() -> bool {
    #[cfg(feature = "conio")]
    return true;
    #[cfg(not(feature = "conio"))]
    return false;
}

pub fn should_displayio() -> bool {
    #[cfg(feature = "displayio")]
    return true;
    #[cfg(not(feature = "displayio"))]
    return false;
}

pub fn should_fini_exit() -> bool {
    #[cfg(feature = "fini_exit")]
    return true;
    #[cfg(not(feature = "fini_exit"))]
    return false;
}

pub fn should_fini_wait() -> bool {
    #[cfg(feature = "fini_wait")]
    return true;
    #[cfg(not(feature = "fini_wait"))]
    return false;
}
pub fn should_debug_log() -> bool {
    #[cfg(feature = "debug_logs")]
    return true;
    #[cfg(not(feature = "debug_logs"))]
    return false;
}

pub fn check_const_correct() {
    assert_eq!(
        should_fini_exit() || should_fini_wait(),
        true,
        "fini_exit or fini_wait must be set"
    );
}

```
### ./kernel/src/interrupts.rs
```rust
use crate::{print, println};
use lazy_static::lazy_static;
use pc_keyboard::{layouts, DecodedKey, HandleControl, Keyboard, ScancodeSet1};
use pic8259_simple::ChainedPics;
use x86_64::structures::gdt::{Descriptor, GlobalDescriptorTable};
use x86_64::structures::idt::{InterruptDescriptorTable, InterruptStackFrame, PageFaultErrorCode};
use x86_64::structures::tss::TaskStateSegment;
use x86_64::VirtAddr;
use x86_64::{
    instructions::port::{Port, PortRead, PortWrite},
    structures::gdt::SegmentSelector,
};
use x86_64::{
    registers::control::Cr2,
    structures::paging::{Mapper, Page, PhysFrame, Size4KiB},
};

pub const DOUBLE_FAULT_IST_INDEX: u16 = 0;
pub const PAGE_FAULT_STACK_INDEX: u16 = 1;
pub const PIC_1_OFFSET: u8 = 32;
pub const PIC_2_OFFSET: u8 = PIC_1_OFFSET + 8;

pub static PICS: crate::shittymutex::Mutex<ChainedPics> =
    crate::shittymutex::Mutex::new(unsafe { ChainedPics::new(PIC_1_OFFSET, PIC_2_OFFSET) });

#[derive(Debug, Clone, Copy)]
#[repr(u8)]
pub enum InterruptIndex {
    Timer = PIC_1_OFFSET, // 0
    Keyboard,             // 1
    _Cascade,             // 2
    COM2,                 // 3
    COM1,                 // 4
    LPT2,                 // 5
    Floppy,               // 6
    UnreliableLPT1,       // 7
    CMOS,                 // 8
    Free1,                // 9
    Free2,
    Free3,
    Mouse,
    FPU,
    PrimaryATA,
    SecondaryATA,
}

impl InterruptIndex {
    fn as_u8(self) -> u8 {
        self as u8
    }

    fn as_usize(self) -> usize {
        usize::from(self.as_u8())
    }
}

lazy_static! {
    static ref TSS: TaskStateSegment = {
        let mut tss = TaskStateSegment::new();
        tss.interrupt_stack_table[DOUBLE_FAULT_IST_INDEX as usize] = {
            const STACK_SIZE: usize = 4096 * 5;
            static mut STACK_DATA: [u8; STACK_SIZE] = [0; STACK_SIZE];
            let stack_start = VirtAddr::from_ptr(unsafe { &STACK_DATA });
            let stack_end = stack_start + STACK_SIZE;
            stack_end
        };
        tss.interrupt_stack_table[PAGE_FAULT_STACK_INDEX as usize] = {
            const STACK_SIZE: usize = 4096 * 128;
            static mut STACK_DATA: [u8; STACK_SIZE] = [0; STACK_SIZE];
            let stack_start = VirtAddr::from_ptr(unsafe { &STACK_DATA });
            let stack_end = stack_start + STACK_SIZE;
            stack_end + (4096 as usize)
        };
        tss.privilege_stack_table[0] = {
            // stack when going back to kmode.
            // for real only used
            const STACK_SIZE: usize = 4096 * 5;
            static mut STACK_DATA: [u8; STACK_SIZE] = [0; STACK_SIZE];
            let stack_start = VirtAddr::from_ptr(unsafe { &STACK_DATA });
            let stack_end = stack_start + STACK_SIZE;
            stack_end
        };

        tss
    };
}
pub fn get_rsp0() -> VirtAddr {
    TSS.privilege_stack_table[0]
}
pub fn set_rsp0(va: VirtAddr) {
    let tss_borrow: &TaskStateSegment = &TSS;
    let tss_mut = unsafe { &mut *(tss_borrow as *const TaskStateSegment as *mut TaskStateSegment) };
    tss_mut.privilege_stack_table[0] = va;
}
pub fn alloc_rsp0() -> VirtAddr {
    const STACK_SIZE: usize = 4096 * 5;
    let stack_start = VirtAddr::from_ptr(crate::memory::malloc(STACK_SIZE));
    let stack_end = stack_start + STACK_SIZE;
    stack_end
}

lazy_static! {
    pub static ref GDT: (GlobalDescriptorTable, Selectors) = {
let mut gdt = GlobalDescriptorTable::new();
        let code_selector = gdt.add_entry(Descriptor::kernel_code_segment()); // 0x08
        let data_selector = gdt.add_entry(Descriptor::kernel_data_segment()); // 0x10
        let userdata = gdt.add_entry(Descriptor::user_data_segment()); // 0x18
        let usercode = gdt.add_entry(Descriptor::user_code_segment()); // 0x20
        let tss_selector = gdt.add_entry(Descriptor::tss_segment(&TSS)); // 0x28
        (gdt, Selectors { code_selector, tss_selector, data_selector, usercode, userdata })
    };
}

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        unsafe {
            idt.double_fault
                .set_handler_fn(double_fault_handler)
                .set_stack_index(DOUBLE_FAULT_IST_INDEX);
        }
        unsafe {
            idt.page_fault
                .set_handler_fn(page_fault_handler)
                .set_stack_index(PAGE_FAULT_STACK_INDEX);
        }
        idt.invalid_opcode.set_handler_fn(invalid_opcode_handler);
        idt.general_protection_fault.set_handler_fn(gpe);
        idt[InterruptIndex::Timer.as_usize()].set_handler_fn(timer_interrupt_handler);
        idt[InterruptIndex::Keyboard.as_usize()].set_handler_fn(keyboard_handler);
        idt[InterruptIndex::COM1.as_usize()].set_handler_fn(com1_handler);
        idt[InterruptIndex::COM2.as_usize()].set_handler_fn(com1_handler);
        idt[InterruptIndex::PrimaryATA.as_usize()].set_handler_fn(com1_handler);
        unsafe { PICS.get().initialize() };
        idt
    };
}

lazy_static! {
    static ref KEYBOARD: crate::shittymutex::Mutex<Keyboard<layouts::Us104Key, ScancodeSet1>> =
        crate::shittymutex::Mutex::new(Keyboard::new(
            layouts::Us104Key,
            ScancodeSet1,
            HandleControl::Ignore
        ));
}

#[derive(Debug, Copy, Clone)]
pub struct Selectors {
    pub code_selector: SegmentSelector,
    pub data_selector: SegmentSelector,
    pub tss_selector: SegmentSelector,
    pub usercode: SegmentSelector,
    pub userdata: SegmentSelector,
}
extern "x86-interrupt" fn breakpoint_handler(stack_frame: &mut InterruptStackFrame) {
    println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame,);
}
extern "x86-interrupt" fn double_fault_handler(
    stack_frame: &mut InterruptStackFrame,
    error_code: u64,
) -> ! {
    panic!(
        "Double fault at: \n{:#?}\nError code: {}",
        stack_frame, error_code
    );
}
extern "x86-interrupt" fn gpe(stack_frame: &mut InterruptStackFrame, error_code: u64) -> () {
    panic!("#GP at: \n{:#?}\nError code: {}", stack_frame, error_code);
}
extern "x86-interrupt" fn invalid_opcode_handler(stack_frame: &mut InterruptStackFrame) -> () {
    panic!("Invalid opcode at: \n{:#?}", stack_frame);
}

extern "x86-interrupt" fn page_fault_handler(
    stack_frame: &mut InterruptStackFrame,
    error_code: PageFaultErrorCode,
) {
    println!("ifF: {}", x86_64::instructions::interrupts::are_enabled());
    let addr = Cr2::read();
    let f = crate::memory::get_flags_for(addr);
    println!("F: {:?}", f);
    // IF it was a COW page AND it was a write, copy and make writable (aka Copy On Write)
    // if error_code.intersects(PageFaultErrorCode::CAUSED_BY_WRITE) {
    //     let addr = Cr2::read();
    //     // reverse lookup to page frame
    //     let f = crate::memory::get_flags_for(addr);
    //     match f {
    //         Some(mut flags) => {
    //             if flags.intersects(crate::memory::allocator::COW_PAGE) {
    //                 // Yes indeed
    //                 // We need a new page
    //                 let pgn = crate::memory::mpage();
    //                 // Copy the data
    //                 unsafe { faster_rlibc::fastermemcpy(pgn, addr.as_ptr(), 4096); }
    //                 crate::memory::MAPPER.get().unmap(Page::<Size4KiB>::containing_address(addr));
    //                 flags.remove(crate::memory::allocator::COW_PAGE);
    //                 crate::memory::map_to(addr, VirtAddr::from_ptr(pgn), flags);
    //                 // Go back
    //                 return;
    //             }
    //         }
    //         None => {}
    //     }
    // }
    // loop
    crate::io::Printer.set_color(255, 0, 0);
    println!(
        "EXCEPTION: PAGE FAULT\nAccessed Address: {:?}\nWhy?: {:?}\n{:#?}",
        Cr2::read(),
        error_code,
        stack_frame
    );
    if (error_code & PageFaultErrorCode::MALFORMED_TABLE) == PageFaultErrorCode::MALFORMED_TABLE {
        panic!("A malformed table was detected.");
    }
    if (error_code & PageFaultErrorCode::INSTRUCTION_FETCH) == PageFaultErrorCode::INSTRUCTION_FETCH
    {
        panic!(
            "Error: We tried to run code at an invalid address {:?}",
            Cr2::read()
        );
    }
    if (error_code & PageFaultErrorCode::CAUSED_BY_WRITE) == PageFaultErrorCode::CAUSED_BY_WRITE {
        panic!(
            "Error: We tried to write to an invalid address {:?} from {:?}",
            Cr2::read(),
            stack_frame.instruction_pointer
        );
    }
    panic!("Page fault");
}

extern "x86-interrupt" fn timer_interrupt_handler(_stack_frame: &mut InterruptStackFrame) {
    unsafe {
        PICS.get()
            .notify_end_of_interrupt(InterruptIndex::Timer.as_u8());
    }
    if !crate::constants::is_test() {
        crate::preempt::yield_task();
    }
    // println!("if: {}", x86_64::instructions::interrupts::are_enabled());
}
extern "x86-interrupt" fn keyboard_handler(_stack_frame: &mut InterruptStackFrame) {
    x86_64::instructions::interrupts::disable();
    let keyboard = KEYBOARD.get();
    let scancode: u8 = unsafe { u8::read_from_port(0x60) };
    if let Ok(Some(key_event)) = keyboard.add_byte(scancode) {
        if let Some(dk) = keyboard.process_keyevent(key_event) {
            crate::task::keyboard::key_enque(dk);
        }
    }
    unsafe {
        PICS.get()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
    drop(keyboard);
    x86_64::instructions::interrupts::enable();
}
extern "x86-interrupt" fn com1_handler(_stack_frame: &mut InterruptStackFrame) {
    x86_64::instructions::interrupts::disable();
    unsafe {
        PICS.get()
            .notify_end_of_interrupt(InterruptIndex::COM1.as_u8());
    }
    x86_64::instructions::interrupts::enable();
}

pub fn init_idt() {
    IDT.load();
}

pub fn init_timer(freq: u32) {
    unsafe {
        u8::write_to_port(0x43, 0x34);
        u8::write_to_port(0x40, ((1193182 / freq) & 0xff) as u8);
        u8::write_to_port(0x40, ((1193182 / freq) >> 8) as u8);
    }
}

const PIC1_DATA: u16 = 0x21;
const PIC2_DATA: u16 = 0xa1;
pub fn noirq(mut line: u8) {
    let port: u16;
    let value: u8;

    if line < 8 {
        port = PIC1_DATA;
    } else {
        port = PIC2_DATA;
        line -= 8;
    }
    unsafe {
        value = u8::read_from_port(port) | (1 << line);
        u8::write_to_port(port, value);
    }
}
pub fn goirq(mut line: u8) {
    let port: u16;
    let value: u8;

    if line < 8 {
        port = PIC1_DATA;
    } else {
        port = PIC2_DATA;
        line -= 8;
    }
    unsafe {
        value = u8::read_from_port(port) & (0xff ^ (1 << line));
        u8::write_to_port(port, value);
    }
}

```
### ./kernel/src/lib.rs
```rust
#![no_std]
#![feature(abi_x86_interrupt)]
#![feature(alloc_error_handler)]
#![feature(allocator_api)]
#![feature(asm)]
#![feature(custom_test_frameworks)]
#![feature(exclusive_range_pattern)]
#![feature(lang_items)]
#![feature(unboxed_closures)]
#![test_runner(crate::main::test_runner)]
#![feature(box_syntax)]
#![reexport_test_harness_main = "test_main"]
#![allow(unused_imports)]
#![feature(naked_functions)]
#![feature(const_ptr_offset)]
#![feature(iter_advance_by)]
#![feature(const_raw_ptr_to_usize_cast)]
#![feature(link_llvm_intrinsics)]

extern crate alloc;
extern crate faster_rlibc;
extern crate kmacros;
extern crate safety_here;

pub mod constants;
pub mod devices;
pub mod drive;
pub mod events;
pub mod exiting;
pub mod init;
pub mod interrupts;
pub mod io;
pub mod stack_canaries;
pub mod ksvc;
pub mod ksymmap;
pub mod main;
pub mod memory;
pub mod pci;
pub mod preempt;
pub mod prelude;
pub mod proc;
pub mod queue;
pub mod shell;
pub mod shittymutex;
pub mod task;
pub mod testing;
pub mod unwind;
pub mod userland;

```
### ./kernel/src/main.rs
```rust
use crate::prelude::*;
use crate::task::{simple_executor::SimpleExecutor, Task};
use core::panic::PanicInfo;
use multiboot2::BootInformation;
use x86_64::{VirtAddr, registers::control::{Cr3, Cr3Flags}, structures::paging::{PhysFrame, Size4KiB}};
use xmas_elf::{self, symbol_table::Entry};

pub fn i2s(n: u32) -> alloc::string::String {
    n.to_string()
}
#[macro_export]
macro_rules! add_fn {
    ($fcn: ident) => {
        crate::backtrace::mark(
            $fcn as *const (),
            &(alloc::string::String::from(stringify!($fcn))
                + " @ "
                + file!()
                + ":"
                + &$crate::i2s(line!())),
        );
    };
}
#[cfg(test)]
pub fn test_runner(tests: &[&dyn Fn()]) {
    println!("Running {} test(s)", tests.len());
    io::Printer.set_color(255, 255, 255);
    for test in tests {
        test();
    }
}

#[test_case]
fn trivial_test() {
    testing::test_header("Trivial Test");
    assert_eq!(1, 1);
    testing::test_ok();
}

#[test_case]
fn int3_no_crash() {
    testing::test_header("Int3 no crash");
    x86_64::instructions::interrupts::int3();
    testing::test_ok();
}

#[test_case]
fn no_crash_alloc() {
    testing::test_header("No crash on alloc");
    let _ = Box::new(41);
    testing::test_ok();
}

#[lang = "eh_personality"]
extern "C" fn eh_personality() {}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    x86_64::instructions::interrupts::disable();
    io::Printer.set_color(255, 0, 0);
    println!("--------------- Kernel Panic (not syncing) ---------------");
    println!("pid: {}", preempt::CURRENT_TASK.pid);
    println!("info: {}", info);
    println!("] stack checking...");
    stack_canaries::stk_chk();
    println!("] stacks checked");
    // unwind::backtrace();
    exiting::exit_fail();
}

#[alloc_error_handler]
fn alloc_error_handler(layout: alloc::alloc::Layout) -> ! {
    panic!("allocation error: {:?}", layout)
}

pub fn forkp() -> (PhysFrame, Cr3Flags) {
    // let's get my pages
    let old_pt = crate::memory::get_l4();
    let new_pt = crate::memory::mpage();
    unsafe {
        crate::faster_rlibc::fastermemcpy(
            new_pt,
            old_pt as *mut x86_64::structures::paging::PageTable as *const u8,
            2048,
        );
        crate::faster_rlibc::fastermemset(new_pt.offset(2048), 0, 2048);
    }
    let flags = Cr3::read().1;
    let q = PhysFrame::containing_address(
        crate::memory::translate(VirtAddr::from_ptr(new_pt)).unwrap(),
    );
    unsafe {
        Cr3::write(
            PhysFrame::containing_address(
                crate::memory::translate(VirtAddr::from_ptr(new_pt)).unwrap(),
            ),
            flags,
        );
    }
    (
        q,
        flags,
    )
}
#[no_mangle]
pub extern "C" fn kmain(boot_info_ptr: u64) -> ! {
    // ralloc::Allocator
    let ptr = unsafe { multiboot2::load(boot_info_ptr as usize) };
    let boot_info = unsafe { &*((&ptr) as *const BootInformation) as &'static BootInformation };
    constants::check_const_correct();
    init::init(boot_info);
    {
        let mut executor = SimpleExecutor::new();
        executor.spawn(Task::new(shell::shell()));
        executor.run();
    }
    loop {}
}
//     init(boot_info);
//     constants::check_const_correct();
//     if constants::should_conio() {
//         println!("Should conio");
//     }
//     if constants::should_displayio() {
//         println!("Should displayio");
//     }
//     // TODO: this really really needs to work. Actually, we should use ld syms as per doug16k's suggestion.
//     // let kernel = unsafe {
//     //     alloc::slice::from_raw_parts(
//     //         boot_info.kernel_addr as *const u8,
//     //         boot_info.kernel_size as usize,
//     //     )
//     // };
//     // TODO: get unwind working
//     // crate::unwind::register_module(kernel);
//     let rsp: usize;
//     unsafe { asm!("mov rax, rsp", out("rax") rsp) };
//     println!("z: {:?}", VirtAddr::new(rsp as u64));
//     #[cfg(test)]
//     test_main();
//     #[cfg(not(test))]
//     {
//         let mut executor = SimpleExecutor::new();
//         executor.spawn(Task::new(shell::shell()));
//         executor.run();
//     }
//     if constants::is_test() {
//         exiting::exit_qemu(exiting::QemuExitCode::Success);
//     } else {
//         exiting::exit();
//     }
// }

#[no_mangle]
pub extern "C" fn _Unwind_Resume() -> () {}

```
### ./kernel/src/memory.rs
```rust
use crate::{dprintln, println};
use alloc::{alloc::Layout, vec::Vec};
use allocator::CUR_ADDR_PUB;
// use bootloader::bootinfo::{MemoryMap, MemoryRegionType};
use crate::shittymutex::Mutex;
use core::sync::atomic::{AtomicUsize, Ordering};
use lazy_static::lazy_static;
use multiboot2::{MemoryArea, MemoryAreaType, MemoryMapTag};
use x86_64::structures::paging::page_table::PageTable;
use x86_64::structures::paging::{
    FrameAllocator, Mapper, MapperAllSizes, OffsetPageTable, Page, PageTableFlags, PhysFrame,
    Size4KiB,
};
use x86_64::{registers::control::Cr3, structures::paging::page_table::PageTableEntry};
use x86_64::{PhysAddr, VirtAddr};
// pub static PHBASE: AtomicUsize = AtomicUsize::new(0);

pub mod allocator;
pub fn munmap(area: VirtAddr) {
    let u = crate::memory::get_mapper()
        .unmap(Page::<Size4KiB>::containing_address(area))
        .unwrap()
        .1;
    u.flush();
}
pub fn try_munmap(area: VirtAddr) {
    match crate::memory::get_mapper().unmap(Page::<Size4KiB>::containing_address(area)) {
        Ok(o) => {
            o.1.flush();
        }
        Err(_) => {}
    }
}
pub fn map_to(from: VirtAddr, to: VirtAddr, flags: PageTableFlags) {
    if translate(to).is_some() {
        return;
    }
    let frame = PhysFrame::<Size4KiB>::containing_address(crate::memory::translate(from).unwrap());
    let flags = PageTableFlags::PRESENT | flags;
    let map_to_result = unsafe {
        crate::memory::get_mapper().map_to(
            Page::containing_address(to),
            frame,
            flags,
            &mut crate::memory::FRAME_ALLOC.get().unwrap(),
        )
    };
    map_to_result.expect("map_to failed").flush();
}
#[macro_export]
macro_rules! phmem_offset {
    () => {
        x86_64::VirtAddr::new(0 as u64)
    };
}

pub fn get_mapper() -> OffsetPageTable<'static> {
    unsafe { OffsetPageTable::new(get_l4(), phmem_offset!()) }
}
pub fn get_l4() -> &'static mut PageTable {
    let (level_4_table_frame, _) = Cr3::read();

    let phys = level_4_table_frame.start_address();
    let virt = phmem_offset!() + phys.as_u64();
    let ptr: *mut PageTable = virt.as_mut_ptr();

    unsafe { &mut *ptr }
}
pub fn convig(x: u64) -> u64 {
    crate::memory::translate(VirtAddr::new(x as u64))
        .unwrap()
        .as_u64()
}
pub fn convpc<T>(x: *const T) -> u64 {
    crate::memory::translate(VirtAddr::new(x as u64))
        .unwrap()
        .as_u64()
}
pub fn convpm<T>(x: *mut T) -> u64 {
    crate::memory::translate(VirtAddr::new(x as u64))
        .unwrap()
        .as_u64()
}
pub fn translate(addr: VirtAddr) -> Option<PhysAddr> {
    get_mapper().translate_addr(addr)
}
fn get_entry_for(addr: VirtAddr) -> Option<&'static PageTableEntry> {
    use x86_64::registers::control::Cr3;
    use x86_64::structures::paging::page_table::FrameError;

    // read the active level 4 frame from the CR3 register
    let (level_4_table_frame, _) = Cr3::read();

    let table_indexes = [
        addr.p4_index(),
        addr.p3_index(),
        addr.p2_index(),
        addr.p1_index(),
    ];
    let mut frame = level_4_table_frame;
    let mut last_entry = unsafe { &*(0 as *const PageTableEntry) };
    // traverse the multi-level page table
    for &index in &table_indexes {
        // convert the frame into a page table reference
        let virt = phmem_offset!() + frame.start_address().as_u64();
        let table_ptr: *const PageTable = virt.as_ptr();
        let table = unsafe { &*table_ptr };

        // read the page table entry and update `frame`
        let entry = &table[index];
        frame = match entry.frame() {
            Ok(frame) => frame,
            Err(FrameError::FrameNotPresent) => return None,
            Err(FrameError::HugeFrame) => return Some(entry),
        };
        last_entry = entry;
    }

    // calculate the physical address by adding the page offset
    Some(last_entry)
}
pub fn get_flags_for(va: VirtAddr) -> Option<PageTableFlags> {
    match get_entry_for(va) {
        Some(f) => Some(f.flags()),
        None => None,
    }
}

pub fn create_example_mapping(page: Page, frame_allocator: &mut impl FrameAllocator<Size4KiB>) {
    let frame = PhysFrame::containing_address(PhysAddr::new(0xfd000000));
    let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;

    let map_to_result = unsafe { get_mapper().map_to(page, frame, flags, frame_allocator) };
    map_to_result.expect("map_to failed").flush();
}

static NEXT: AtomicUsize = AtomicUsize::new(0);
#[derive(Debug, Copy, Clone)]
pub struct BootInfoFrameAllocator {
    memory_map: &'static MemoryMapTag,
}

extern "C" {
    pub static es: u8;
    pub static esz: u8;
    pub static ee: u8;
}

impl BootInfoFrameAllocator {
    pub unsafe fn init(memory_map: &'static MemoryMapTag) -> Self {
        BootInfoFrameAllocator { memory_map }
    }
    fn usable_frames(&self) -> impl Iterator<Item = PhysFrame> {
        let regions = self.memory_map.memory_areas();
        let usable_regions = regions.filter(|r| r.typ() == MemoryAreaType::Available);
        let addr_ranges = usable_regions.map(|r| r.start_address()..r.end_address());
        let frame_addresses = addr_ranges.flat_map(|r| r.step_by(4096));
        frame_addresses
            .filter(|p| unsafe { core::mem::transmute::<&u8, u64>(&ee) < *p })
            .map(|addr| PhysFrame::containing_address(PhysAddr::new(addr)))
    }
    pub fn show(&self) {
        let x: Vec<PhysFrame<Size4KiB>> = self.usable_frames().collect();
        println!("Frames: {:#?}", x);
    }
}
unsafe impl FrameAllocator<Size4KiB> for BootInfoFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> {
        let frame = self
            .usable_frames()
            .nth(NEXT.fetch_add(1, Ordering::Relaxed));
        frame
    }
}
lazy_static! {
    pub static ref FRAME_ALLOC: Mutex<Option<BootInfoFrameAllocator>> = Mutex::new(None);
}

#[no_mangle]
pub extern "C" fn malloc(size: usize) -> *mut u8 {
    unsafe {
        let lsz = core::mem::size_of::<usize>();
        let data = crate::memory::allocator::WRAPPED_ALLOC
            .do_alloc(Layout::from_size_align(size + lsz, 8).unwrap());
        if data == 0 as *mut u8 {
            panic!("Bad alloc!");
        }
        (*(data as *mut usize)) = size + lsz;
        data.offset(lsz as isize)
    }
}

#[no_mangle]
pub extern "C" fn free(el: *mut u8) {
    unsafe {
        crate::memory::allocator::WRAPPED_ALLOC.do_dealloc(
            el.offset(-8),
            Layout::from_size_align(*(el.offset(-8) as *const usize), 8).unwrap(),
        )
    };
}

pub fn mpage() -> *mut u8 {
    unsafe {
        let data = crate::memory::allocator::WRAPPED_ALLOC
            .do_alloc(Layout::from_size_align(4096, 4096).unwrap());
        if data == 0 as *mut u8 {
            panic!("Bad alloc!");
        }
        data
    }
}

pub fn fpage(el: *mut u8) {
    unsafe {
        crate::memory::allocator::WRAPPED_ALLOC
            .do_dealloc(el, Layout::from_size_align(4096, 4096).unwrap())
    };
}
#[no_mangle]
pub extern "C" fn brk(to: *const u8) -> *mut u8 {
    // yeee
    if to == 0 as *const u8 {
        return allocator::CUR_ADDR_PUB.load(Ordering::Relaxed) as *mut u8;
    }
    let data = allocator::CUR_ADDR_PUB.load(Ordering::Relaxed);
    allocator::expand_by((to as u64) - data);
    to as *mut u8
}

```
### ./kernel/src/prelude.rs
```rust
pub use crate::shittymutex::Mutex;
pub use crate::*;
pub use crate::{
    _ezy_static, counter, dbg, dprint, dprintln, ezy_static, input, io::Printer, print, println,
    testing,
};
pub use alloc::format;
pub use alloc::{boxed::Box, collections::*, collections::*, string::*, vec, vec::Vec};
pub use core::sync::atomic::*;
pub use lazy_static::lazy_static;
pub use x86::io::*;
pub use x86_64::VirtAddr;
#[macro_export]
#[allow(non_upper_case_globals)]
macro_rules! ezy_static {
    { $name:ident, $type:ty, $init:expr } => {
        #[allow(non_upper_case_globals)]
        lazy_static! {
            pub static ref $name: Mutex<$type> = {
                Mutex::new($init)
            };
        }
    }
}

#[macro_export]
macro_rules! _ezy_static {
    { $name:ident, $type:ty, $init:expr } => {
        lazy_static! {
            static ref $name: Mutex<$type> = {
                Mutex::new($init)
            };
        }
    }
}

#[macro_export]
macro_rules! counter {
    ( $NAME: ident ) => {
        pub static COUNTER: AtomicUsize = AtomicUsize::new(0);
        struct $NAME;
        #[allow(dead_code)]
        impl $NAME {
            pub fn get(&self) -> usize {
                return COUNTER.load(Ordering::Relaxed);
            }
            pub fn inc(&self) -> usize {
                return COUNTER.fetch_add(1, Ordering::Relaxed) + 1;
            }
            pub fn reset(&self) -> usize {
                let old = self.get();
                COUNTER.store(0, Ordering::Relaxed);
                return old;
            }
        }
    };
}

```
### ./kernel/src/stack_canaries.rs
```rust
use crate::prelude::*;

ezy_static! { CANARIES, Vec<(VirtAddr, String, u64)>, vec![] }


pub fn add_canary(p: VirtAddr, s: String, size: u64) {
    let q = unsafe { p.as_ptr::<u8>() };
    unsafe {
        *(q as *mut u64) = 0xdeadbeef;
    }
    CANARIES.get().push((p, s.clone(), size));
}

extern {
    #[link_name = "llvm.returnaddress"]
    fn return_address(level: i32) -> *const u8;
}


pub fn stk_chk() {
    for c in CANARIES.get() {
        let q = unsafe { c.0.as_ptr::<u8>() };
        let stka = unsafe {
            *(q as *mut u64)
        };
        if stka != 0xdeadbeef {
            println!("=={}== ERROR: AddressSanitizer stack-overflow on address {:p} at pc {:p} stack {}", preempt::CURRENT_TASK.pid, q, unsafe { return_address(0) }, c.1);
            println!("SMASHED of size 8 at {:p} (now is {:#x?})", q, stka);
            println!("=={}== ABORTING", preempt::CURRENT_TASK.pid);
        }
    }
}
```
### ./kernel/src/userland.rs
```rust
use crate::{
    drive::{gpt::GetGPTPartitions, RODev},
    memory::map_to,
    prelude::*,
};
use kmacros::handle_read;
use x86_64::{
    structures::paging::{Mapper, Page, PageTableFlags, PhysFrame, Size4KiB},
    VirtAddr,
};
use xmas_elf::{self, program::Type};

fn read_from_user<T>(ptr: *mut T) -> &'static T {
    if ptr as u64 >= 0xFFFF800000000000 {
        unsafe { ptr.as_ref().unwrap() }
    } else {
        panic!("Security violation: read attempted from {:p}", ptr);
    }
}
pub fn ensure_region_safe(ptr: *mut u8, len: usize) {
    if (ptr as u64) < 0xFFFF800000000000 {
        panic!("Security violation: read attempted from {:p}", ptr);
    } else if (ptr as u64).overflowing_add(len as u64).0 < 0xFFFF800000000000 {
        panic!("Security violation: read attempted from {:p}", ptr);
    }
}
fn user_gets(mut ptr: *mut u8, n: u64) -> String {
    let mut s = "".to_string();
    unsafe {
        for _ in 0..n {
            s += &((*read_from_user(ptr)) as char).to_string();
            ptr = ptr.offset(1);
        }
    }
    s
}

ezy_static! { SVC_MAP, spin::Mutex<BTreeMap<String, u64>>, spin::Mutex::new(BTreeMap::new()) }
fn freebox1() {
    match preempt::CURRENT_TASK.box1 {
        Some(s) => {
            // free it
            preempt::CURRENT_TASK.get().box1 = None;
            unsafe {
                Box::from_raw(s as *const [u8] as *mut [u8]);
            }
        }
        None => {}
    };
}
fn freebox2() {
    match preempt::CURRENT_TASK.box2 {
        Some(s) => {
            // free it
            preempt::CURRENT_TASK.get().box2 = None;
            unsafe {
                Box::from_raw(s as *const [u8] as *mut [u8]);
            }
        }
        None => {}
    };
}
pub fn syscall_handler(sysno: u64, arg1: u64, arg2: u64) -> u64 {
    dprintln!(" ===> enter {} {:#x?}", preempt::CURRENT_TASK.pid, sysno);
    let v = match sysno {
        0 => {
            /* sys_exit */
            loop {
                preempt::yield_task();
            }
        }
        1 => {
            /* sys_bindbuffer */
            match preempt::CURRENT_TASK.box1 {
                Some(s) => {
                    // free it
                    preempt::CURRENT_TASK.get().box1 = None;
                    unsafe {
                        Box::from_raw(s as *const [u8] as *mut [u8]);
                    }
                }
                None => {}
            };
            let mut p = vec![];
            p.resize(arg2 as usize, 0);
            unsafe {
                accelmemcpy(p.as_mut_ptr(), arg1 as *const u8, arg2 as usize);
            }
            preempt::CURRENT_TASK.get().box1 = Some(Box::leak(p.into_boxed_slice()));
            0
        }
        2 => {
            /* sys_getbufferlen */
            match preempt::CURRENT_TASK.box1 {
                Some(s) => s.len() as u64,
                None => 0,
            }
        }
        3 => {
            /* sys_readbuffer */
            match preempt::CURRENT_TASK.box1 {
                Some(s) => {
                    unsafe {
                        accelmemcpy(arg1 as *mut u8, s.as_ptr(), s.len());
                    };
                    s.len() as u64
                }
                None => 0,
            }
        }
        4 => {
            /* sys_swapbuffers */
            let buf1 = preempt::CURRENT_TASK.box1;
            let buf2 = preempt::CURRENT_TASK.box2;
            preempt::CURRENT_TASK.get().box2 = buf1;
            preempt::CURRENT_TASK.get().box1 = buf2;
            0
        }
        5 => {
            /* sys_send */
            let target = user_gets(arg1 as *mut u8, arg2);
            if target == "kfs" {
                ksvc::dofs();
                dprintln!(" <=== exit {}", preempt::CURRENT_TASK.pid);
                return 0;
            }
            x86_64::instructions::interrupts::without_interrupts(|| {
                if ksvc::KSVC_TABLE.contains_key(&target) {
                    ksvc::KSVC_TABLE.get().get(&target).unwrap()();
                    dprintln!(" <=== exit {}", preempt::CURRENT_TASK.pid);
                    return;
                }
                let p = *SVC_MAP.lock().get(&target).unwrap();
                for r in preempt::TASK_QUEUE.get().iter_mut() {
                    if p == r.pid {
                        while !r.is_listening {
                            preempt::yield_task();
                        }
                        r.is_listening = false;
                        r.box1 = preempt::CURRENT_TASK.box1;
                        r.box2 = preempt::CURRENT_TASK.box2;
                        while !r.is_done {
                            preempt::yield_task();
                        }
                        freebox1();
                        freebox2();
                        preempt::CURRENT_TASK.get().box1 = r.box1;
                        preempt::CURRENT_TASK.get().box1 = r.box2;
                        r.box1 = None;
                        r.box2 = None;
                    }
                }
            });
            0
        }
        6 => {
            /* sys_listen */
            let name = user_gets(arg1 as *mut u8, arg2);
            preempt::CURRENT_TASK.get().is_listening = false;
            SVC_MAP.lock().insert(name, preempt::CURRENT_TASK.pid);
            // preempt::CURRENT_TASK.pid
            0
        }
        7 => {
            /* sys_accept */
            let nejm = user_gets(arg1 as *mut u8, arg2);
            assert_eq!(
                *SVC_MAP
                    .lock()
                    .get(&nejm)
                    .unwrap(),
                preempt::CURRENT_TASK.pid
            );
            preempt::CURRENT_TASK.get().is_done = false;
            preempt::CURRENT_TASK.get().is_listening = true;
            x86_64::instructions::interrupts::without_interrupts(|| {
                while preempt::CURRENT_TASK.get().is_listening {
                    preempt::yield_task()
                }
            });

            0
        }
        8 => {
            /* sys_exec */
            x86_64::instructions::interrupts::without_interrupts(|| {
                let l = preempt::CURRENT_TASK.box1;
                do_exec(l.unwrap());
            });
            0
        }
        9 => {
            /* sys_respond */
            x86_64::instructions::interrupts::without_interrupts(|| {
                preempt::CURRENT_TASK.get().is_done = true;
                preempt::yield_task();
                preempt::CURRENT_TASK.get().is_done = true;
            });
            0
        }
        10 => {
            /* sys_klog */
            print!("{}", user_gets(arg1 as *mut u8, arg2));
            0
        }
        11 => {
            /* sys_sbrk */
            let len = arg1;
            let oldbrk = preempt::CURRENT_TASK.program_break;
            let newbrk = ((oldbrk + len + 4095) / 4096) * 4096;
            preempt::CURRENT_TASK.get().program_break = newbrk;
            for i in 0..(((newbrk - oldbrk) / 4096) + 1) {
                let pageaddr = oldbrk + i * 4096;

                let data = crate::memory::mpage();
                // pages.push(data);
                if pageaddr < 0xFFFF800000000000 {
                    panic!("Invalid target for sbrk! {:#x?}", pageaddr);
                }
                map_to(
                    VirtAddr::from_ptr(data),
                    VirtAddr::new(pageaddr),
                    PageTableFlags::USER_ACCESSIBLE | PageTableFlags::WRITABLE,
                );
                preempt::yield_task();
            }
            dprintln!(" <=== exit {}", preempt::CURRENT_TASK.pid);
            return newbrk;
        }
        _ => (-1 as i64) as u64,
    };

    dprintln!(" <=== exit {}", preempt::CURRENT_TASK.pid);
    v
}
#[no_mangle]
unsafe extern "C" fn syscall_trampoline_rust(sysno: u64, arg1: u64, arg2: u64) -> u64 {
    syscall_handler(sysno, arg1, arg2)
}
extern "C" {
    static mut RSP_PTR: u64;
}
pub fn get_rsp_ptr() -> VirtAddr {
    unsafe { VirtAddr::new(RSP_PTR) }
}
pub fn set_rsp_ptr(va: VirtAddr) {
    unsafe {
        RSP_PTR = va.as_u64();
    }
}
pub fn alloc_rsp_ptr(stack_name: String) -> VirtAddr {
    const STACK_SIZE: usize = 4096 * 5;
    let stack_start = VirtAddr::from_ptr(crate::memory::malloc(STACK_SIZE));
    let stack_end = stack_start + STACK_SIZE;
    stack_canaries::add_canary(stack_start, stack_name, STACK_SIZE as u64);
    stack_end
}
pub fn init_rsp_ptr(stack_name: String) {
    set_rsp_ptr(alloc_rsp_ptr(stack_name));
}
#[naked]
pub unsafe fn new_syscall_trampoline() {
    asm!(
        "
        cli
        push rcx
        push r11
        push rbp
        mov rbp, rsp
        mov rsp, [RSP_PTR]
        push rbp
        push rbx
        push rcx
        push rdx
        push r12
        push r13
        push r14
        push r15
        mov rbp, rsp
        call syscall_trampoline_rust
        mov rsp, rbp
        pop r15
        pop r14
        pop r13
        pop r12
        pop rdx
        pop rcx
        pop rbx
        pop rbp
        mov rsp, rbp
        pop rbp
        pop r11
        pop rcx
    just_a_brk:
        sysretq
    .global RSP_PTR
    RSP_PTR:
        .space 0x8, 0x00
    "
    );
}
unsafe fn accelmemcpy(to: *mut u8, from: *const u8, size: usize) {
    x86_64::instructions::interrupts::without_interrupts(|| {
        if size < 8 {
            faster_rlibc::memcpy(to, from, size);
            return;
        }
        if size & 0x07 != 0 {
            faster_rlibc::memcpy(
                to.offset((size & 0xfffffffffffffff8) as isize),
                from.offset((size & 0xfffffffffffffff8) as isize),
                size & 0x07,
            );
        }
        faster_rlibc::fastermemcpy(to, from, size & 0xfffffffffffffff8);
    });
}
ezy_static! { PID_COUNTER, u64, 1 }
pub fn mkpid(ppid: u64) -> u64 {
    let r = (*PID_COUNTER.get() << 1) | (ppid & 1);
    *PID_COUNTER.get() += 1;
    r
}
pub fn getpid() -> u64 {
    preempt::CURRENT_TASK.pid
}
#[handle_read]
pub fn readfs(path: &str) -> &[u8] {
    panic!("asds");
}

pub fn loaduser() {
    init_rsp_ptr("syscall-stack:/bin/init".to_string());
    let loaded_init = readfs("/bin/init");
    let mut pages: Vec<*mut u8> = vec![];
    let exe = xmas_elf::ElfFile::new(&loaded_init).unwrap();
    let mut program_break: u64 = 0xFFFF800000000000;
    for ph in exe.program_iter() {
        if ph.get_type().unwrap() == Type::Load {
            let mut flags = PageTableFlags::NO_EXECUTE | PageTableFlags::PRESENT;
            // if ph.flags().is_read() {
            flags |= PageTableFlags::USER_ACCESSIBLE;
            // }
            // if ph.flags().is_execute() {
            flags ^= PageTableFlags::NO_EXECUTE;
            // }
            // if ph.flags().is_write() {
            flags |= PageTableFlags::WRITABLE;
            // }
            let page_count = (ph.file_size() + 4095 + (ph.virtual_addr() % 4096)) / 4096;
            for i in 0..page_count {
                let data = crate::memory::mpage();
                pages.push(data);
                if ph.virtual_addr() + (i * 4096) < 0xFFFF800000000000 {
                    panic!("Invalid target for ELF loader!");
                }
                map_to(
                    VirtAddr::from_ptr(data),
                    VirtAddr::new(ph.virtual_addr() + (i * 4096)),
                    flags,
                );
            }
            let maybe_new_program_break = ph.virtual_addr() + (page_count * 4096);
            program_break = if maybe_new_program_break < program_break {
                program_break
            } else {
                maybe_new_program_break
            };
            unsafe {
                accelmemcpy(
                    ph.virtual_addr() as *mut u8,
                    loaded_init.as_ptr().offset(ph.offset() as isize),
                    ph.file_size() as usize,
                );
            }
        }
    }
    // now initialize all the necessary fields.

    // To free just fpage() all of the `pages`
    preempt::CURRENT_TASK.get().pid = mkpid(preempt::CURRENT_TASK.pid);
    preempt::CURRENT_TASK.get().program_break = ((program_break + 4095) / 4096) * 4096;
    unsafe {
        jump_user(exe.header.pt2.entry_point());
    }
}

pub fn do_exec(kernel: &[u8]) {
    let slice = kernel.to_vec();
    let ve = preempt::CURRENT_TASK.get().box2.take();
    freebox1();
    freebox2();
    let path = String::from_utf8(slice).unwrap();
    let path2 = path.clone();
    preempt::task_alloc(move || unsafe {
        x86_64::instructions::interrupts::disable();
        let slice = readfs(&path);

        let ncr3 = main::forkp();
        let mut pages: Vec<*mut u8> = vec![];
        let exe = xmas_elf::ElfFile::new(&slice).unwrap();
        let mut program_break: u64 = 0xFFFF800000000000;
        for ph in exe.program_iter() {
            let mut flags = PageTableFlags::NO_EXECUTE | PageTableFlags::PRESENT;
            flags |= PageTableFlags::USER_ACCESSIBLE;
            flags ^= PageTableFlags::NO_EXECUTE;
            flags |= PageTableFlags::WRITABLE;
            let page_count = (ph.file_size() + 4095) / 4096;
            for i in 0..page_count {
                let data = crate::memory::mpage();
                pages.push(data);
                if ph.virtual_addr() + (i * 4096) < 0xFFFF800000000000 {
                    panic!("Invalid target for ELF loader!");
                }
                map_to(
                    VirtAddr::from_ptr(data),
                    VirtAddr::new(ph.virtual_addr() + (i * 4096)),
                    flags,
                );
            }
            let maybe_new_program_break = ph.virtual_addr() + (page_count * 4096);
            program_break = if maybe_new_program_break < program_break {
                program_break
            } else {
                maybe_new_program_break
            };
            accelmemcpy(
                ph.virtual_addr() as *mut u8,
                slice.as_ptr().offset(ph.offset() as isize),
                ph.file_size() as usize,
            );
        }
        x86_64::registers::control::Cr3::write(ncr3.0, ncr3.1);
        preempt::CURRENT_TASK.get().box1 = ve;
        preempt::CURRENT_TASK.get().pid = mkpid(preempt::CURRENT_TASK.pid);
        preempt::CURRENT_TASK.get().program_break = program_break;
        x86_64::instructions::interrupts::enable();
        jump_user(exe.header.pt2.entry_point());
    }, format!("syscall-stack:{}", path2.clone()));
}

unsafe fn jump_user(addr: u64) {
    asm!("
    mov ds,ax
    mov es,ax 
    mov fs,ax 
    mov gs,ax

    mov rsi, rsp
    push rax
    push rsi
    push 0x200
    push rdx
    push rdi
    iretq", in("rdi") addr, in("ax") 0x1b, in("dx") 0x23, in("rsi") 0);
    unreachable!();
}

```
### ./kernel/user/test.s
```null
[BITS 64]
%define sys_exit 0
%define sys_bindbuffer 1
%define sys_getbufferlen 2
%define sys_readbuffer 3
%define sys_swapbuffers 4
%define sys_send 5
%define sys_listen 6
%define sys_accept 7
%define sys_exec 8
%define sys_respond 9
%define sys_klog 10
%macro do_syscall 3
    push rcx
    push r11
    mov rdi, %1
    mov rsi, %2
    mov r8, %3
    syscall
    pop r11
    pop rcx
%endmacro
default rel
section .text
user:
    mov rsp, stack_bottom
    lea rax, [hello_world]
    do_syscall sys_klog, rax, 0
    jmp $
section .data
hello_world:
    db "Hello, userland world!", 0
align 8
stack_top:
    resb 4096
stack_bottom:
```
### ./kernel/user/user.ld
```null
SECTIONS {
    . = 0xFFFF800000000000;
    .text : { 
        *(.text)
    }
    . = ALIGN(4096);
    .data : { 
        *(.data)
    }
}
```
### ./kernel/.gdbinit
```{ "rs": "rust", "md": "md", "fish": "fish", "sh": "bash", "Makefile": "make", "js": "typescript", "ts": "typescript", "json": "json", "toml": "toml" }
set disassembly-flavor intel
add-symbol-file build/kernel.elf
add-symbol-file rootfs/bin/kinfo
target remote :1234
alias break=hbreak
c
```
### ./kernel/Cargo.toml
```toml
[package]
authors = ["pitust <stelmaszek.piotrpilot@gmail.com>"]
autobins = false
edition = "2018"
name = "an_os"
version = "0.1.0"

[dependencies]
acpi = "2.1.0"
bitflags = "1.2.1"
fontdue = "0.4.0"
linked_list_allocator = "0.8.0"
multiboot2 = "0.9.0"
pc-keyboard = "0.5.0"
pic8259_simple = "0.2.0"
serde_derive = "1.0.117"
spin = "0.5.2"
x86 = "0.34.0"
x86_64 = "0.12.0"
xmas-elf = "0.7.0"

[dependencies.conquer-once]
default-features = false
version = "0.2.0"

[dependencies.cpp_demangle]
default-features = false
features = ["alloc"]
version = "0.3.1"

[dependencies.crossbeam-queue]
default-features = false
features = ["alloc"]
version = "0.2.1"

[dependencies.fallible-iterator]
default-features = false
features = ["alloc"]
version = "0.2.0"

[dependencies.faster_rlibc]
path = "faster_rlibc"

[dependencies.font8x8]
default-features = false
features = ["unicode"]
version = "0.2"

[dependencies.gimli]
default-features = false
features = ["read"]
version = "0.23.0"

[dependencies.hex]
default-features = false
version = "0.4"

[dependencies.kmacros]
path = "kmacros"

[dependencies.lazy_static]
features = ["spin_no_std"]
version = "1.0"

[dependencies.libc]
default-features = false
version = "0.2.45"

[dependencies.log]
default-features = false
version = "0.4.11"

[dependencies.postcard]
features = ["alloc"]
version = "0.5.1"

[dependencies.ralloc]
default-features = false
features = ["unsafe_no_mutex_lock"]
git = "https://github.com/pitust/ralloc"

[dependencies.safety-here]
path = "safety-here"

[dependencies.serde]
default-features = false
features = ["alloc"]
version = "1.0.117"

[dependencies.serde_json]
default-features = false
features = ["alloc"]
version = "1.0.59"

[features]
conio = []
debug_logs = []
default = []
displayio = []
fini_exit = []
fini_wait = []

[lib]
crate-type = ["staticlib"]
path = "src/lib.rs"

[patch."https://github.com/pitust/ralloc".ralloc_shim]
path = "shm"

```
### ./kernel/Makefile
```{ "rs": "rust", "md": "md", "fish": "fish", "sh": "bash", "Makefile": "make", "js": "typescript", "ts": "typescript", "json": "json", "toml": "toml" }
run: build/oh_es.iso build/data.img
	qemu-system-x86_64 -hda build/oh_es.iso -s -debugcon file:logz.txt -global isa-debugcon.iobase=0x402 -accel kvm -cpu host -vnc :1 -monitor none -serial stdio
build/oh_es.iso: build/kernel.elf cfg/grub.cfg
	rm -rf iso
	mkdir -p iso/boot/grub
	cp cfg/grub.cfg iso/boot/grub
	cp build/kernel.elf iso/boot
	# grub-mkrescue -o build/oh_es.iso iso
	xorriso -as mkisofs -graft-points --modification-date=2020111621031700 \
		-b boot/grub/i386-pc/eltorito.img -no-emul-boot -boot-load-size 4 \
		-boot-info-table --grub2-boot-info --grub2-mbr \
		/usr/lib/grub/i386-pc/boot_hybrid.img -apm-block-size 2048 \
		--efi-boot efi.img -efi-boot-part --efi-boot-image --protective-msdos-label \
		-o build/oh_es.iso -r cfg/grub --sort-weight 0 / --sort-weight 1 /boot iso

build/kernel.elf: target/x86_64-unknown-none/debug/liban_os.a build/boot.o
	ld.lld target/x86_64-unknown-none/debug/liban_os.a /opt/cross/lib/gcc/x86_64-elf/10.2.0/libgcc.a --allow-multiple-definition -T/home/pitust/code/an_os/link.ld build/boot.o  -o build/kernel.elf -n
	grub-file --is-x86-multiboot2 build/kernel.elf
build/test.elf: build/test.o
	ld -T user/user.ld build/test.o -o build/test.elf
target/x86_64-unknown-none/debug/liban_os.a: faux build/test.elf
	cargo build --features "fini_exit debug_logs"
build/initrd.cpio: $(wildcard initrd/*) build/kernel.elf initrd/ksymmap.pcrd
	sh create-initrd.sh
initrd/ksymmap.pcrd: build/ksymmap.pcrd
	cp build/ksymmap.pcrd initrd
build/ksymmap.pcrd: build/kernel.elf
	ts-node -T sym-city.ts
build/boot.o: asm/boot.s
	nasm asm/boot.s -f elf64 -o build/boot.o
build/test.o: user/test.s
	nasm -felf64 user/test.s -o build/test.o
build/data.img: build/data.note $(shell find rootfs)
	mke2fs \
		-L 'ohes-sysroot' \
		-N 0 \
		-O ^64bit \
		-d rootfs \
		-m 5 \
		-r 1 \
		-t ext2 \
		"build/ext.img" \
		128M

	dd conv=notrunc if=build/ext.img of=build/data.img bs=512 seek=4048 status=progress
build/data.note:
	dd if=/dev/zero bs=512K count=280 of=build/data.img status=progress
	dd if=/dev/zero bs=512K count=256 of=build/ext.img status=progress
	sfdisk build/data.img <cfg/disklayout.sfdsk
	touch build/data.note
faux:
ffonts:
	wget https://fonts.gstatic.com/s/robotomono/v12/L0xuDF4xlVMF-BfR8bXMIhJHg45mwgGEFl0_3vq_S-W4Ep0.woff2 -Ofonts/roboto.woff2
	woff2_decompress fonts/roboto.woff2
.PHONY: faux run ffonts
```
### ./kernel/doc.txt
```null
IF PE = 0
    THEN GOTO REAL-ADDRESS-MODE; # false, PE = 1 LMA = 1
ELSIF (IA32_EFER.LMA = 0)
    THEN
        IF (EFLAGS.VM = 1)
                THEN GOTO RETURN-FROM-VIRTUAL-8086-MODE;
                ELSE GOTO PROTECTED-MODE;
        FI;
    ELSE GOTO IA-32e-MODE; #us
FI;
TASK-RETURN: (* PE = 1, VM = 0, NT = 1 *)
    SWITCH-TASKS (without nesting) to TSS specified in link field of current TSS;
    Mark the task just abandoned as NOT BUSY;
    IF EIP is not within CS limit
        THEN #GP(0); FI;
END;
RETURN-TO-VIRTUAL-8086-MODE:
    (* Interrupted procedure was in virtual-8086 mode: PE = 1, CPL=0, VM = 1 in flag image *)
    IF EIP not within CS limit
        THEN #GP(0); FI;
    EFLAGS ← tempEFLAGS;
    ESP ← Pop();
    SS ← Pop(); (* Pop 2 words; throw away high-order word *)
    ES ← Pop(); (* Pop 2 words; throw away high-order word *)
    DS ← Pop(); (* Pop 2 words; throw away high-order word *)
    FS ← Pop(); (* Pop 2 words; throw away high-order word *)
    GS ← Pop(); (* Pop 2 words; throw away high-order word *)
    CPL ← 3;
    (* Resume execution in Virtual-8086 mode *)
END;
PROTECTED-MODE-RETURN: (* PE = 1 *)
    IF CS(RPL) > CPL
        THEN GOTO RETURN-TO-OUTER-PRIVILEGE-LEVEL;
        ELSE GOTO RETURN-TO-SAME-PRIVILEGE-LEVEL; FI;
END;
RETURN-TO-OUTER-PRIVILEGE-LEVEL:
    IF OperandSize = 32
        THEN
                ESP ← Pop();
                SS ← Pop(); (* 32-bit pop, high-order 16 bits discarded *)
    ELSE IF OperandSize = 16
        THEN
                ESP ← Pop(); (* 16-bit pop; clear upper bits *)
                SS ← Pop(); (* 16-bit pop *)
        ELSE (* OperandSize = 64 *)
                RSP ← Pop();
                SS ← Pop(); (* 64-bit pop, high-order 48 bits discarded *)
    FI;
    IF new mode ≠ 64-Bit Mode
        THEN
                IF EIP is not within CS limit
                        THEN #GP(0); FI;
        ELSE (* new mode = 64-bit mode *)
                IF RIP is non-canonical
                            THEN #GP(0); FI;
    FI;
    EFLAGS (CF, PF, AF, ZF, SF, TF, DF, OF, NT) ← tempEFLAGS;
    IF OperandSize = 32 or or OperandSize = 64
        THEN EFLAGS(RF, AC, ID) ← tempEFLAGS; FI;
    IF CPL ≤ IOPL
        THEN EFLAGS(IF) ← tempEFLAGS; FI;
    IF CPL = 0
        THEN
                EFLAGS(IOPL) ← tempEFLAGS;
                IF OperandSize = 32 or OperandSize = 64
                        THEN EFLAGS(VIF, VIP) ← tempEFLAGS; FI;
    FI;
    CPL ← CS(RPL);
    FOR each SegReg in (ES, FS, GS, and DS)
        DO
                tempDesc ← descriptor cache for SegReg (* hidden part of segment register *)
                IF (SegmentSelector == NULL) OR (tempDesc(DPL) < CPL AND tempDesc(Type) is (data or non-conforming code)))
                        THEN (* Segment register invalid *)
                            SegmentSelector ← 0; (*Segment selector becomes null*)
                FI;
        OD;
END;
RETURN-TO-SAME-PRIVILEGE-LEVEL: (* PE = 1, RPL = CPL *)
    IF new mode ≠ 64-Bit Mode
        THEN
                IF EIP is not within CS limit
                        THEN #GP(0); FI;
        ELSE (* new mode = 64-bit mode *)
                IF RIP is non-canonical
                            THEN #GP(0); FI;
    FI;
    EFLAGS (CF, PF, AF, ZF, SF, TF, DF, OF, NT) ← tempEFLAGS;
    IF OperandSize = 32 or OperandSize = 64
        THEN EFLAGS(RF, AC, ID) ← tempEFLAGS; FI;
    IF CPL ≤ IOPL
        THEN EFLAGS(IF) ← tempEFLAGS; FI;
    IF CPL = 0
            THEN
                    EFLAGS(IOPL) ← tempEFLAGS;
                    IF OperandSize = 32 or OperandSize = 64
                        THEN EFLAGS(VIF, VIP) ← tempEFLAGS; FI;
    FI;
END;
IA-32e-MODE:
    IF NT = 1
        THEN #GP(0);
    ELSE IF OperandSize = 32
        THEN
                EIP ← Pop();
                CS ← Pop();
                tempEFLAGS ← Pop();
        ELSE IF OperandSize = 16
                THEN
                        EIP ← Pop(); (* 16-bit pop; clear upper bits *)
                        CS ← Pop(); (* 16-bit pop *)
                        tempEFLAGS ← Pop(); (* 16-bit pop; clear upper bits *)
                FI;
        ELSE (* OperandSize = 64 *) # us
                THEN
                            RIP ← Pop();
                            CS ← Pop(); (* 64-bit pop, high-order 48 bits discarded *)
                            tempRFLAGS ← Pop();
    FI;
    IF CS.RPL > CPL
        THEN GOTO RETURN-TO-OUTER-PRIVILEGE-LEVEL;
        ELSE
                IF instruction began in 64-Bit Mode
                        THEN
                            IF OperandSize = 32
                                THEN
                                    ESP ← Pop();
                                    SS ← Pop(); (* 32-bit pop, high-order 16 bits discarded *)
                            ELSE IF OperandSize = 16
                                THEN
                                    ESP ← Pop(); (* 16-bit pop; clear upper bits *)
                                    SS ← Pop(); (* 16-bit pop *)
                                ELSE (* OperandSize = 64 *)
                                    RSP ← Pop();
                                    SS ← Pop(); (* 64-bit pop, high-order 48 bits discarded *)
                            FI;
                FI;
                GOTO RETURN-TO-SAME-PRIVILEGE-LEVEL; FI;
END;
```
### ./kernel/link.ld
```null
ENTRY(_start)
 
/* Tell where the various sections of the object files will be put in the final
   kernel image. */
SECTIONS
{
	/* Begin putting sections at 1 MiB, a conventional place for kernels to be
	   loaded at by the bootloader. */
	. = 1M;
 
	/* First put the multiboot header, as it is required to be put very early
	   early in the image or the bootloader won't recognize the file format.
	   Next we'll put the .text section. */
	.text : ALIGN(4K)
	{
		*(.multiboot_header)
		*(.text)
		*(.text.*)
	}
 
	/* Read-only data. */
	.rodata : ALIGN(4K)
	{
		*(.rodata)
		*(.rodata.*)
	}
 
	/* Read-write data (initialized) */
	.data : ALIGN(4K)
	{
		*(.data)
		*(.data.*)
	}
 
	/* Read-write data (uninitialized) and stack */
	.bss : ALIGN(4K)
	{
		*(.bss)
		*(.bss.*)
	}
	.got : ALIGN(4K)
	{
		*(.got)
		*(.got.*)
	}
	es = .;
	.eh_frame : ALIGN(4K)
	{
		*(.eh_frame)
		*(.eh_frame.*)
	}
	ee = .;
	esz = ee - es;
 
	/* The compiler may produce other sections, by default it will put them in
	   a segment with the same name. Simply add stuff here as needed. */
}
```
### ./kernel/rootfs/etc/init.rc
```null
{
    "_init": {
        "run": [
            "fail",
            "-error",
            "_init is a meta-task that is launched by the init program itself"
        ],
        "use_fs": "kfs"
    },
    "target:boot": {
        "trigger_after": "_init",
        "run": [
            "kinfo",
            "Booted Oh Es!"
        ],
        "wants": [
            "info-fun"
        ]
    },
    "info-fun": {
        "run": [
            "kinfo",
            "\u001b[44m\u001b[30m ~ \u001b[0m\u001b[34m\ue0b0\u001b[0m I want powerline fonts here!"
        ]
    }
}
```
### ./kernel/shm/Cargo.toml
```toml
[package]
name = "ralloc_shim"
version = "0.1.1"
authors = ["pitust <piotr@stelmaszek.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
[dependencies]
log = { version = "0.4.11", default-features = false }

```
### ./kernel/shm/src/lib.rs
```rust
#![no_std]

pub mod thread_destructor {
    pub fn register(t: *mut u8, dtor: unsafe extern "C" fn(*mut u8)) {}
}
pub mod config {
    pub fn default_oom_handler() {}
    pub const OS_MEMTRIM_LIMIT: usize = 0x1000000;
    pub const OS_MEMTRIM_WORTHY: usize = 0;
    pub const LOCAL_MEMTRIM_STOP: usize = 1024;
    pub const LOCAL_MEMTRIM_LIMIT: usize = 1024;
    pub const FRAGMENTATION_SCALE: usize = 10;
    pub const MIN_LOG_LEVEL: u8 = 0;

    pub fn extra_brk(size: usize) -> usize {
        // TODO: Tweak this.
        /// The BRK multiplier.
        ///
        /// Alignment
        const MULTIPLIER: usize = 1;

        (size + MULTIPLIER - 1) / MULTIPLIER * MULTIPLIER - size
    }
    /// Canonicalize a fresh allocation.
    ///
    /// The return value specifies how much _more_ space is requested to the fresh allocator.
    // TODO: Move to shim.
    #[inline]
    pub fn extra_fresh(size: usize) -> usize {
        /// The multiplier.
        ///
        /// The factor determining the linear dependence between the minimum segment, and the acquired
        /// segment.
        const MULTIPLIER: usize = 2;
        /// The minimum extra size to be BRK'd.
        const MIN_EXTRA: usize = 64;
        /// The maximal amount of _extra_ bytes.
        const MAX_EXTRA: usize = 1024;

        core::cmp::max(MIN_EXTRA, core::cmp::min(MULTIPLIER * size, MAX_EXTRA))
    }
    pub fn log(s: &str) -> i32 {
        unsafe {
            crate::syscalls::out(s);
        }
        0
    }
}
pub mod syscalls {

    extern "C" {
        pub fn brk(to: *const u8) -> *mut u8;
    }
    extern "Rust" {
        pub fn out(s: &str);
    }
}

```
### ./README.md
```md
# Oh Es Root
![Banner](logo.png)
This is the main repo for the oh es operating system. This is mostly a container for a bunch of submodules.i
## Developmnet
The kernel uses GNU Make and Cargo as its build system. It requires nightly rust, nasm, llvm and GNU ld cross compiled for x86-64 and libgcc cross compiled for x86-64 at /opt/cross.
## Userland
The userland is built with a custom build system, ohbuild. It requires rust (the exact same nightly as kernel is tested) and GNU ld cross compiled for x86-64. Installing is done by navigating into ohbuild submodule and executing `cargo install --path .`. Then, ohbuild can be invoked by using `ohbuild --path path/to/cache/dir --out path/to/out/dir`.
## Updating submodules
There is a handy oneliner: 
```
git submodule update --remote kernel ohbuild && git add . && git commit -s -m "Update submodules" && git push origin main
```
## Generating code
Use `csa`. The name is _borrowed_ from v8's builtin codegen thing, CodeStubAssembler. You need `jq` for that though.

```
### ./ohprogs/Cargo.toml
```toml
[package]
name = "ohes_init"
version = "0.1.0"
authors = ["pitust <piotr@stelmaszek.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
liboh = { path = "liboh" }
rlibc = "1.0.0"
postcard = { version = "0.5.1", features = ["alloc"] }
serde = { version = "1.0.117", default-features = false, features = ["alloc"] }
serde_json = { version = "1.0.59", default-features = false, features = ["alloc"] }
serde_derive = "1.0.117"

[[bin]]
path = "src/init.rs"
name = "init"

[[bin]]
path = "src/kinfo.rs"
name = "kinfo"


[[bin]]
path = "src/shtdwn.rs"
name = "shutdown"

```
### ./ohprogs/Makefile
```{ "rs": "rust", "md": "md", "fish": "fish", "sh": "bash", "Makefile": "make", "js": "typescript", "ts": "typescript", "json": "json", "toml": "toml" }
BINS = $(wildcard src/*.rs)
BIN_TGDS = $(patsubst src/%.rs,rootfs/bin/%,$(BINS))
build: $(BIN_TGDS)
# .PHONY: build
rootfs/bin/%: src/%.rs
	mbcc $<
```
### ./ohprogs/src/init.rs
```rust
#![feature(default_alloc_error_handler)]
#![feature(box_syntax)]
#![no_std]
#![no_main]
#![feature(alloc_prelude)]

use core::{alloc::Layout, fmt::Write};
#[macro_use]
use liboh::prelude::*;

pub use alloc::{
    borrow, boxed::Box, collections, collections::*, collections::*, fmt, format, prelude::v1::*,
    slice, string::*, vec, vec::Vec,
};
pub use core::sync::atomic::*;

extern crate alloc;
extern crate rlibc;
use postcard;
use serde_derive::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum KSvcResult {
    Success,
    Failure(String),
}

#[derive(Serialize, Deserialize, Clone)]
pub enum FSOp {
    Read,
    ReadDir,
    Stat,
}
#[derive(Serialize, Deserialize, Clone)]
pub enum FSResult {
    Text(Vec<u8>),
    Dirents(Vec<String>),
    Stats(u16),
    Failure(String),
}

#[derive(Serialize, Deserialize, Debug)]
struct Node {
    run: Vec<String>,
    trigger_after: Option<String>,
    wants: Option<Vec<String>>,
    use_fs: Option<String>,
    with_fs: Option<String>,
    provides: Option<String>,
}

fn read_file(s: String) -> String {
    let t: FSResult = liboh::service::request("kfs", (FSOp::Read, s));
    let p = match t {
        FSResult::Text(p) => p,
        _ => unreachable!(),
    };
    String::from_utf8(p).unwrap()
}

fn readbin(s: String) -> Vec<u8> {
    let t: FSResult = liboh::service::request("kfs", (FSOp::Read, s));
    let p = match t {
        FSResult::Text(p) => p,
        _ => unreachable!(),
    };
    p
}

fn we_did_task(
    rq: &mut VecDeque<String>,
    done: &mut BTreeSet<String>,
    r: &BTreeMap<String, Node>,
    t: String,
) {
    println!("Done {}", t);
    done.insert(t.clone());
    match &r[&t].provides {
        Some(p) => {
            done.insert(p.clone());
        }
        None => {}
    }
    for k in r {
        match &k.1.trigger_after {
            Some(p) => {
                if p == &t {
                    rq.push_back(k.0.clone())
                }
            }
            None => {}
        }
    }
}

fn do_task(
    r: String,
    rq: &mut VecDeque<String>,
    done: &mut BTreeSet<String>,
    pr: &BTreeMap<String, Node>,
) {
    if done.contains(&r) {
        return;
    }
    let mut can_do_now = true;
    if !pr.contains_key(&r) {
        for k in pr {
            match &k.1.provides {
                Some(u) => {
                    if u == &r {
                        rq.push_front(k.0.clone());
                        return;
                    }
                }
                None => {}
            }
        }
        println!("[ERR] Cannot find unknown task {}, skipping!", r);
        done.insert(r.clone());
        return;
    }
    match &pr[&r].wants {
        Some(p) => {
            for w in p {
                if done.contains(w) {
                    continue;
                }
                can_do_now = false;
                rq.push_back(w.clone());
            }
        }
        None => {}
    };
    if can_do_now {

        liboh::exec(pr[&r].run.clone());
        we_did_task(rq, done, pr, r);
    } else {
        rq.push_back(r);
    }
}

pub fn enforce<T>(s: serde_json::Result<T>) -> T {
    match s {
        Ok(p) => p,
        Err(f) => {
            panic!(
                "Failed reading init.rc:\n At {}:{} {}",
                f.line(),
                f.column(),
                f.to_string()
            );
        }
    }
}

pub fn main_fn() {
    // "\ue0b0"
    println!(":: read init.rc...");
    // this is a hack to allow easier testing
    let txt = read_file("etc/init.rc".to_string());
    let p: BTreeMap<String, Node> = enforce(serde_json::from_str(&txt));
    let mut q = VecDeque::new();
    let mut done = BTreeSet::new();
    q.push_back("_init".to_string());
    we_did_task(&mut q, &mut done, &p, "_init".to_string());
    liboh::syscall::sys_listen("initd");
    loop {
        while q.len() != 0 {
            do_task(q.pop_front().unwrap(), &mut q, &mut done, &p);
        }
        
        liboh::service::accept("initd", |x| {
            q.push_back(x);
            ()
        })
    }
}
main!(main_fn);

```
### ./ohprogs/src/kinfo.rs
```rust
#![feature(default_alloc_error_handler)]
#![feature(box_syntax)]
#![no_std]
#![no_main]
#![feature(alloc_prelude)]

use core::{alloc::Layout, fmt::Write};
use liboh::prelude::*;

pub use alloc::{
    borrow, boxed::Box, collections, collections::*, collections::*, fmt, format, prelude::v1::*,
    slice, string::*, vec, vec::Vec,
};
pub use core::sync::atomic::*;

extern crate alloc;
extern crate rlibc;
use postcard;

#[macro_export]
macro_rules! println {
    ($($tail:tt)*) => { writeln!(liboh::klog::KLog, $($tail)*).unwrap(); }
}
fn read_buf<'a, U: serde::Deserialize<'a> + Clone>() -> U {
    let l = liboh::syscall::sys_getbufferlen();
    let a = unsafe { alloc::alloc::alloc(Layout::from_size_align(l as usize, 8).unwrap()) };
    let slc = unsafe { core::slice::from_raw_parts_mut(a, l as usize) };
    liboh::syscall::sys_readbuffer(slc);
    let x: U = postcard::from_bytes::<'a, U>(slc).unwrap().clone();
    unsafe { alloc::alloc::dealloc(a, Layout::from_size_align(l as usize, 8).unwrap()) }
    x
}

pub fn main_fn() {
    // "\ue0b0"
    println!("[info] {}", read_buf::<Vec<String>>()[1]);
}
main!(main_fn);

```
### ./ohprogs/src/shtdwn.rs
```rust
#![feature(default_alloc_error_handler)]
#![feature(box_syntax)]
#![no_std]
#![no_main]
#![feature(alloc_prelude)]

use core::{alloc::Layout, fmt::Write};
use liboh::prelude::*;

pub use alloc::{
    borrow, boxed::Box, collections, collections::*, collections::*, fmt, format, prelude::v1::*,
    slice, string::*, vec, vec::Vec,
};
pub use core::sync::atomic::*;

extern crate alloc;
extern crate rlibc;
use postcard;
use serde_derive::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum KIOOpResult {
    Success,
    ReadResultByte(u8),
    ReadResultWord(u16),
    ReadResultDWord(u32),
    ReadResultQWord(u64),
    Failure(String),
}
#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum IOOpData {
    WriteByte(u8),
    WriteWord(u16),
    WriteDWord(u32),
    WriteQWord(u64),

    ReadByte(),
    ReadWord(),
    ReadDWord(),
    ReadQWord(),
}

#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct IOOp {
    pub port: u16,
    pub data: IOOpData,
}

#[macro_export]
macro_rules! println {
    ($($tail:tt)*) => { writeln!(liboh::klog::KLog, $($tail)*).unwrap(); }
}

pub fn main_fn() {
    // "\ue0b0"
    println!("ohes is shutting down...");

    println!(" ---- qemu method ----");
    let _: KIOOpResult = liboh::service::request(
        "kio",
        IOOp {
            port: 0x604,
            data: IOOpData::WriteWord(0x2000),
        },
    );
    println!(" ---- bochs/old qemu method ----");
    let _: KIOOpResult = liboh::service::request(
        "kio",
        IOOp {
            port: 0xB004,
            data: IOOpData::WriteWord(0x2000),
        },
    );
    println!(" ---- vbox method ----");
    let _: KIOOpResult = liboh::service::request(
        "kio",
        IOOp {
            port: 0x4004,
            data: IOOpData::WriteWord(0x3400),
        },
    );
    println!(" AYYY, get a better vm will ya? i can't power off!")
}
main!(main_fn);

```
### ./csa
```bash
#!/bin/bash
set -e
if [ $1 = _createf ]; then
    FILE=$2
    if [ -d $FILE ]; then
        exit
    fi
    EXT=$(basename $FILE | tr '.' ' ' | awk '{print($2)}')
    LANG=$(cat langmatrix.json | jq -r ".$EXT")
    echo "### $FILE"
    echo "\`\`\`"$LANG
    cat $FILE
    echo -e "\n\`\`\`"
else
    rm -f out.md
    find | grep -vP '(git|build|target|logo|lock)' | xargs -n 1 ./csa _createf >out.md
fi

```
### ./langmatrix.json
```json
{
    "rs": "rust",
    "md": "md",
    "fish": "fish",
    "sh": "bash",
    "Makefile": "make",
    "js": "typescript",
    "ts": "typescript",
    "json": "json",
    "toml": "toml"
}
```
