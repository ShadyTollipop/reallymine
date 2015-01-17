The JMICRON-CP48 538S is one of many different USB-SATA bridge chips used by WD MyBooks with encryption. (According to the database used by Western Digital's firmware upgrade tool's file database, there are several models that use this chip AND the firmware described below, but not the encryption...) There appears to be absolutely no mention of this chip on the Internet, let alone documentation.

I won't release the dump that I got from the same place Western Digital's own firmware updater software gets it from, nor will I release the disassembly — at least not on github. Here's what you need to know about the file, though:

```
filename: Release-VS-1025-20130711.bin
size:     49152 bytes
crc32:    cfd13030
md5:      7f75e5d59cfac57579effe2d9d5388de
sha1:     a14c6fb97cfa76e5bd1d007897bde20be132c4b8
```

Due to the complete lack of documentation, we will have to work entirely from scratch here.

The firmware is powered by what appears to be a regular old Intel 8501 core.

IDA doesn't want to play nice with this code so let's pick out important items.

The function at 0x1BF0 takes the four bytes after it on the caller's instruction stream as parameters...

RAM 0x3206 appears to be the start of the key sector...?

The code at 0x613 indicates that our ROM is not a boot ROM and that it should be mapped to code memory at 0x4000. Therefore, in the following, if I prefix an address with ~, it is a virtual address.

TODO does this have combined ROM/RAM? 0x1BF0 is still the function above...

~0x5287 writes the `WDv1` signatures and the two bytes after the first `WDv1` to 0x3206; this is called at ~0x7335, which is subsequently followed by copying those first 0x60 bytes of the key sector to RAM 0x3800...

At ~0xB432 is an array of structures describing devices, of the following form:
```go
type deviceDescriptorValues struct {
	VendorID         uint16
	ProductID        uint16
	unknown          [3]byte
	unkStringPtrs    [3]uint16       // [0] appears to be vendor; [2] appears to be product
}
```
The last element is at ~0xB584. I do not know where this is accessed, or how the firmware knows which device it is being used with currently. (Hardcoded by the firmware updater? This won't be important for our purposes, but finding where this array is used **is**)

I do not see either a CBW or a CSW signature in here, at least not directly...

The function at ~0x5217 sets `DPTR` to whatever's at RAM 0x409C and then checks `DPTR` and `DPTR+0x5C` to see if they match `WDv1`. It then checks a few other things that I don't know about... (checksum of just the first 0x58 bytes?)

```
KEY SECTOR STRUCTURE
Offset	Size	What		RAM		Code
CHUNK 0 - HEADER (0x60 BYTES)
0		4		WDv1		0x3206	[TODO]
4		2		checksum	0x320A	[TODO]
...
0x20	1		01			0x3226	[TODO]
...
0x24	4		[00 00]FP	0x322A	[TODO]
...
0x30	1		00			0x3236	[TODO]
0x31	1		see next	0x3237	[TODO]
	bit 0 - ??
	bit 1 - set if bit 5 of 0x3F2F is set
	bit 2-7 - ??
0x32	1		FF			0x3238	[TODO]
0x33	1		00			0x3239	[TODO]
...
0x5C	4		WDv1		0x3262	[TODO]
CHUNK 1 - SOMETHING ELSE (0xXXX(0x30?0x90?) BYTES)
0x60	4		WDq1		0x3266	~0x543B
0x64
0x68	0x10	????		0x326E	[TODO]
0x68	2		checksum	0x326E	[TODO] WTF?
...
0x20	1		00			0x3286	[TODO]
	(other bits TODO; set before bit 5)
	bit 5 - set by ~0x5566 if bit 2 of 0x3F2F set
	bit 6-7 - ????
0x21	1		00			0x3287	~0x556D
...
[CHUNK 2?]
0x90	4		WDqe		0x3296	[TODO]
```

The function at ~0x457F is memset. R6:R7 is the destination, R5 is the byte to fill with, and R3 is the number of bytes.

At ~0xC547 are three sets of suspicious 16-byte blocks...

Checks for `WDq1` at 0x3C00 are at ~0x5164, ~0x5194, ~0x5300, and ~0x5390.

Is ~0xBA7E the firmware's main loop? If so 0x3F44 is the current argument. It might not be...
>Upon further investigation it may actually be the key sector writing loop...?

## Known boot ROM routines
I believe these are provided by the boot ROM; if they are actually in RAM and copied on system startup, I do not know (TODO).
- 0x1BBC - compare R0 to R4, R1 to R5, R2 to R6, R3 to R7
- 0x2B50 - memcpy
	- R6:R7 - (TODO source or destination? initial observation suggested source but after seeing random register assignment order I'm not sure)
	- R4:R5 - (TODO opposite of above)
	- R3 - length (bytes)
	- Is 0x2B6C the same, but for copying from code memory?

TODO could 0x329F be related to disk transfers — or worse, to encryption?

TODO could 0x2B46 be memclr?