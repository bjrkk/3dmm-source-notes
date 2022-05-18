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
~=  |=  ^=  &=  >>= <<=
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

Numerical literals can only be represented through integers (e.g.: `100`, `250`) and so-called "tags", which are formed by putting at-most 4 valid ASCII characters surrounded by 2 apostrophes at the beginning and end, which is then fit into a 32-bit unsigned integer. (e.g.: `'MIDS'`, `'MBMP'`, `WAVE`)

## Language overview
Note that this does not go in detail over the scripting language used for the Kidspace system.

### Variables
```
set <identifier|name> <operator|op> [<identifier,literal|value>]
```
where `op` has to specifically be either an assignment operator or a unary operator. `value` is required if `op` is an assignment operator.

Variables can *only* exist in a global scope.

### Built-in Instructions
Your implementation shall internally keep track of 2 16-bit unsigned integers that are used to represent the byte-order and operating-system kind. 

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

Note that despite CHOMP allowing you to use `set` on BO and OSK, it does not alter the output of the `BO` and `OSK` keywords, and thus are impossible to reference directly as global-scoped variables.

### Chunk declaration
```
chunk(<numeric|tag>, <numeric|number>, [<textual|name>])
	[...]
endchunk
```
where `tag` acts as an identifier/hash for the chunk, and `number` as the number of the chunk. `name` is optional, but is used to describe the name of a chunk. The chunk declaration ends once the `endchunk` keyword is declared.

Inside of the chunk declaration, shall contain the data of the chunk. You may not declare a chunk inside of a chunk.