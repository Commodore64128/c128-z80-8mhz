
# C128 and 8MHz Z80

With a simple daughterboard, a GAL and a 74LS74 the Z80 in C128 can run at dot clock frequency (almost 8MHz) during its 1MHz phase of the system clock, effectively doubling the speed.

Basic benchmarks written in Turbo Pascal show that indeed it is about twice faster.

CP/M is visibly snappier, to the point of being usable.

<img src="media/01.rev2-installed.jpg" alt="Tower of development daughterboards" height=800>

This circuit works fine, during tests it was stable for several hours, busy with drawing fractals and calculating pi using Monte-Carlo method.

The CPU doesn't even get warm to the touch.

Disk I/O worked fine, as well as REU ram disk M: (emulated by UII+).

## BACKGROUND

There was a message sent to [cbm-hackers mailing list](https://www.softwolves.com/arkiv/cbm-hackers/7/7361.html) back in 2002, about a PCB for C128 that boosted Z80 to 8MHz

Unfortunately that device was not fully reverse-engineered and it seems that it was never mentioned again.

## PREPARING C128

I haven't tried to overclock the original Z80B, I replaced it some time ago by a Z84C0020 that uses much less energy.

Following modifications can be done using bending Z80 pins and soldering to the mainboard, but having a daughterboard will keep everything tidy.

We need to isolate `/WAIT` input (pin 24) from the mainboard because it is tied to `/NMI` there.
We will also disconnected from CPU the onboard `CLK` signal (pin 6).
We will use it as an GAL input to have an option of switching between the original clock (4MHz half of the time) and fast mode (8MHz half of the time).
The new `CLK` will be generated by GAL. We will also use few other lines from Z80 as GAL inputs.

We need two more inputs not available on the Z80 chip/Z80 socket. We want `DOT CLOCK` from the expansion port (pin 6) and the system clock 1MHz signal
(called `CLK1MHZ` below and `D1MHZ` on C128 schematic) from U12 (pin 11).

## OPERATION

This is the normal timing of Z80 in C128. During system clock `CLK1MHZ` low phase VIC generates two clock ticks for Z80.

<img src="media/02.standardclock.jpg" alt="Oscilloscope view of D1MHZ and Z80 CLK" height=800>

We use `DOT CLOCK` instead to pass four ticks.

Whenever Z80 wants to read or write to/from memory or I/O we have to stop it until the start of the next 1MHz clock low phase.

Latched data from 74LS74 will hold the `/WAIT` line low until the end of current low phase of `CLK1MHZ` (Z80 turn).
During the following high phase (VIC's turn) 74LS74 will stay in reset and release `WAIT` to inactive (high) state, but during that time we will not output any clock pulses - just like in the original setup.

I had to add a case to `CLKOUT` to handle memory writes. Waitstate is not enough, the CPU had to be really stopped. That's not an issue for reads because they are buffered with U12.

Here is the effect - for two clocks Z80 was running at 1MHz because of memory write, then picked up the speed:

<img src="media/04.fastclock-with-waitstates.jpg" alt="Oscilloscope view of D1MHZ and Z80 CLK" height=800>

## CIRCUIT

The Kicad project is in [z80-dotclock-gal-and-latch/](z80-dotclock-gal-and-latch) folder.

Schematic is also [available in PDF form](z80-dotclock-gal-and-latch/plots/z80-dot-gal.pdf).

## PCB

Gerber files generated from PCB project that should fit C128 and C128DCR are in  [z80-dotclock-gal-and-latch/gerbers/](z80-dotclock-gal-and-latch/gerbers) folder.

## LOGIC EQUATIONS

PLD file to be used with WinCUPL to generate JED file for programming GAL is available in two versions:

- for 22V10 in [z80-dotclock-gal-and-latch/gal22v10/](z80-dotclock-gal-and-latch/gal22v10/)
- for 16V8 in [z80-dotclock-gal-and-latch/gal6v8/](z80-dotclock-gal-and-latch/gal16v8/)

You will find there all the files generated by WinCUPL, including JED files ready to be flased to GAL devices.

They differ only in pin numbering. I just happened to have an ATF22V10 available.
The circuit is ready for both options - just align pin 1 of 16V8 with pin 1 of the socket.

### 74LS74 and bus timing

Here is how it is supposed to work:

0. VIC cycle phase starts, latch is in reset state, latch `D` input is
   constant `1`, but due to reset `Q` output is `0` and negated `/Q` is `1`; Z80 is
   stopped as its clock is held low
1. Z80 cycle phase starts, latch goes out of reset, `/Q` output = `WAITLATCH`
   stays `1`, Z80 sees that as inactive `/WAIT` and runs program, without bus
   access requests `WAITTRIGGER` stays `0`
2. Z80 wants to access the bus, `WAITTRIGGER` will rise to `1`; this will cause
   latch D input connected to `1` be copied to the output, so `/Q` becomes `0`, Z80
   will see that as an active `/WAIT` and will stop
3. This state will not change until start of the next VIC cycle phase
4. VIC cycle phase starts, latch is reset - will set latch `/Q` to `1`, so for
   Z80 `/WAIT` now becomes inactive; but Z80 won't run because during this
   half-cycle the CPU clock signal is not passed at all
5. Z80 cycle phase starts, it can continue because after latch reset `/WAIT`
   is inactive; Z80 accesses the bus so `WAITTRIGGER` stays `1` until Z80 gets
   what it needs from data lines - then it goes back to `0`

The idea of this is that during Z80 clock phase when `WAITTRIGGER` goes from
inactive to active we can latch `/WAIT` line low to hold it low at least
until the start of VIC cycle.

We release it before/at the start of Z80 cycle it so `/WAIT` becomes high
again - even though the latch trigger condition is still active.


### CLKOUT
The simplest way to handle CLKOUT I did was:
```
  CLKOUT = !DOTCLK & !CLK1MHZ;
```
C128 started fine, but CP/M wouldn't boot. With pieces of Z80 code I was able to confirm that this works for I/O reads and writes but only memory reads.
Memory writes never happened, hence the special case with `MREQ & WR`. 

## 16MHZ?

I tried using the 16MHz clock signal from VDC (pin 2) instead of dot clock.

The C128 will start, but CP/M won't boot. However all my simple tests passed.
I suspect that memory access is still an issue and some of the memory writes fail.
16MHz clock seems possible.

## OPEN QUESTIONS

1) How can we keep it simple but run at full speed all the time and stall only on memory and I/O access? It's not clear if the device mentioned in 2002 was really working at 8MHz all the time or just during Z80's turn.

2) Since I already used ATF22V10 which has asynchronous reset for registers, it should be possible to get rid of 74LS74 with different assignment of pin 1 (GAL CLK input). I wasn't able to do it.
   I'm probably doing something wrong - messing up high/low vs active/inactive or my CUPL skills are lacking (they are).

3) What is the problem with 16MHz clock? Maybe it should be created by doubling the dot clock to keep things in sync? (No need to bother if (1) is possible).
