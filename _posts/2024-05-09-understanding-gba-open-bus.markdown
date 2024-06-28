---
layout: post
title: "Understanding gba's \"open bus\""
---

I should warn that this post isn't written with intent to educate like any other
post that begins with "understanding". This is a genuine attempt to understand how 
this garbage works. Any and all conclusions in this post are not reliable and 
should not be used anywhere. Treat me like that one crazy homeless dude, who keeps 
rambling nonsense.

## The problem

I want to write an emulator of the gameboy advance(aka gba) console and usually, the first step is to emulate the cpu.

Gba uses arm7tdmi, which is an arm7 variant with extensions that catered to nintendo's 
needs at the time. It implements the armv4 instruction set and bla bla bla... that's
not important right now. What we should care about is how the cpu interacts with 
external hardware, specifically memory.

Memory access happens every instruction to prefetch the next one, besides that,
access can happen when cpu executes ldr, str families of instructions. ldr in 
particular is used like this:
{% highlight asm %}
ldr r1, [r2]
{% endhighlight %}
it means that whatever is inside r2 is an address and we should read from memory at 
that address and write it into r1.
Now, what if the address is invalid? Like there is no actual hardware mapped to that
memory address. What will happen if we read from it?

Cpu itself doesn't handle that, it is a responsibility of the memory managment unit. 
If invalid access happens - mmu signals abort to the cpu, cpu enters abort state,
and proper abort handling will happen.

Yey, no problems. Right?

The thing is gba doesn't have mmu at all, which kinda makes sense. It is expensive 
and slows things down, basically unnecessary bloat. What? Invalid access? Pft, 
don't you know that "I've been programming in c for 20 years and I've nevSegmentation 
fault (core dumped)"TM.

Yeah, invalid accesses happen A LOT. And cpu behaviour is really funky when it happens,
and even though it is funky it is reproducible kinda, because of that many games
unintentionally rely on it. So we have to emulate it too.

Good thing is that there is something that can be called the bible of gba.
[GBATEK][gbatek] - the greatest technical reference for gba, nds, dsi, and 3ds. 
It describes what we need in the section "GBA Unpredictable Things".
```
 Accessing unused memory at 00004000h-01FFFFFFh, and 10000000h-FFFFFFFFh 
(and 02000000h-03FFFFFFh when RAM is disabled via Port 4000800h)
returns the recently pre-fetched opcode. For ARM code this is simply:  
  WORD = [$+8]  
For THUMB code the result consists of two 16bit fragments and depends on
the address area and alignment where the opcode was stored.  
For THUMB code in Main RAM, Palette Memory, VRAM, and Cartridge ROM this is:  
  LSW = [$+4], MSW = [$+4]  
For THUMB code in BIOS or OAM (and in 32K-WRAM on Original-NDS (in GBA mode)):  
  LSW = [$+4], MSW = [$+6]   ;for opcodes at 4-byte aligned locations  
  LSW = [$+2], MSW = [$+4]   ;for opcodes at non-4-byte aligned locations  
For THUMB code in 32K-WRAM on GBA, GBA SP, GBA Micro, NDS-Lite (but not NDS):  
  LSW = [$+4], MSW = OldHI   ;for opcodes at 4-byte aligned locations  
  LSW = OldLO, MSW = [$+4]   ;for opcodes at non-4-byte aligned locations  
Whereas OldLO/OldHI are usually:  
  OldLO=[$+2], OldHI=[$+2]  
Unless the previous opcode's prefetch was overwritten; that can happen if 
the previous opcode was itself an LDR opcode, ie. if it was itself reading data:  
  OldLO=LSW(data), OldHI=MSW(data)  
  Theoretically, this might also change if a DMA transfer occurs.  
Note: Additionally, as usually, the 32bit data value will be rotated if the
data address wasn't 4-byte aligned, and the upper bits of the 32bit value will be
masked in case of LDRB/LDRH reads.  
Note: The opcode prefetch is caused by the prefetch pipeline in the CPU itself,  
not by the external gamepak prefetch, ie. it works for code in ROM and RAM as well. 
```
A common approach that is often recommended is to implement everything else and 
then emulate the behavior of the bus that is described above. I don't like it. 
I understand that this advice comes from experience, but my gut tells me that 
treating the core issue instead of symptoms would result in much simpler and robust
code.

So I will try to understand why invalid memory access behaves like that? 
Why does it differ from memory region to memory region? Why it depends on access size?  
So before we start speculations, we should summarize what we know about cpu's 
interaction with the memory subsystem.

## Bus intro

Our cpu implements classic von Neuman architecture. So all 
interactions with external hardware happen via something that's called a bus.  
Our bad boy in particular boasts 32 bit address and data bus (bidirectional 
and unidirectional variants). In this post we care only about bidirectional data bus, 
which means both input and output can happen with it.

Very often we need to refer to specific bits on the bus, official reference manuals
do this using the next notation: `D[10:7]`. It means that we refer to bits 7 through 
10 (inclusive) on the data bus. We will also refer the same way to address:
`A[31:2]` means address without the bottom 2 bits.

In our case, one of the main challenges in understanding the relationship between cpu 
and memory is the fact that the memory bus and cpu bus can have a different width.
But before we delve into the details we should understand what cpu expects from 
the memory subsystem.

### Expectations
There are a few simple rules to understand how a memory system should
place data on the bus.
The first one can be summarized as follows: each byte of the data should be placed on  
the corresponding bus region depending on its address. Details are specified in the 
table.
```
----------------------------------
| Lower address bits of the byte |
----------------------------------
|  00  |   01  |   10   |   11   | 
----------------------------------
|D[7:0]|D[15:8]|D[23:16]|D[31:24]| <- where on the bus byte has to be
----------------------------------
```
So if you do a halfword access at address 0xaaaaaaaa, said halfword should occupy 
`D[16:31]`. It is the responsibility of the cpu to properly extract data from the bus.

Also worth mentioning that this depends on the endian configuration, but since
gba uses only little endian, we talk assuming little endian.

The second rule is pretty simple as well. During word access bits `A[1:0]` should be 
ignored, during halfword access - bit `A[0]`, and during byte nothing is ignored.
Basically, this rule says that all accesses will be forcefully aligned. So if you try 
to do a word access at address 0xaaaaaaaa, the memory system will perform access at 
0xaaaaaaa8 and put it on the bus.

Now let's look at how those rules apply in practice.

### Word sized memory systems
Actually, the situation is pretty simple. Rules as described above make word access 
and subword access equivalent. To properly illustrate it:
```
       -------------
Memory:|B3|B2|B1|B0|
       -------------
Addr:   ^0x8  ^0xa 

these are instructions required to perform halfword read at 0xa
mov  r0, 0xa 
ldrh r1, [r0]
and they should result in the bus that looks like this 
-------------
|X0|X1|B1|B0|  where X0 and X1 are some bytes that don't matter.
-------------

If we do word access at 0xa, it will be forcefully aligned to access at 0x8. And it 
will result in the bus:
-------------
|B3|B2|B1|B0|
-------------
Which is what we wanted.
```
So wordwide memory systems _usually_ will just supply a whole word on the bus.
It is a nobrainer choice to use it this way. But there exists an alternative solution,
but to understand it, we need first talk about

### Subword memory systems
Arm7tdmi can to connect to memory systems that have only 16 bit wide bus.
It might seem weird considering the requirements on data positioning on the bus.

It is made possible thanks to the 4 Byte Latch signals or `BL[3:0]` for short. Thanks 
to it, we can control where exactly on the bus our data should end up. 
If we drive `BL[3:2]` HIGH, then our halfword will end up on `D[31:16]`.
Thanks to it we can even perform 32 bit access with subword wide memory system. 
It is actually described in arm7tdmi technical reference manual rev. 3 on page 3-24.
> For example, a word access to halfword wide memory must take 
place in two memory cycles:
- in the first cycle, the data for D[15:0] is obtained from the memory and latched 
into the core on the falling edge of MCLK when BL[1:0] are both HIGH.
- in the second cycle, the data for D[31:16] is latched into the core on the falling 
edge of MCLK when BL[3:2] are both HIGH and BL[1:0] are both LOW

It also has an example with byte wide memory subsystem, which in its essense is the same.
Latch each byte where it should belong. What is interesting is one remark:
>  in the first access:  
- BL[3:0] are driven to 0xF
- the correct data is latched from D[7:0]  
- *unknown* data is latched from D[31:8]

#### Unknown data

What is that unknown data?

Well, I have a guess. Whatever I will be spouting next is pure speculation. I have 
no way to test it, let alone confirm.

Okay, I think that unknown data is just requested data repeated enough times to fill 
the bus. 

`BL[3:0]` are... latches? I mean it is in the name. Well, to be specific 
they are enable signals for corresponding d latches. So by driving specific `BL` to 
LOW we disable writing to that specific part of the bus.

To better understand what I mean, here is an example of read of halfword at the address 
0xa. Reminder that it should be placed on the `D[31:16]`.
```
-------
|B1|B0| <- requested data at 0xa 
-------
 |   |
 ----------
 |  |  |  |
 \/ \/ \/ \/ 
-------------
|B1|B0|B1|B0| <- somewhere around connection of 16bit and 32bit buses.
-------------
 |  |  |  |
 x  x  |  |   <- here we drove BL[1:0] to LOW
 \/ \/ \/ \/ 
-------------
|X1|X0|B1|B0| <- resulting bus; X1 and X0 are some previous bytes
-------------
0    ...   31
```
Naturally, if we drove all byte latches to HIGH, we would get our halfword repeated
twice on the bus.

So why did I arrive at this conclusion? Well, that's the only sane way it could be 
implemented. At least my dummy brain couldn't think of anything else.

### Best usage 
Now that we know (pulled an explanation out of our asses) how byte latching works, we 
can assume how memory subsystems utilize it. 

First is, once again, a nobrainer. Word sized memory systems should always drive 
`BL[3:0]` to 0xf. No need for additional logic and works as cpu expects, there should 
be no sane human who would do otherwise _wink wink_.

Second is less obvious. Subword sized memory systems should only utilize byte latches 
if and only if there is a word access (and halfword access 
in the case of byte sized memory systems). Otherwise `BL[3:0]` should be driven to high, it will work 
as intended that way. 

Now that we came up with bunch of assumptions, we should check how they compare
to reality using the bible of gba - gbatek. Oh wait, we didn't talk about 

## Open bus 
yet...

Nice transition!

_Ahem_. When we perform a read from an invalid address, we end up sending some memory
request into the void, but cpu can't know that because there is no mmu to notify us 
of our mess up. Since cpu thinks that everything is aight, it will just read 
whatevs on the bus.  What is on the bus? Well since there was no memory to access, 
data on the bus wouldn't change from previous successful memory access.

Usually, the last successful memory access is a prefetch of the next opcode. 

As we know, arm7tdmi has 2 modes: arm (32 bit instructions) and thumb 
(16 bit instructions). So for arm mode, open bus data can only be the last prefetched 
opcode. Thumbs case is a lot more complicated because its instructions 
don't occupy the full bus, so it fully depends on how the memory subsystem utilizes byte latches.

We can infer what kind of strategy each memory subsystem uses by looking at the 
gbatek and what kind of data is on the open bus.

### Sweet dreams are made of this 
Who am I to disagree?

I should probably get some sleep. 

Well, let's start exploring every memory region one by one.

Oh, btw, gbatek uses some weird terminology. For example, it uses LSW and MSW, which 
probably mean least significant word and most significant word, when they refer to 
upper or lower half of the bus. Just mentally replace word for halfword and it will 
make sense.

#### Main RAM, Palette Memory, VRAM, and Cartridge ROM 
Doesn't really fit a definition of one. Anyway.

Those have 16 bit buses. And gbatek saith:
> LSW = \[$+4\], MSW = \[$+4\]  

So both halfwords on the bus are the same after halfword access? Well well well, ain't 
it a classic case of driving `BL[3:0]` to 0xf.

This one was easy. To the next one!

#### BIOS, and OAM 
Closer, but still far away from "one by one". Anyway.

Those boast 32 bit buses. And survey said:
>   LSW = [$+4], MSW = [$+6]   ;for opcodes at 4-byte aligned locations  
  LSW = [$+2], MSW = [$+4]   ;for opcodes at non-4-byte aligned locations  

Here it is also obvious. It just force aligns and dumps a whole word on the bus. Just 
as God intended. And obviously `BL[3:0]` is driven to 0xf.

So far so good. Our model can be used to describe observable effects. Onto the next one!

#### 32K-WRAM
Yey, finally one. Or is it a bad sign? Indeed it is. Let's just dive into it.

>  LSW = [$+4], MSW = OldHI   ;for opcodes at 4-byte aligned locations  
  LSW = OldLO, MSW = [$+4]   ;for opcodes at non-4-byte aligned locations  
Whereas OldLO/OldHI are usually:  
  OldLO=[$+2], OldHI=[$+2]  
Unless the previous opcode's prefetch was overwritten; that can happen if 
the previous opcode was itself an LDR opcode, ie. if it was itself reading data:  
  OldLO=LSW(data), OldHI=MSW(data)  

It should be obvious that some weird byte latching is going on behind the scenes. And 
the weirdest thing is this memory region has 32 bit bus! Why? Just drive all latches to 
HIGH, everything will work. And it seems that nds does exactly that when it runs in gba 
mode! Those who designed nds probably assumed that no one would be insane enough to 
dabble with byte latches when you have word wide memory.

So yeah, it does seem to latch specific halfword using either `BL[1:0]` or `BL[3:2]`
depending on the alignment of the access. That's for sure. What interests me is 
whether it latches specific bytes, we can't infer that from gbatek unfortunately. If it 
does latch individual bytes then it will complicate a lot composition of the 
`OldLo` and `OldHi`, because they could consist of one byte of the previous instruction 
and one byte from some access to memory, and those bytes could be swapped depending on 
alignment. 

Also, it isn't mentioned but I think store can also affect open bus value in this region.

I just want to ask why? Did someone want to play around with latches, but forgot to 
take this stupid garbage out of the end product?

Oh, don't forget that everything in here is just a speculation, as well as rambling of 
an insane man.

## Encore
I am sleeeeeeepy.

It took me *WEEKS* to come up with this explanation. I was so dumb
most of the time. The thing that confused me greately are unaligned accesses. 
In arm7tdmi datasheet it says that ldr will rotate data before placing it in register
and I got really confused because I thought that it was responsibility of the bus to 
rotate, mask, and signextend the data. Looking back it is such a dumb assumption. It 
shoud have been obvious that those transformations can only happen inside the cpu, 
bus is only responsible for moving data.

Anyway, if someone read it till the end, have a nice day. 

Peace \/ 

[gbatek]: https://problemkaputt.de/gbatek.htm
