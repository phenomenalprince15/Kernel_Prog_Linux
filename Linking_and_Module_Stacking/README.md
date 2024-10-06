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

### module_param(varibale static, data_type, permissions) // 0660, sysfs visibility

- Without any parameters, debug level zero.
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ modinfo -p modparams1.ko 
mp_debug_level:Debug level [0-2]; 0 => no debug messages, 2 => high verbosity (int)
mp_strparam:A demo string parameter (charp)
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo dmesg -C
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo insmod modparams1.ko 
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo dmesg
[ 2646.142087] modparams1: loading out-of-tree module taints kernel.
[ 2646.142979] modparams1:modparams1_init(): inserted
[ 2646.142994] modparams1:modparams1_init(): module parameters passed: mp_debug_level=0 mp_strparam=My string param
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ 
```

- with paramemetrs passing
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo insmod modparams1.ko mp_debug_level=2 mp_strparam=\"hello modparams1\"
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo dmesg
[ 2945.172643] modparams1:modparams1_init(): inserted
[ 2945.172667] modparams1:modparams1_init(): module parameters passed: mp_debug_level=2 mp_strparam=hello modparams1
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo rmmod modparams1 
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo dmesg
[ 2945.172643] modparams1:modparams1_init(): inserted
[ 2945.172667] modparams1:modparams1_init(): module parameters passed: mp_debug_level=2 mp_strparam=hello modparams1
[ 2963.363710] modparams1:modparams1_exit(): module parameters passed: mp_debug_level=2 mp_strparam=hello modparams1
[ 2963.363731] modparams1:modparams1_exit(): removed
```

#### File is generated inside /sys/module/(module_name)
- charp is character pointer
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ sudo insmod modparams1.ko 
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ ls /sys/module/modparams1/
coresize  holders  initsize  initstate  notes  parameters  refcnt  sections  srcversion  taint  uevent  version
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams1$ ls /sys/module/modparams1/parameters/
mp_debug_level  mp_strparam
```

#### What if user must pass params explicitly

- It is handled with parameter static int "control_freak"
```
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams2$ sudo insmod modparams2.ko 
insmod: ERROR: could not insert module modparams2.ko: Invalid parameters
wiki@pi:~/Linux-Kernel-Programming_2E/ch5/modparams/modparams2$ sudo dmesg
[  833.958088] modparams2: loading out-of-tree module taints kernel.
[  833.958671] modparams2:modparams2_init(): inserted
[  833.958679] modparams2:modparams2_init(): I'm a control freak; thus, you *Must* pass along module parameter 'control_freak', value in the range [1-5]; aborting...
```

#### Additional modules
```
module_param_cb() // for callback
module_param_desc() // for description
```

#### Overriding the module's parameters name
```
module_param_named(current_allocated_bytes, dm_bufio_current_allocated, ulong, S_IRUGO)
MODULE_PARAM_DESC(current_allocated_bytes, "Memory currently used by the cache")
```
current_allocated_bytes -> alias or name override of internal variable in dm_bufio_current_allocated


#### Hardware related kernel parameters

- It has a seperate macro: module_param_hw[_named|array]()
- https://lwn.net/Articles/708274/
- Module params that specify hardare params: io ports, iomem addresses, irqs, dma channels, fixed dma buffers and other types.
- This enables params to be locked down in the core parameter parser for secure boot support.

### Floating point not allowed in kernel
- well then how can we do FP operations ?
- Use, user-space mode, take help from there.
- APIs like call_usermodehelper*()
- Also, there exists a way to do forcing FP in the kernel, put your FP code between kernel_fpu_begin() and kernel_fpu_end() macros.
- It's used in crypto, AES, CRC and others...
- You'll end up with Call Trace in dmesg.

### Auto-loading of modules at boot
```
install:
        sudo make -C $(KDIR) M=$(PWD) modules_install

sudo modprobe module_name
# modprobe is intelligent version of insmode
```
- Files are stores in order at modules.order it will be installed at /lib/modules/$(uname -r)/extra or /updates (in arm)
- depmod creates modules.dep files (it arranges the modules) after modules are installed into kernel, it takes care of it.
```
wiki@pi:/lib/modules/6.8.0-1012-raspi/updates$ grep user_lkm /lib/modules/$(uname -r)/* 2>/dev/null
/lib/modules/6.8.0-1012-raspi/modules.dep:updates/user_lkm.ko: updates/core_lkm.ko
```
- modules.symbols has information on all exported module symbols, similarly modules.symvers
- we can /proc/kallsyms pseduofile, all symbols in kernel symbol table - exported and private along with their virtual addresses are seen here. (try sudo cat /proc/kallsyms)

- Only modern linux, e have systemd taking care of auto-loading kernel modules by parsing the content of files: /etc/modules-load.d/*
(responsible for systemd-modules-load.service)
- if any module misbehaves, we can blacklist it in /etc/modules-load.d
```
module_blacklist= ....
```
