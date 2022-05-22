# CH(unky) (c)OMP(iler)
CHOMP is a command-line tool used for compiling Chunky source files (`.CHT`, `.CHH`, `.I`) into Chunky binary files, which 3DMM uses to read data from, such as MIDIs, bitmaps, models, etc. It can also be used to decompile binary Chunky files.

In the 3DMM source code, every `.CHT` file goes through MSVCs preprocessor, whose output gets saved into a `.I` file. CHOMP can't directly compile the `.CHT` files, as the compiler does not contain a built-in preprocessor.

The file format for binary Chunky files is described in [here](https://github.com/microsoft/Microsoft-3D-Movie-Maker/blob/main/kauai/SRC/CHUNK.CPP). The official documentation for CHOMP can be found [here](https://github.com/microsoft/Microsoft-3D-Movie-Maker/blob/main/kauai/DOC/CHOMP.DOC).

## Implementations
* official `chomp`: https://github.com/microsoft/Microsoft-3D-Movie-Maker/blob/main/kauai/TOOLS/CHOMP.CPP
* bjrkk's `chaw`: https://github.com/bjrkk/chaw

## Usage
```
chomp [/c] <srcTextFile>  <dstChunkFile>  - compile chunky file
chomp  /d  <srcChunkFile> [<dstTextFile>] - decompile chunky file
```
By default, CHOMP is set to compile a chunky file, therefore `/c` is declared as being optional.

## Language lexemes
### Keywords
```
ITEM       FREE
VAR        SET         BYTE       SHORT      LONG
ADOPT      CHILD       PARENT     LONER
STN        STZ         SZ         ST
BO         MACBO       WINBO
OSK        MACOSK      WINOSK
CHUNK      ENDCHUNK 

ALIGN
FILE       SUBFILE     PACKEDFILE 
PACK       PREPACKED   PACKFMT
META       MASK        BITMAP     MIDI       CURSOR    PALETTE
GL         AL          GG         AG
SCRIPT     SCRIPTPF
GST        AST
```
### Punctuators
```
[ ] ( ) { } ,
```
Note that despite `,` being a valid punctuator, it is always ignored.
### Operators
```
 =   -
++  --  +=  -=  *=  /=  %=
~=  |=  ^=  &= >>= <<=
```
### Identifiers
Identifiers must adhere to these rules:
* The first character of an identifier shall not be a numerical digit, else it'll be declared as a numeric literal.
* The first character of an identifier shall be either a letter or an underscore.
* Identifiers shall only be composed of letters, underscores and digits.
* Identifiers shall be case-sensitive.

### Literals
Literals can either be textual or numerical.

Textual literals (strings) are formed by putting text in quotation marks. (e.g.: `"Hello world!"`)

Numerical literals can only be represented through integers (e.g.: `100`, `250`) and so-called "tags", which are formed by putting at-most 4 valid ASCII characters surrounded by 2 apostrophes at the beginning and end, which is then fit into a 32-bit unsigned integer. (e.g.: `'MIDS'`, `'MBMP'`, `'WAVE'`)

## Language overview
Note that this does not go in detail over the scripting language used for the Kidspace system.

### Variables
```
SET <identifier|name> <operator|op> [<identifier,literal|value>]
```
where `op` has to specifically be either an assignment operator or a unary operator. `value` is required if `op` is an assignment operator.

Variables can *only* exist in a global scope.

### Internal variables
CHOMP internally keeps track of 2 16-bit unsigned integers that are used to represent the byte-order and operating-system kind. 

As specified in `src/KAUAI/UTILSTR.H`, the byte-order variable shall be:
* `0x0100` in a little-endian environment
* `0x0200` in a big-endian environment

In the same vein, the operating system kind shall be:
* `0x0202` on 68k-compatible MacOS with ANSI enabled, 
* `0x0303` on Windows with ANSI enabled
* `0x0404` on 68k-compatible MacOS with Unicode enabled.
* `0x0505` on Windows with Unicode enabled

These can also manually be set through the `MACBO`, `WINBO`, `MACOSK`, `WINOSK` instructions, which can be called anywhere.

Any instance of the `BO` keyword may be replaced with the value present in the byte-order variable, and consequently, any instance of the `OSK` keyword may be replaced with the value present in the operating-system kind variable.

Note that despite CHOMP allowing you to change the values of `BO` and `OSK` through `SET`, it does not alter their output. This is because `BO` and `OSK` get treated as identifiers when using `SET` on them; however, outside of that, they get treated as keywords, and thus it is impossible to reference them directly as global-scoped variables.

### Chunk declaration
```
CHUNK(<numeric|tag>, <numeric|number>, [<textual|name>])
	[...]
ENDCHUNK
```
where `tag` acts as an identifier/hash for the chunk, and `number` as the number of the chunk. `name` is optional, but is used to describe the name of a chunk. The chunk declaration ends once the `ENDCHUNK` keyword is declared.

Inside of the chunk declaration, shall contain the data of the chunk. You may not declare a chunk inside of a chunk.

### Importing raw data
In the data section, you are usually free to input your own hex values. 
The byte-size of the hex values you input are determined by the usage of the `BYTE` (1-byte), `SHORT` (2-bytes), and `LONG` (4-bytes) commands.
You can also input strings, which in turn will be imported as a BSTR, where the first byte determines the length of the string, and ends with a 0x00 null-terminator byte a la C strings.

You may import raw data from a file using the `FILE` command:
```
FILE <textual|filepath>
```
where `filepath` is the path to the raw data.

CHOMP, strangely enough, gives you the ability to import uncompressed files through the `PACKEDFILE` command:
```
PACKEDFILE <textual|filepath>
```
where `filepath` is the path to uncompressed data. The file *must* start with the 4 ASCII character `kaup` or `puak`.

### Importing recognized data types
While in a data section, CHOMP gives you the ability to import some data types that it recognizes.

#### Metafiles
```
META <textual|filepath>
```
where `filepath` is the path to an [Enhanced Metafile](https://en.wikipedia.org/wiki/Windows_Metafile#EMF) file. Despite the code for CHOMP suggesting that it may work with [Windows Metafile](https://en.wikipedia.org/wiki/Windows_Metafile) files, it does not.

#### Bitmaps
```
BITMAP(<numeric|is_transparent>, <numeric|x_pos>, <numeric|y_pos>) <textual|filepath>
  MASK(<numeric|is_transparent>, <numeric|x_pos>, <numeric|y_pos>) <textual|filepath>
```
where `filepath` is the path to a color-indexed BMP image file. If you're importing a `MASK` image, then the BMP image file should only contain 2 colors: the first representing full transparency, and the second one being fully black. Trying to pass a 1-bit monochrome BMP image will not work.

#### Palettes
```
PALETTE <textual|filepath>
```
where `filepath` is the path to a color-indexed BMP image file *with* 256 colors. This directly imports the palette from the image file given.

#### MIDIs
```
MIDI <textual|filepath>
```
where `filepath` is the path to a MIDI file.

#### Cursors
```
CURSOR <textual|filepath>
```
where `filepath` is the path to a Windows CUR file.

### Compressing data
CHOMP gives you the ability to compress the data you're feeding through the data section. This is done by declaring `PACK` before the data you want to compress.

CHOMP also gives you the ability to import compressed data by doing this:
```
PACKEDFILE <textual|filepath>
```
where `filepath` is the path to compressed data. The file *must* start with the 4 ASCII character `kapa` or `apak`.

You may specify the compression format by doing this:
```
PACKFMT(<numeric|compression_format>)
```
TODO: specify the compression formats that can be used

## 3DMMs Chunky Macros
This part of the documentation will go over the macros found in the header files used by the Chunky source files in 3DMMs source code.

### `INC/KIDGS.CHH`

#### `__NAME`
```
__NAME(<textual|name>)
```
This is commonly used in the chunk-related macros. If `NAMES` is defined, then it will push `name` as given; else, it will do nothing. `NAMES` is not defined at all in 3DMMs Chunky source files, presumably as an act of reducing the filesize of the Chunky binary files.

#### `__PACK`
```
__PACK
```
This macro doesn't take any arguments. However, if `PACKALL` is defined, then it will push the `PACK` keyword. In comparsion to `__NAME`, `PACKALL` does actually get defined, which again was presumably as an act of reducing the filesize of the Chunky binary files.

#### `SCRIPTCHUNK`
```
SCRIPTCHUNK(<textual|name>,  <numeric|number>)
```
where `name` acts as the name of the chunk, and `number` as the number of the chunk. This begins the declaration of a chunk with the tag `'GLOP'`.

This gets expanded to:
```
CHUNK('GLOP', <number>, __NAME(<name>))
	SCRIPT
```

#### `MBMPCHUNK`
```
MBMPCHUNK(<textual|filepath>,  <numeric|number>, <numeric|is_transparent>, <numeric|x_pos>, <numeric|y_pos>)
```
where `filepath` acts as the path to the image file, and `number` as the number of the chunk. This begins the declaration of a chunk with the tag `'MBMP'` and with the rest of the information being given by the arguments of the macro.

This gets expanded to:
```
CHUNK('MBMP', <number>, __NAME(<filepath>))
	__PACK
	BITMAP(<is_transparent>, <x_pos>, <y_pos>) <filepath>
ENDCHUNK
```

#### `PALETTECHUNK`
```
PALETTECHUNK(<textual|name>,  <numeric|number>, <textual|filepath>)
```
where `name` acts as the name of the chunk, `number` as the number of the chunk, and `filepath` as the path to the image file. This begins the declaration of a chunk with the tag `'GLCR'`.

This gets expanded to:
```
CHUNK('GLCR', <number>, __NAME(<name>))
	__PACK
	PALETTE <filepath>
ENDCHUNK
```

#### `TILECHUNK`
```
TILECHUNK(<textual|name>, <numeric|number>)
```
where `name` acts as the name of the chunk, and `number` as the number of the chunk. This begins the declaration of a chunk with the tag `'TILE'`.

This gets expanded to:
```
CHUNK('TILE', <number>, __NAME(<name>))
	__PACK
	SHORT BO OSK
```