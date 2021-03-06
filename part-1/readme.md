# Part 1 - Getting Started

Although the Raspberry-Pi comes with a good Linux distribution, the Pi is about software
development, and sometimes we want a real-time system without an operating system. I
decided it'd be great to do a tutorial outside of Linux to get to the resources of this great
piece of hardware in a similar vein to the
[Cambridge University Tutorials](http://www.cl.cam.ac.uk/freshers/raspberrypi/tutorials/)
which are excellently written. However, they don't create an OS as purported and they start
from assembler rather than C. I will simply mimic their tutorial here, but using C instead
of assembler. The C compiler simply converts C syntax to assembler and then assembles this
into executable code for us anyway.

I highly recommend going through the Cambridge University Raspberry Pi tutorials as they are
excellent. If you want to learn a bit of assembler too, then definitely head off to there!
These pages provide a similar experience, but with the additional of writing code in C and
understanding the process behind that.

## Cross -Compiling for the Raspberry Pi (BCM2835/6)

The [GCC ARM Embedded](https://launchpad.net/gcc-arm-embedded) project on Launchpad gives us
a GCC toolchain to use for ARM compilation on either Windows or Linux. I suggest you install
it (on Windows I would install it somewhere without spaces in the file path) so that you
have it in your path. On Linux install the relevant package which is usually something like
`gcc-arm-none-eabi` and also the debugger package `gdb-arm-none-eabi`. You should be able to
type `arm-none-eabi-gcc` on the command line and get a response like the following:

```
>arm-none-eabi-gcc
arm-none-eabi-gcc: fatal error: no input files
compilation terminated.
```

### Compiler Version
I tried the current 4.9 release and it doesn't work with some of the examples here, it
generates incorrect code, so stick to the 4.7 series until that's sorted out.
[This](https://launchpad.net/gcc-arm-embedded/4.7/4.7-2013-q3-update) is the compiler I'm
using throughout these tutorials. Having the same helps because you'll be able to step
through the disassembly listings as the tutorial progresses.

The [eLinux page](http://elinux.org/RPi_Software#ARM) gives us the optimal GCC settings
for compiling code for the original Raspberry-Pi:

```
-Ofast -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s
```

It is noted that -Ofast may cause problems with some compilations, so it is probably
better that we stick with the more traditional -O2 optimisation setting. The other flags merely
tell GCC what type of floating point unit we have, tell it to produce hard-floating point code
(GCC can create software floating point support instead), and tells GCC what ARM processor
architecture we have so that it can produce optimal and compatible assembly/machine code.

For the Raspberry-Pi 2 we know that the architecture is different. The ARM1176 from the original
pi has been replaced by a quad core Cortex A7 processor. Therefore, in order to compile
effectively for the Raspberry-Pi 2 we use a different set of compiler options:

```
-O2 -mfpu=neon-vfpv4 -mfloat-abi=hard -march=armv7-a -mtune=cortex-a7
```

You can see from the
[ARM specification of the Cortex A7](http://www.arm.com/products/processors/cortex-a/cortex-a7.php)
that it contains a VFPV4 floating point processor and a NEON engine. The settings are gleaned
from the [GCC ARM options](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html) page.

## Getting to know the Compiler and Linker

In order to use a C compiler, we need to understand what the compiler does and what the linker
does in order to generate executable code. The compiler converts C statements into assembler
and performs optimisation of the assembly instructions. This is in-fact all the C compiler
does!

The C compiler then implicitly calls the assembler to assemble that file (usually a temporary)
into an object file. This will have relocatable machine code in it along with symbol
information for the linker to use. These days the C compiler pipes the assembly to the
assembler so there is no intermediate file as creating files is a lot slower than passing data
from one program to another through a pipe.

The linker's job is to link everything into an executable file. The linker requires a linker
script. The linker script tells the linker how to organise the various object files. The linker
will resolve symbols to addresses when it has arranged all the objects according to the rules
in the linker script.

What we're getting close to here is that a C program isn't just the code we type. There are
some fundamental things that must happen for C code to run. For example, some variables need to
be initialised to certain values, and some variables need to be initialised to 0. This is all
taken care of by an object file which is usually implicitly linked in by the linker because
the linker script will include a reference to it. The object file is called crt0.o
(C Run-Time zero)

This code uses symbols that the linker can resolve to clear the start of the area where
initialised variables starts and ends in order to zero this memory section. It generally sets up
a stack pointer, and it always includes a call to _main. Here's an important note: symbols present
in C code get prepended with an underscore in the generation of the assembler version of the code.
So where the start of a C program is the main symbol, in assembler we need to refer to it as it's
assembler version which is _main.

### Github

All of the source in the tutorials is available from the
[Github repo](https://github.com/BrianSidebotham/arm-tutorial-rpi).
So go clone or fork now so you have all the code to compile and modify as you work through the
tutorials.

Let's have a look at compiling one of the simplest programs that we can. Lets compile and link
the following program:

## part-1/armc-00

```
int main(void)
{
    while(1)
    {

    }

    return 0;
}
```

There are build "scripts" for each of the different types of Raspberry-pi under the part-1/armc-00
directory of the tutorials source code. There are three types, the original which is targeted with
built.bat/sh, the B+ which has the extended IO connector and fixing holes but it still a V1 RPi
which is targeting using build-rpi-bplus.bat/sh and finally the V2 board which features a quad
core processor and also has the extended IO connector and fixing holes which is targeted with
build-rpi-2.bat/sh.

The V1 boards are fitted with the Broadcom BCM2835 (ARM1176) and the V2 board uses the BCM2836
(ARM Cortex A7). Because of the processor difference, we use different build commands to build for
V1 or V2 boards, hence the different scripts for building:

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s arm-test.c
```

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv7-a -mtune=cortex-a7 arm-test.c
```

GCC does successfully compile the source code (there are no C errors in it), but the linker fails
with the following message:

```
.../arm-none-eabi/lib/fpu\libc.a(lib_a-exit.o): In function `exit':
exit.c:(.text.exit+0x2c): undefined reference to `_exit'
collect2.exe: error: ld returned 1 exit status
```

So with our one-line command above we're invoking the C compiler, the assembler and the linker.
The C compiler does most of the menial tasks for us to make life easier for us, but because we're
embedded engineers (aren't we?) we MUST be aware of how the compiler, assembler and linker work
at a very low level as we generally work with custom systems which we must describe intimately
to the toolchain.

So there's a missing `_exit` symbol. This symbol is reference by the C library we're using. It is
in-fact a system call. It's designed to be implemented by the OS. It would be called when a
program terminates. In our case, we are our own OS at we're the only thing running, and in fact we
will never exit so we do not need to really worry about it. System calls can be blank, they just
merely need to be provided in order for the linker to resolve the symbol.

So the C library has a requirement of system calls. Sometimes these are already implemented as
blank functions, or implemented for fixed functionality. For a list of system calls see the
[newlib documentation on system calls](http://sourceware.org/newlib/libc.html#Stubs).
Newlib is an open source, and lightweight C library.

The C library is what provides all of the C functionality found in standard C header files such
as `stdio.h`, `stlib.h`, `string.h`, etc.

At this point I want to note that the standard Hello World example won't work here without an OS,
and it is exactly unimplemented system calls that prevent it from being our first example. The
lowest part of printf(...) includes a write function "write" - this function is used by all of
the functions in the C library that need to write to a file. In the case of printf, it needs to
write to the file stdout. Generally when an OS is running stdout produces output visible on a
screen which can then be piped to another file system file by the OS. Without an OS, stdout
generally prints to a UART to so that you can see program output on a remote screen such as a PC
running a terminal program. We will discuss write implementations later on in the tutorial
series, let's move on...

The easiest way to fix the link problem is to provide a minimal exit function to satisfy the
linker. As it is never going to be used, all we need to do is shut the linker up and let it
resolve _exit. So now we can compile the next version of the code:

### part-1/armc-01

```
int main(void)
{
    while(1)
    {

    }

    return 0;
}

void exit(int code)
{
    while(1)
        ;
}
```

It's important to have an infinite loop in the exit function. In the C library, which is not
intended to be used with an operating system (hence arm-NONE-eabi-*), _exit is marked as being
noreturn. We must make sure it doesn't return otherwise we will get a warning about it. The
prototype for _exit always includes an exit code int too.

Now using the same build command above we get a clean build! Yay! But there is really a problem,
in order to provide a system underneath the C library we will have to provide linker scripts and
our own C Startup code. In order to skip that initially and to simply get up and running we'll
just use GCC's option not to include any of the C startup routines, which excludes the need for
exit too, `-nostartfiles`

## Getting to Know the Processor

As in the Cambridge tutorials we will copy their initial example of illuminating an LED in order
to know that our code is running correctly.

### Raspberry-Pi Boot Process

First, let's have a look at how a Raspberry-Pi processor boots. The BCM2385 from Broadcom includes
two processors that we should know about, one is a Videocore(tm) GPU which is why the Raspberry-Pi
makes such a good media-centre and the other is the ARM core which runs the operating system. Both
of these processors share the peripheral bus and also have to share some interrupt resources.
Although in this case, share means that some interrupt sources are not available to the ARM
processor because they are already taken by the GPU.

The GPU starts running at reset or power on and includes code to read the first FAT partition of
the SD Card on the MMC bus. It searches for and loads a file called bootcode.bin into memory and
starts execution of that code. The bootcode.bin bootloader in turn searches the SD card for a file
called start.elf and a config.txt file to set various kernel settings before searching the SD card
again for a kernel.img file which it then loads into memory at a specific address (0x8000) and
starts the ARM processor executing at that memory location. The GPU is now up and running and the
ARM will start to come up using the code contained in kernel.img. The start.elf file contains the
code that runs on the GPU to provide most of the requirements of OpenGL, etc.

Therefore in order to boot your own code, you need to firstly compile your code to an executable
and name it kernel.img, and put it onto a FAT formatted SD Card, which has the GPU bootloader
(bootcode.bin, and start.elf) on it as well. The latest Raspberry-Pi firmware is available on
[GitHub](https://github.com/raspberrypi/firmware). The bootloader is located under the [boot
sub-directory](https://github.com/raspberrypi/firmware/tree/master/boot). The rest of the firmware
provided is closed-binary video drivers. They are compiled for use under Linux so that accelerated
graphics drivers are available. As we're not using Linux these files are of no use to us, only the
bootloader firmware is.

All this means that the processor is already up and running when it starts to run our code. Clock
sources and PLL settings are already decided and programmed in the bootloader which alleviates
that problem from us. We get to just start messing with the devices registers from an already
running core. This is something I'm not that used too, normally the first thing in my code would
be setting up correct clock and PLL settings to initialise the processor, but the GPU has setup
the basic clocking scheme for us.

The first thing we will need to set up is the GPIO controller. There are no drivers we can rely on
as there is no OS running, all the bootloader has done is boot the processor into a working state,
ready to start loading the OS.

You'll need to get the
[Raspberry-Pi BCM2835 peripherals datahsheet](http://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf)
which gives us the information we require to control the IO peripherals of the BCM2835. I'll guide
us through using the GPIO peripheral - there are as always some gotcha's:

We'll be using the GPIO peripheral, and it would therefore be natural to jump straight to that
documentation and start writing code, but we need to first read some of the 'basic' information
about the processor. The important bit to note is the virtual address information. On page 5 of
the BCM2835 peripherals page we see an IO map for the processor. Again, as embedded engineers we
must have the IO map to know how to address peripherals on the processor and in some cases how to
arrange our linker scripts when there are multiple address spaces.

![ARM Virtual addresses](http://www.valvers.com/wp-content/uploads/2013/01/arm-c-virtual-addresses.jpg)

The VC CPU Bus addresses relate to the Broadcom Video Core CPU. Although the Video Core CPU is what
bootloads from the SD Card, execution is handed over to the ARM core by the time our kernel.img code
is called. So we're not interested in the VC CPU Bus addresses.

The ARM Physical addresses is the processors raw IO map when the ARM Memory Management Unit (MMU)
is not being used. If the MMU is being used, the virtual address space is what what we'd be
interested in.

Before an OS kernel is running, the MMU is also not running as it has not been initialised and the
core is running in kernel mode. Addresses on the bus are therefore accessed via their ARM Physical
Address. We can see from the IO map that the VC CPU Address 0x7E000000 is mapped to ARM Physical
Address 0x20000000 for the original Raspberry Pi. This is important!

Although not documented anywhere, the Raspberry-Pi 2 has the ARM IO base set to 0x3F000000 instead
of the original 0x20000000 of the original Raspberry-Pi. Unfortunately for us software engineers
the Raspberry-Pi foundation don't appear to be good a securing the documentation we need, in fact,
their [attitude suggests](http://www.raspberrypi.org/forums/viewtopic.php?f=72&amp;t=98400) they
think we're magicians and don't actually need any. What a shame! Please if you're a member of the
forum, campaign for more documentation. As engineers, especially in industry we wouldn't accept
this from a manufacturer, we'd go elsewhere! In fact, we did at my work and use the TI Cortex
A8 from the Beaglebone Black, a very good and well documented SoC!

Anyway, the base address can be gleaned from searching for uboot patches. The Raspberry Pi 2 uses
a BCM2836 so we can search for that and u-boot and we come along a
[patch for supporting the Raspberry-Pi 2](http://lists.denx.de/pipermail/u-boot/2015-February/204483.html).

Further on in the manual we come across the GPIO peripheral section of the manual (Chapter 6,
page 89).

## Visual Output and Running Code

Finally, let's get on and see some of our code running on the Raspberry-Pi. We'll continue with
using the first example of the Cambridge tutorials by lighting the OK LED on the Raspberry-Pi
board. This'll be our equivalent of the ubiquitous Hello World example. Normally an embedded
Hello World is a blinking LED, so that we know the processor is continuously running. We'll move
on to that in a bit.

The GPIO peripheral has a base address in the BCM2835 manual at 0x7E200000. We know from getting
to know our processor that this translates to an ARM Physical Address of 0x20200000 (0x3F200000
for RPI2). This is the first register in the GPIO peripheral register set, the 'GPIO Function
Select 0' register.

In order to use an IO pin, we need to configure the GPIO peripheral. From the
[Raspberry-Pi schematic diagrams](http://www.raspberrypi.org/wp-content/uploads/2012/10/Raspberry-Pi-R2.0-Schematics-Issue2.2_027.pdf)
the OK LED is wired to the GPIO16 line (Sheet 2, B5) . The LED is wired active LOW - this is
fairly standard practice. It means to turn the LED on we need to output a 0 (the pin is connected
to 0V by the processor) and to turn it off we output a 1 (the pin is connected to VDD by the
processor).

Unfortunately, again, lack of documentation is rife and we don't have schematics for the
Raspberry-Pi 2 or plus models! This is important because the GPIO lines were re-jigged and as
Florin has noted in the comments section, the Raspberry Pi Plus configuration has the LED on
GPIO47, so I've added the changes in brackets below for the RPI B+ models (Which includes the
RPI 2).

Back to the processor manual and we see that the first thing we need to do is set the GPIO
pin to an output. This is done by setting the function of GPIO16 (GPIO47 RPI+) to an output.

Bits 18 to 20 in the 'GPIO Function Select 1' register control the GPIO16 pin.

Bits 21 to 23 in the 'GPIO Function Select 4' register control the GPIO47 pin. (RPI B+)

In C, we will generate a pointer to the register and use the pointer to write a value into the
register. We will mark the register as volatile so that the compiler explicitly does what I tell
it to. If we do not mark the register as volatile, the compiler is free to see that we do not
access this register again and so to all intents and purposes the data we write will not be used
by the program and the optimiser is free to throw away the write because it has no effect.

The effect however is definitely required, but is only externally visible (the mode of the GPIO
pin changes). We inform the compiler through the volatile keyword to not take anything for
granted on this variable and to simply do as I say with it:

```
#ifdef RPI2
    #define GPIO_BASE 0x3F200000UL
#else
    #define GPIO_BASE 0x20200000UL
#endif

volatile unsigned int* gpio_fs1 = (unsigned int*)(GPIO_BASE+0x04);
volatile unsigned int* gpio_fs4 = (unsigned int*)(GPIO_BASE+0x10);
```

In order to set GPIO16 as an output then we need to write a value of 1 in the relevant bits of
the function select register. Here we can rely on the fact that this register is set to 0 after
a reset and so all we need to do is set:

```
#if defined( RPIPLUS ) || defined ( RPI2 )
    *gpio_fs4 |= (1<<21);
#else
    *gpio_fs1 |= (1<<18);
#endif
```

This code looks a bit messy, but we will tidy up and optimise later on. For now we just want to
get to the point where we can light an LED and understand why it is lit!

The ARM GPIO peripherals have an interesting way of doing IO. It's actually a bit different to
most other processor IO implementations. There is a SET register and a CLEAR register. Writing
1 to any bits in the SET register will SET the corresponding GPIO pins to 1 (logic high), and
writing 1 to any bits in the CLEAR register will CLEAR the corresponding GPIO pins to 0
(logic low). There are reasons for this implementation over a register where each bit is a pin
and the bit value directly relates to the pins output level, but it's beyond the scope of this
tutorial.

So in order to light the LED we need to output a 0. We need to write a 1 to bit 16 in the CLEAR
register:

```
*gpio_clear |= (1<<16);
```

Putting what we've learnt into the minimal example above gives us a program that compiles and
links into an executable which should provide us with a Raspberry-Pi that lights the OK LED
when it is powered. Here's the complete code we'll compile:

### part-1/armc-02
```
/* The base address of the GPIO peripheral (ARM Physical Address) */
#ifdef RPI2
    #define GPIO_BASE       0x3F200000UL
#else
    #define GPIO_BASE       0x20200000UL
#endif

#if defined( RPIBPLUS ) || defined( RPI2 )
    #define LED_GPFSEL      GPIO_GPFSEL4
    #define LED_GPFBIT      21
    #define LED_GPSET       GPIO_GPSET1
    #define LED_GPCLR       GPIO_GPCLR1
    #define LED_GPIO_BIT    15
#else
    #define LED_GPFSEL      GPIO_GPFSEL1
    #define LED_GPFBIT      18
    #define LED_GPSET       GPIO_GPSET0
    #define LED_GPCLR       GPIO_GPCLR0
    #define LED_GPIO_BIT    16
#endif

#define GPIO_GPFSEL0    0
#define GPIO_GPFSEL1    1
#define GPIO_GPFSEL2    2
#define GPIO_GPFSEL3    3
#define GPIO_GPFSEL4    4
#define GPIO_GPFSEL5    5

#define GPIO_GPSET0     7
#define GPIO_GPSET1     8

#define GPIO_GPCLR0     10
#define GPIO_GPCLR1     11

#define GPIO_GPLEV0     13
#define GPIO_GPLEV1     14

#define GPIO_GPEDS0     16
#define GPIO_GPEDS1     17

#define GPIO_GPREN0     19
#define GPIO_GPREN1     20

#define GPIO_GPFEN0     22
#define GPIO_GPFEN1     23

#define GPIO_GPHEN0     25
#define GPIO_GPHEN1     26

#define GPIO_GPLEN0     28
#define GPIO_GPLEN1     29

#define GPIO_GPAREN0    31
#define GPIO_GPAREN1    32

#define GPIO_GPAFEN0    34
#define GPIO_GPAFEN1    35

#define GPIO_GPPUD      37
#define GPIO_GPPUDCLK0  38
#define GPIO_GPPUDCLK1  39

/** GPIO Register set */
volatile unsigned int* gpio;

/** Simple loop variable */
volatile unsigned int tim;

/** Main function - we'll never return from here */
int main(void)
{
    /* Assign the address of the GPIO peripheral (Using ARM Physical Address) */
    gpio = (unsigned int*)GPIO_BASE;

    /* Write 1 to the GPIO16 init nibble in the Function Select 1 GPIO
       peripheral register to enable GPIO16 as an output */
    gpio[LED_GPFSEL] |= (1 &lt;&lt; LED_GPFBIT);

    /* Never exit as there is no OS to exit to! */
    while(1)
    {
        for(tim = 0; tim &lt; 500000; tim++)
            ;

        /* Set the LED GPIO pin low ( Turn OK LED on for original Pi, and off
           for plus models )*/
        gpio[LED_GPCLR] = (1 &lt;&lt; LED_GPIO_BIT);

        for(tim = 0; tim &lt; 500000; tim++)
            ;

        /* Set the LED GPIO pin high ( Turn OK LED off for original Pi, and on
           for plus models )*/
        gpio[LED_GPSET] = (1 &lt;&lt; LED_GPIO_BIT);
    }
}
```

We now compile with the slightly modified compilation line:

```
arm-none-eabi-gcc -O2 -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s -nostartfiles armc-2.c -o kernel.elf
```

The linker gives us a warning, which we'll sort out later, but importantly the linker has resolved
the problem for us. This is the warning we'll see and ignore:

```
.../arm-none-eabi/bin/ld.exe: warning: cannot find entry symbol _start; defaulting to 00008000
```

As we can see from the compilation, the standard output is ELF format which is essentially an
executable wrapped with information that an OS may need to know. We need a binary ARM executable
that only includes machine code. We can extract this using the objcopy utility:

```
arm-none-eabi-objcopy kernel.elf -O binary kernel.img
```

### A quick note about the ELF format

[ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) is a file format used by some
OS, including Linux which wraps the machine code with meta-data. The meta-data can be useful.
In Linux and in fact most OS these days, running an executable doesn't mean the file gets loaded
into memory and then the processor starts running from the address at which the file was loaded.
There is usually an executable loader which uses formats like ELF to know more about the
executable, for example the Function call interface might be different between different
executables, this means that code can use different calling conventions which use different
registers for different meanings when calling functions within a program. This can determine
whether the executable loader will even allow the program to be loaded into memory or not. The
ELF format meta-data can also include a list of all of the shared objects (SO, or DLL under
Windows) that this executable also needs to have loaded. If any of the required libraries are
not available, again the executable loader will not allow the file to be loaded and run.

This is all intended (and does) to increase system stability and compatibility.

We however, do not have an OS and the bootloader does not have any loader other than a disk read,
directly copying the kernel.img file into memory at 0x8000 which is then where the ARM processor
starts execution of machine code. Therefore we need to strip off the ELF meta-data and simply
leave just the compiled machine code in the kernel.img file ready for execution.

### Back to our example

This gives us the kernel.img binary file which should only contain ARM machine code. It should be
tens of bytes long. You'll notice that kernel.elf on the otherhand is ~34Kb. Rename the kernel.img
on your SD Card to something like old.kernel.img and save your new kernel.img to the SD Card.
Booting from this SD Card should now leave the OK LED on permanently. The normal startup is for
the OK LED to be on, then extinguish. If it remains extinguished something went wrong with
building or linking your program. Otherwise if the LED remains lit, your program has executed
successfully.

A blinking LED is probably more appropriate to make sure that our code is definitely running.
Let's quickly change the code to crudely blink an LED and then we'll look at sorting out the C
library issues we had earlier as the C library is far too useful to not have access to it.

Compile the code in part-1/armc-03. The code listing is identical to part-1/armc-02 but the build
scripts use objcopy to convert the ELF formatted binary to a raw binary ready to deploy on the SD
Card.

...and see the OK LED Blink! :D

As this is the first example where code should run and give you a visible output on your RPi,
I've included the kernel binaries for each Raspberry-Pi board so that you can load the pre-built
binary and see the LED flash before compiling your own kernel to make sure the build process is
working for you. After this tutorial, you'll have to build your own binaries!

Although the code may appear to be written a little odd, please stick with it! There are reasons
why it's written how it is. Now you can experiment a bit from a basic starting point, but beware
- automatic variables won't work, and nor will initialised variables because we have no C Run
Time support yet.

That will be where we start with
[Step 2 of Bare metal programming the Raspberry-Pi](http://www.valvers.com/embedded-linux/raspberry-pi/step02-bare-metal-programming-in-c-pt2)!

**EDIT:** 23/01/14 - Added some more information about the boot process because of a few questions
I had emailed to me. Hopefully now the bootloading process is a bit clearer</em>. Also added "A
quick note about the ELF format" for clarification of why we use `objcopy -O binary`

**Edit** Added part-1 to the Git repo so the code is better organised and updated the tutorial so
it works with all Raspberry-Pi boards such as the B+ and V2
