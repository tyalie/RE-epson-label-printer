# Epson Label Printer protocol

*Note* All numbers in hex, unless specified otherwise with 0dXX

## Physical layer

The protocol is essentially the same independent of the physical layer
(Bluetooth, USB or Ethernet). The SDK generates a complete command sequence
(e.g. a whole image) and then hands that data over to the respective interfaces
which handle the transfer.

## Communication Sequence

This is an example communication between the host and label printer (BTv1) over
the course of opening the app, printing a label and closing the app again.

1. reset request status [v1]
    - afterwards status messages appear (see status message)
2. reset request status [v1]
    - a second time - unsure why, because status already on its way
3. reset printer
4. set request status [v1]
5. ~multiple commands~ - job environment cmd (static)
    1. tape feed print end
    2. unknown 1
    3. setting tape cut options
    4. set print density
    5. unknown 2
6. ~multiple commands~ - page environment cmd
    1. set request status
    2. set page length
    3. set margin
7. send printing raster
8. ~printing~ (waiting for ST=PrintEnd)
9. reset request status [v1]
10. reset request status [v1]

There are two significant blocks of commands here that set-ups the label which are

### Job Environment Commands

These are always
1. 40: tape feed print end
2. 7b: unknown 1
3. 43: set tape cut options
4. 44: set print density
5. 47: unknown 2

### Page Environment Commands

Outside from a shared set of commands, does this mostly depend on the
capability level of the attached printer.

always
- 4c: page length
- 54: set margin

cap=2
- 57: set width
- 4f: a bit unknown, but something with tapewidth
- 74: set tape properties

cap=3
- 48: unknown 3
- 73: print speed

cap=4
- 48: unknown 3
- 73: print speed
- 58: set half cut continuous
- maybe 79: print priority setting
- maybe 6f: set object type

cap=5
- maybe 79: print priority setting


## Status message

> *NOTE* this here is only valid for the Bluetooth protocol 1 which is for
> Bluetooth models with a status length of 8 for all others more research
> should be done.

### Bluetooth Protocol v1

The status message is UTF-8 and must start with a `@` and must be 0d64 bytes
long. It can be split into small packages of 5 bytes separated by `;`. Each
package consists of two chars property name, a `:` character and a 2 char hex
number. The program itself does not check for the existence of `;` but instead
does a find on the property name (e.g. `ST`) and derives the offsets from
there.

```
PrintPage = 0
Status = "ST"  (see mappings)
Error = ErrorRemain = "ER"
TapeWidth = "TW"
MappedTapeWidth = {
    0:0, 1:2, 2:3, 3:4, 4:5, 5:6, 6:7, 7:C, 0B:1, 11:8, 12:9, 21:a, 23:b,
    51:2, 52:3, 53:4, 54:5, 55:6, 56:7, 57:C, 5B:1, default:ffffffff
  }["TW"]  // see tape width mappings below
TapeKind = "TR"
ErrorDetail = "EI"
ErrorDetail2 = "EJ"
TapeOption = "TO"
InkRibbon = "IR"
RibbonRest = "RR"
WR = "WR"
RV = "RV"
CT = "CT"
OP = "OP"

it is unknown what the last four properties do
```

## Commands
There is a set of commands that is used to communicate with the printer (and
back?).

In general commands are 6 bytes long, but can be followed up by a data section
if the command supports such. All commands start with the ASCII escape sequence
(ESC)

There seem to be two structures how a command can be built. This seems to be
decided on the second byte of the sequence. See structure for more 

Structure
```
ESC type data

if type == 2E:
    raster command
if type == 7B/'{'
    data = length commands... checksum 7D/'}'
    - whereby length represents len(commands) + 1
    - 'checksum' is byte long ∑ of all command bytes
```

### [H->C] image raster transfer
Rasterized images are transferred as binary data with each row separately.

Stuff might be split into pages.

> *Note* Current (untested) assumption: Each page is terminated by an FF and as
> such multiple pages can be sent using FF as a separator. For each page this
> data transfer process repeats and at the end the `send print end` command is
> transferred. This can happen very quickly is not dependent on any printer
> response.

The structure is:
```
Header: (8 bytes)
    ESC 2e 0 0 0 1 lower(height) upper(height)
Data: (height / 8) bytes
    bit encoded row of image with B/W
```

Example data (letter h)
```
1b 2e 000000 01 48 00 0001fffffffffe0000
1b 2e 000000 01 48 00 0001fffffffffe0000
1b 2e 000000 01 48 00 0001fffffffffe0000
1b 2e 000000 01 48 00 0001fffffffffe0000
1b 2e 000000 01 48 00 0001fffffffffe0000
1b 2e 000000 01 48 00 000000000780000000
1b 2e 000000 01 48 00 0000000003c0000000
1b 2e 000000 01 48 00 0000000001c0000000
1b 2e 000000 01 48 00 0000000001e0000000
1b 2e 000000 01 48 00 0000000000e0000000
1b 2e 000000 01 48 00 0000000000f0000000
1b 2e 000000 01 48 00 0000000000f0000000
1b 2e 000000 01 48 00 0000000000f0000000
1b 2e 000000 01 48 00 0000000000f0000000
1b 2e 000000 01 48 00 0000000001f0000000
1b 2e 000000 01 48 00 0000000001f0000000
1b 2e 000000 01 48 00 0000000003e0000000
1b 2e 000000 01 48 00 0001ffffffe0000000
1b 2e 000000 01 48 00 0001ffffffe0000000
1b 2e 000000 01 48 00 0001ffffffc0000000
1b 2e 000000 01 48 00 0001ffffff00000000
1b 2e 000000 01 48 00 0001fffffc00000000
```

The system seems to send FF until the print job is done or at the end of page.

### [H->C] battery ???

> *TODO* This seems to be only for Bluetooth devices of type 2. More research
> should be done to unravel this

Structure
```
- getBatteryType()
open database?
    ESC { 4 55 1 56 }
do request?
    ESC { 5 66 DC2 10 88 }
close database?
    ESC { 4 55 0 55 }

- setBatteryType(int i)
open database?
    ESC { 4 55 1 56 }
prepare?
    ESC { 3 77 77 }
set the battery? (0d12 bytes)
    43 46 3B 7 0 3B DC2 10 3B i 3B (i + 98 + 3B)
close database?
    ESC { 4 55 0 55 }
```

### [H->C] 21: reset printer

Structure
```
Header: (6 bytes)
    ESC { 3 21 21 }
```

### [H->C] 2B: tape feed

#### without cutting
Structure
```
Header: (7 bytes)
    ESC { 4 2B 0 2B }
```

#### with cutting
> *NOTE* unsure about the tape feed

Structure
```
Header: (7 bytes)
    ESC { 4 2B 1 2C }
```

### [H->C] 40: tape feed print end
Send at the end of a (non-cancelled) print process.

> *NOTE* unsure about the non-cancelled

Structure
```
Header: (6 bytes)
    ESC { 3 40 40 }

whereby the 40/0d64 above is SignedBytes.MAX_POWER_OF_TWO (unsure why this or
whether there is an error here by the decompiler)
```

### [H->C] 43: Set tape cut options

Our options are:
- TapeCut
    - 0: cut after each label
    - 1: cut after job
    - 2: don't cut
- HalfCut (*TODO*: unsure about the mapping)
    - 0: fast
    - 1: slow

Structure
```
Header: (0d11 bytes)
    ESC { 7 43 data σ }

whereby data:
  if TapeCut = 2 (no cut)
    data = 00 00 00 00
  elif TapeCut = 0 (after each label)
    if HalfCut = 0
      data = 01 01 01 01
    elif HalfCut = 1
      data = 02 02 01 01
  elif TapeCut = 1 (after job)
    if HalfCut = 0
      data = 01 00 01 01
    elif HalfCut = 1
      data = 02 00 01 01

all options and their mapping:
  TapeCut = 0, HalfCut = 0: 01 01 01 01
  TapeCut = 0, HalfCut = 1: 02 02 01 01
  TapeCut = 1, HalfCut = 0: 01 00 01 01
  TapeCut = 1, HalfCut = 1: 02 00 01 01
  TapeCut = 2, HalfCut = 0: 00 00 00 00
  TapeCut = 2, HalfCut = 1: 00 00 00 00

there also is an option for TapeCut ∉ {0,1,2}, but unsure when this is set:
  TapeCut = x, HalfCut = 0: 01 00 00 01
  TapeCut = x, HalfCut = 1: 02 00 00 01
```

### [H->C] 44: set print density

The print density seems to specify the brightness of the overall print and is a
value between -3 and 3 (according to the UI).

```
Header: (7 bytes)
    ESC { 4 44 density' σ }

whereby density':
  if -5 <= density <= 5:
    density' = density + 5
  else
    density' = 0
```

### [H->C] 47: unknown 2

Unknown what it does, is always hardcoded

Structure
```
Header: (6 bytes)
    ESC { 3 47 47 }
```

### [H->C] 48: unknown 3 !cap∈{3,4}

Unknown what it does, is always hardcoded.

Structure
```
Header: (7 bytes)
    ESC { 4 48 XX σ }

the makeCommand accepts a parameter here, 
  but it is only used with:
XX = 5
```

### [H->C] 4c: page length

Set the page length / height of the label to be printed.

Structure
```
Header: (0d11 bytes)
  ESC { 7 4c le(length, 4) σ }
```

### [H->C] 4f: something with tapewidth !cap=2

Capabilities level 2 only feature. Sets something using MappedTapeWidth,
TapeWidth (TW) and TapeOption (TO) from its settings.

Structure
```
Header: (8 bytes)
    ESC { 5 4f data σ }

MappedTapeWidth, TapeOption, TapeWidth

if tapewidth = 0d11:
  if TapeOption & 0x80 = 0
    data = 61 00
  else
    data = 9c 00
elif tapewidth = 0d10
  data = 89 01
else
  if TapeWidth = 0d33
    data = 89 01
  else
    if TapeOption & 0x80 = 0
      data = 61 00
    else
      data = 9c 00
```

### [H->C] 51,49: request status

#### Set request status
Structure
```
Header: (8 bytes)
    ESC { 5 51 5 0 56 }

[v2] Header: (8 bytes)
    ESC { 5 49 5 0 4E }
```

#### Reset request status

Structure
```
Header: (8 bytes)
    ESC { 5 51 0 0 51 }

[v2] Header: (8 bytes)
    ESC { 5 49 0 0 49 }
```

### [H->C] 54: set margin

Set the print margin after the print

> *TODO* it is not fully clear what exactly it influences 
> and what the unit of this property is

Structure
```
Header: (8 bytes)
    ESC { 5 54 le(margin, 2) σ }
```

### [H->C] 57: set width !cap=2

Sets the page width (maybe tape width). This is for capability level 2 only.

> *NOTE* unit of width is unknown

Structure
```
Header: (8 bytes)
    ESC { 5 57 le(width, 2) σ }
```

### [H->C] 58: set half cut continuous !cap=4

Sets the half cut continuous property. Only for capability level 4.

Structure
```
Header: (7 bytes)
    ESC { 4 58 param σ }

whereby param = halfCutContinous ∈ {0,1}
```

### [H->C] 6f: Set object type !cap=4

This sets the object type to be printed. Is only send if the ObjectType
property can be found in the print database. Only for capability level 4.

Possible values:
- None = 0
- Barcode1D = 1
- Barcode2D = 2

Structure
```
Header: (8 bytes)
    ESC { 5 6f param σ }

whereby param is the object type
```

### [H->C] 73: print speed !cap∈{3,4}

Sets print speed. Only for capability level 3 and 4.

> *NOTE* current is 0 for slow or 1/2 for high.

Structure
```
Header: (7 bytes)
    ESC { 4 73 speed σ }

if cap = 3
  speed = 0 or 1
if cap = 4
  speed = 0 or 2
```

### [H->C] 74: set tape properties !cap=2

Sets tape properties TapeWidth (TW), TapeKind (TR) and InkRibbon (IR). Only for
capabilities level 2.

Structure:
```
Header: (0d9 bytes)
    ESC { 6 74 TW TR IR σ }
```

### [H->C] 79: print priority setting !cap∈{4,5}

Is only send if the print priority setting is enabled. Only for capability
level 4 and 5.

Structure:
```
Header: (6 bytes)
    ESC { 3 79 79 }
```

### [H->C] 7b: unknown 1

Unknown what it does, is always hardcoded

Structure
```
Header: (0d11 bytes)
  ESC { 7 7b 0 0 53 54 22 }
```

## Product capabilities

This depends on the product model and derives the following properties
- has halfcut
- has low speed
- capability level
- has resume job
- resolution
- support tape kinds
- status length

Following is an (incomplete list from `LWPrintProductInformation.java`)

|   Model    | cap level |     tape kinds   | res | halfcut? | low speed? | resume job? | len(stat) |
|------------|-----------|------------------|-----|----------|------------|-------------|-----------|
| LW 1000P   |         3 | 2,3,4,5,6,7      | 360 |      yes |        yes |         yes |        20 |
| LW 600P    |         1 | 2,3,4,5,6        | 180 |       no |         no |          no |         8 |
| LW OK1000P |         3 | 1,2,3,4,5,6,7    | 360 |      yes |        yes |         yes |        20 |
| LW OK600P  |         1 | 1,2,3,4,5,6      | 180 |       no |         no |          no |         8 |
| LW PX800   |         3 | 1,2,3,4,5,6,7    | 360 |      yes |        yes |         yes |        20 |
| LW PX400   |         3 | 1,2,3,4,5,6      | 180 |       no |         no |          no |         8 |
| LW C410    |         5 | 1,2,3,4,5        | 180 |       no |         no |          no |         8 |
| LW C610    |         5 | 1,2,3,4,5        | 360 |       no |         no |          no |        78 |
| LW Z710    |         1 | 1,2,3,4,5,6      | 180 |       no |         no |          no |         8 |
| LW MP100   |         1 | 1,2,3,4,5,6      | 180 |       no |         no |          no |         8 |
| LW Z5010   |         4 | 1,2,3,4,5,6,7,12 | 300 |      yes |        yes |         yes |        78 |
| LW Z5000   |         4 | 1,2,3,4,5,6,7,12 | 300 |      yes |        yes |         yes |        78 |

## Mappings
There are a few enums here, so let's go

### Status
The possible values for status are:
```
00 = Idle
01 = Feeding
02 = Printing
03 = DataSending
04 = FeedEnd
05 = PrintEnd
06 = PickAndPrintPrinting
10 = DemoPrinting
11 = DeviceFeeding
12 = DevicePrinting
13 = FirmwareUpdating
20 = SmallRollWaiting
22 = WaitingForTapeRemoval
48 = Engraving
49 = EngravingEnd
4A = EngravingFeed
4B = EngravingFeedEnd
FF = UnexpectedError
```

### Tape Kind
The possible tape kinds are

```
00 = Normal
01 = Transfer
10 = Cable
11 = Index
40 = Braille
50 = Olefin
51 = ThermalPaper
52 = Tube
53 = PET
54 = DieCut1
55 = DieCut2
56 = DieCut3
57 = DieCut4
58 = DieCut5
59 = WideReserved1
5a = WideReserved2
5b = WideReserved3
5c = WideReserved4
5d = WideReserved5
5e = WideReserved6
5f = WideReserved7
56 = DieCutCircle
61 = DieCutEllipse
62 = DieCutRoundedCorners
63 = DieCutReserved1
64 = DieCutReserved2
65 = DieCutReserved3
66 = DieCutReserved4
67 = DieCutReserved5
68 = DieCutReserved6
69 = DieCutReserved7
6a = DieCutReserved8
6b = DieCutReserved9
6c = DieCutReserved10
6d = DieCutReserved11
6e = DieCutReserved12
6f = DieCutReserved13
70 = HST
80 = Vinyl
72 = Cleaning
ff = Unknown
``` 

### Tape Width

```
00 = None
01 = 4 mm
02 = 6 mm
03 = 9 mm
04 = 12 mm
05 = 18 mm
06 = 24 mm
07 = 36 mm
08 = 24 mm Cable
09 = 36 mm Cable
0A = 50 mm 
0B = 100 mm
0C = new 50 mm
FF = unknown
```

## Glossary

- FF: ASCII Form Feed (0x0C | 0d12)
- ESC: ASCII Escape sequence (0x1B | 0d27)
- DC2: ASCII Device Control 2 (0x12 | 0d18)
- lower, upper: means lower and upper byte of 16bit integer
- {,}: the ascii characters { and } standing for 0x7B and 0x7D respectively
- σ: cumulative sum of command part as checksum. See overall structure
- `le(data, len)`: `data` as little-endian of `len` bytes
