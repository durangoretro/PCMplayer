# PCMplayer
8-bit mono PCM playback for _Durango_ home computer with optional _Tri-PSG_ sound card.

## Hardware requirements
- **Durango-X** home computer (_-S_ will do, or any variant if you don't care about the screen).
- [Tri-PSG sound card **with PCM option**](https://github.com/zuiko21/minimOS/tree/master/hard/kicad/durango/cartridges/tri-psg), either mono or stereo (PCM playback is _mono_ only).
- A [**bankswitching** ROM cartridge](https://github.com/zuiko21/minimOS/tree/master/hard/kicad/durango/cartridges/cart32bbs) (the [_universal_ cartridge](https://github.com/zuiko21/minimOS/tree/master/hard/kicad/durango/cartridges/cart2832bbspsg) is suitable as well) **configured for _16 KiB_ banks**.

## Specs
- **8-bit linear PCM mono** playback.
- _Sampling rate_ is defined by software. Up to **37 kHz** is feasible on a Durango _v1_, while a **3.5 MHz Durango _v2_** will happily do **85 kHz**. Much lower rates are of course available; _12 kHz_ seems to sound pretty acceptably and gives much longer playback time.
- _Playback time_ depends on both ROM size and sampling rate. For instance, a 128 kiB ROM recorded at 12 kHz will hold about **10.75 seconds** of audio.

|Sampling Rate (Hz)|Playback time (128 kiB)|
|------------------|-----------------------|
|8000              |16.128 s               |
|12000             |10.752 s               |
|16000             |8.064 s                |
|22050             |~5.85 s                |
|31250             |~4.129 s               |
|44100             |~2.926 s               |
|48000             |2.688 s                |

## ROM format
Each 16 kiB bank stores the [player code](pcm-player.s) at the last page (addresses `$FF00-$FFFF`, including mandatory _6502 hard vectors_), preceded by **15.75 kiB** (16128 bytes) of raw audio data.

The use of _16 kiB_ banks is needed because audio data can be accessed **uninterrupted** by the I/O page thru the _mirror_ image at `$8000-$BEFF`; while code is also accesible at `$BF00-$BFFF`, it's always executed at the very last page of the bank, where the _hard vectors_ are expected to be.

_**Note:** player code might be adapted for a single, non-bankswitching ROM or even sample data in RAM; but the I/O page (actually `$DF80-$DFFF`) **must** be avoided, otherwise a glitch will be heard and **unexpected I/O activity** may happen._

Since longer samples cannot be loaded or executed other than from an _actual_ ROM cartridge, no support for ROM _images_ or the _Perdita_ emulator is granted, thus no standard header is provided. For shorter playback times, however, code could be adapted to skip a suitable header.

## Computer speed

Without any kind of DMA available, this is of course _hardwired **cycle-count** code_ and playback rate will differ according to CPU clock speed. Current version of the code provides **automatic CPU speed detection**, deciding whether it's running on a _fast_ Durango (v2 at **3.5 MHz** TURBO mode) or a _slow_ one (original v1 at **1.536 MHz**, non-TURBO v2 at **1.75 MHz** or _overclocked_ v1 at **2 MHz**).

Sample rate is adjusted via a delay loop inside the player code; base routine takes a _constant_ **41 clock cycles** per sample, thus a suitable delay must be added. `A` register is loaded with a precomputed value (one out of two possible values) and the delay loop will take **5A+2 clock cycles** -- add 41 to that and determine `A` for the desired playback rate (and clock speed). Expected delay values for popular sampling rates are as follows:

|Sampling rate (Hz)|v1, 1.536 MHz|v2, 1.75 MHz|v1 _o.c._ 2 MHz|v2 _TURBO_ 3.5 MHz|
|------------------|-------------|------------|---------------|------------------|
|8000              |151          |178         |209            |397               |
|12000             |87           |105         |126            |251               |
|16000             |55           |68          |84             |178               |
|22050             |29           |38          |50             |118               |
|31250             |8            |15          |23             |71                |
|44100             |n/a          |n/a         |4              |38                |
|48000             |n/a          |n/a         |~0             |32                |

In order to compute the appropriate _delay values_ for `A` of the delay loop, **subtract 2** from the values on the table above, then **divide by 5**, rounding as needed.

### Adapting source code

The current [player code](pcm-player.s) puts any Durango into two separate categories: _fast_ and _slow_. The actual threshold is around **MHz** and is defined on line 78:

```
	CPY #10					; TURBO threshold
```

With value `10` meaning **~3.2 MHz**. It's the count of _blocks of 1287 clock cycles_ needed between 4 ms interrupts.

This comparison chooses a _delay value_ between two, defined in lines 12-13 as:

```
#define	FAST_D	49
#define	SLOW_D	16
```

The `FAST_D` value will be used on **v2 (3.5 MHz)** computers, while you may choose one from the three remaining columns for `SLOW_D`.

You might think that the _non-TURBO v2_ (1.75 MHz) is the natural choice for `SLOW_D`, because the other "slow" machines stay within **~14%** of the nominal speed; however, since the most common configuration (besides _TURBO v2_) will be an _unmodified v1 at **1.536 MHz**_, you may want to take these values as standard.

The aforementioned example (as supplied in the [source code](pcm_player-s)) is designed around a **12 kHz sample rate** (which is a reasonable balance between quality and length) and both the **1.536 and 3.5 MHz** speeds.

#### Fine-tuning sampling rate (optional)

When computing the actual _delay values_ from the previous table, some times **rounding** is needed, thus the sampling rate won't be accurate; however, _these errors will usually stay below 1%_, which is **negligible**.

However, if the utmost precision is needed (?) you may add _delay instructions_ around the delay loop. Since the delay loop works on _5-cycle chunks_, most suitable instructions are:
```
	NOP			; adds 2 clock cycles
	LDA ptr		; adds 3 clock cycles
```
or any combo of these. The sample code on line 162 uses two `NOP`s, accounting for **4 clock cycles**.

_Note that this optimisation can only be done for **one** of the two speed categories_ but, again, it won't be noticeable in most cases.

## Progam usage

The [supplied demo](pcm-bank.s) simply clears the screen with a red 'N' on the top left and something resembling a green 'S' (!) on the bottom right.
Press `SPACE` to turn on (or resume) playback and `NMI` to pause it. Once all ROM banks are finished, playback resumes from start.
