# Kernel_Prog_Linux

## 1. Linking multiple source files


## 2. Module Stacking

- It provides library like feature to kernel module authors.
- It will include data structures and functionalities (functions/APIs) that will be exported to other kernel modules in the project (or who wants to use them).

Examples::
Inside VM
```
$ lsmod | grep vbox
vboxnetadp          28672   0
vboxnetfit          28672   1
vboxdrv             614400  3 vvboxnetadp, vboxnetfit
```
- Third column is usage count, any modules after 3rd column are module dependencies.
- Kernel module on the right depend on the kernel module on the left.
- So, vboxnetadp, vboxnetfit depends on vboxdrv
- usage count or reference count.
- As we see usage count is 3, the kernel modules that depend on it are stacked on top of it!!!
```
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ lsmod | awk '$3 > 0 {print $0}' | sort -k3n
Module                  Size  Used by
aes_generic            32768  1 aes_arm64
algif_hash             12288  1
algif_skcipher         12288  1
binfmt_misc            16384  1
brcmfmac              348160  1 brcmfmac_wcc
.
.
.
snd_soc_hdmi_codec     20480  2
v4l2_mem2mem           45056  2 bcm2835_codec,rpivid_hevc
vc_sm_cma              28672  2 bcm2835_mmal_vchiq,bcm2835_isp
videobuf2_memops       12288  2 videobuf2_vmalloc,videobuf2_dma_contig
aes_arm64              12288  3
bcm2835_mmal_vchiq     36864  3 bcm2835_codec,bcm2835_v4l2,bcm2835_isp
cmac                   12288  3
libaes                 12288  3 aes_arm64,bluetooth,aes_generic
```

- So basically, we can't apply rmmod on a kernel module if it's usage count is non-zero.

Example 2: Here user module depends on core module
```
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo insmod ./user_lkm.ko 
sudo: unable to resolve host pi: Name or service not known
insmod: ERROR: could not insert module ./user_lkm.ko: Unknown symbol in module
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ ls
core_lkm.c   core_lkm.mod    core_lkm.mod.o  Makefile       Module.symvers  user_lkm.ko   user_lkm.mod.c  user_lkm.o
core_lkm.ko  core_lkm.mod.c  core_lkm.o      modules.order  user_lkm.c      user_lkm.mod  user_lkm.mod.o
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo insmod core_lkm.ko 
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ lsmod | grep core_lkm
core_lkm               12288  0
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo insmod user_lkm.ko
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ lsmod | egrep "core_lkm|user_lkm"
user_lkm               12288  0
core_lkm               12288  1 user_lkm
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ dmesg
[ 3802.914192] user_lkm: Unknown symbol exp_int (err -2)
[ 3802.914227] user_lkm: Unknown symbol get_skey (err -2)
[ 3802.914249] user_lkm: Unknown symbol llkd_sysinfo2 (err -2)
[ 3847.536414] core_lkm:core_lkm_init(): inserted
               Exported: get_skey(), llkd_sysinfo2() and exp_int
[ 4126.818740] user_lkm:user_lkm_init(): inserted
[ 4126.818760] core_lkm:get_skey(): /home/wiki/Documents/Linux-Kernel-Programming_2E/ch5/modstacking/core_lkm.c:120: I've been called
[ 4126.818770] user_lkm:user_lkm_init(): Called get_skey(), ret = 0x123abc567def = 20043477188079
[ 4126.818778] user_lkm:user_lkm_init(): exp_int = 200
[ 4126.818785] core_lkm:llkd_sysinfo2(): llkd_sysinfo2(): minimal Platform Info:
               CPU: Aarch64, little-endian; 64-bit OS.
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo rmmod core_lkm 
rmmod: ERROR: Module core_lkm is in use by: user_lkm
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo rmmod core_lkm 
rmmod: ERROR: Module core_lkm is in use by: user_lkm
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo dmesg -C
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo rmmod core_lkm 
rmmod: ERROR: Module core_lkm is in use by: user_lkm
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo rmmod user_lkm 
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ lsmod | egrep "core_lkm|user_lkm"
core_lkm               12288  0
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo rmmod core_lkm 
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ lsmod | egrep "core_lkm|user_lkm"
wiki@pi:~/Documents/Linux-Kernel-Programming_2E/ch5/modstacking $ sudo dmesg
[ 4464.057569] user_lkm:user_lkm_exit(): bids you adieu
[ 4487.063682] core_lkm:core_lkm_exit(): bids you adieu
```

## Module Parameters:: Passing to kernel module

- It's helpful in debugging, setting debug_level (default = 0, nothing will appear).
- Use mp_debug_level (module parameter)
- But how does it work ? There is no main with arguements, it's simply a linker thing, declare your intended mp as a global static variables, then specify to the build system by using module_param() macro.

```
/* Module parameters */
static int mp_debug_level;
module_param(mp_debug_level, int, 0660);
```
- Don't inititalize static int -> 0
- module_param(varibale static, data_type, permissions) // 0660, sysfs visibility
