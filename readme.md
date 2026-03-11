# files

FANUC Karel file I/O library — validated open/read/write/close with full error reporting.

## Overview

Karel's built-in `OPEN FILE` / `READ` / `WRITE` statements fail silently: a bad open just sets an I/O status integer that most code ignores. `files` wraps every file operation with `IO_STATUS` checks and `karelError` calls that name the file and produce a human-readable fault message on the teach pendant. Every Layer 4+ module that touches disk (`csv`, `xmlib`, `TPE`, `layout`) delegates its file handle work here.

---

## Files

| File | Purpose |
|------|---------|
| `include/files.klh` | All `ROUTINE` declarations. Include this in every caller. |
| `include/files.klt` | 16 error code `%define`s + `MAX_PIPE = 500` constant. |
| `src/files.kl` | All 21 routine implementations. |
| `test/test_files.kl` | Stub test file (no assertions — tested via `csv` and `TPE` suites). |
| `package.json` | rossum manifest: `depends = [errors, Strings, ktransw-macros, systemlib]` |

---

## API Reference

### Includes

```karel
%include files.klt    -- error codes
%include files.klh    -- routine declarations
```

All routines also available under the short alias `file__*`.

---

### File Lifecycle

#### `files__open`
```karel
files__open(filename : STRING; access_typ : STRING; fl : FILE)
```
Opens a file. `access_typ` values:

| Value | Behavior |
|-------|---------|
| `'RO'` | Read-only |
| `'RW'` | Read-write, truncates to zero on open |
| `'AP'` | Append — writes go to end of file |
| `'UD'` | Update — read/write without truncation |

Calls `CLR_IO_STAT` before opening and `files__check_open` after. Aborts on most errors, warns on `FILE_MULTIPLE_ACCESS`.

#### `files__close`
```karel
files__close(filename : STRING; fl : FILE)
```
Closes the file and validates the close succeeded.

#### `files__create`
```karel
files__create(filename : STRING; fl : FILE) : BOOLEAN
```
Creates the file if it does not exist. Always returns `TRUE`. **Does not leave the file open** — call `files__open` after if you need to write to it.

#### `files__clear`
```karel
files__clear(filename : STRING; fl : FILE)
```
Truncates the file to zero bytes. Safe to call whether the file exists or not.

#### `files__load_on_disk`
```karel
files__load_on_disk(filename : STRING; overwrite : BOOLEAN)
```
Loads a `.pc` binary onto the controller via the FANUC `LOAD` built-in. Set `overwrite = TRUE` to replace an existing program.

---

### Read Operations

#### `files__read_line`
```karel
files__read_line(filename : STRING; fl : FILE) : STRING
```
Reads the next newline-delimited line. Returns `''` at EOF or if the line is uninitialized. File must be open before calling.

#### `files__read_line_ref`
```karel
files__read_line_ref(filename : STRING; fl : FILE; out_str : STRING)
```
Same as `read_line` but writes into `out_str` instead of returning. Avoids a function-return copy — preferred in tight loops.

#### `files__read_bytes`
```karel
files__read_bytes(filename : STRING; fl : FILE; buffer_size : INTEGER; out_str : STRING)
```
Reads `buffer_size` raw bytes into `out_str`. Returns `'EOF'` if no bytes remain. Useful for binary or socket-stream files.

#### `files__purge_bytes`
```karel
files__purge_bytes(filename : STRING; fl : FILE; buffer_size : INTEGER)
```
Discards `buffer_size` bytes without storing them. Use to skip fixed-size headers.

#### `files__read_to_sarr`
```karel
files__read_to_sarr(filename : STRING; fl : FILE;
                    out : ARRAY[*] OF STRING; break_str : STRING) : BOOLEAN
```
Opens the file, reads line-by-line into `out[]`. Stops (returns `TRUE`) when `break_str` appears in a line. Returns `FALSE` if the array fills first (and posts a warning). Closes the file before returning — do not pre-open.

#### `files__read_type`
```karel
files__read_type(parse_str : STRING; split : STRING; offset : INTEGER; size : INTEGER;
                 typ : INTEGER; out_struct : t_DATA_TYPE;
                 out_real_arr : ARRAY[*] OF REAL; out_int_arr : ARRAY[*] OF INTEGER;
                 out_str_arr : ARRAY[*] OF STRING)
```
Parses a pre-read delimited string into typed output. `typ` selects output target:

| `typ` | Output | Notes |
|-------|--------|-------|
| `C_INT` | `out_struct.int` | Field at `offset` |
| `C_REAL` | `out_struct.rl` | Field at `offset` |
| `C_STR` | `out_struct.str` | Fields `[offset, offset+size-1]` joined |
| `C_BOOL` | `out_struct.bool` | Field at `offset` |
| `C_VEC` | `out_struct.vec` | 3 fields (x, y, z) |
| `C_VEC2D` | `out_struct.vec` | 2 fields (x, y); z=0 |
| `C_POS` | `out_struct.pos` | 6 fields (x, y, z, w, p, r) |
| `C_CONFIG` | `out_struct.cnfg` | 1 CONFIG field |
| `C_INT_ARR` | `out_int_arr[]` | `size` integer fields |
| `C_REAL_ARR` | `out_real_arr[]` | `size` real fields |
| `C_STR_ARR` | `out_str_arr[]` | `size` string fields |

Requires `systemlib.datatypes.klt` to be included before `files.klh`.

---

### Write Operations

#### `files__write`
```karel
files__write(str : STRING; filename : STRING; fl : FILE)
```
Writes `str` with no trailing newline.

#### `files__write_line`
```karel
files__write_line(str : STRING; filename : STRING; fl : FILE)
```
Writes `str` followed by a carriage return. Use this for CSV rows, LS file lines, and logs.

#### `files__write_from_pipe`
```karel
files__write_from_pipe(pipname : STRING; pip_fl : FILE;
                       filename : STRING; write_fl : FILE;
                       handle_file : BOOLEAN)
```
Drains a `PIPE:` in-memory file to a disk file. Reads all bytes in 127-byte chunks using `BYTES_AHEAD`, writes them to the disk file. If `handle_file = TRUE`, the routine manages opening and closing the disk file; if `FALSE`, the caller manages the disk handle.

#### `files__write_to_display`
```karel
files__write_to_display(filename : STRING; fl : FILE)
```
Dumps the entire file to the teach pendant display (`TPDISPLAY`). Useful for field debugging of generated output files.

---

### Validation Routines

These are called internally by every open/read/write/close routine. You can also call them explicitly if you need to check state mid-operation.

| Routine | When to use |
|---------|------------|
| `files__check_open(filename, fl)` | After any `OPEN FILE` statement |
| `files__check_closed(filename, fl)` | After any `CLOSE FILE` statement |
| `files__check_rw(filename, fl)` | After each `READ` or `WRITE` in a loop |

---

### Utility

#### `files__is_LS`
```karel
files__is_LS(filename : STRING; fl : FILE) : BOOLEAN
```
Opens the file and scans for the FANUC LS program marker `/MN`. Returns `TRUE` if found (file is a teach-pendant program). Opens and closes the file internally — do not pre-open.

---

## Common Patterns

### Read a file line by line

```karel
VAR
  fl   : FILE
  line : STRING[127]

files__open('MD:/mydata.csv', 'RO', fl)

WHILE BYTES_LEFT(fl) > 0 DO
  files__read_line_ref('MD:/mydata.csv', fl, line)
  IF (line = '') OR UNINIT(line) THEN GOTO SKIP ; ENDIF
  -- process line here
  SKIP::
ENDWHILE

files__close('MD:/mydata.csv', fl)
```

---

### Create-then-append (safe log pattern)

Create the file once at startup; append to it at runtime without clobbering previous data.

```karel
VAR fl : FILE

-- startup
files__create('MD:/run_log.txt', fl)          -- no-op if already exists

-- at runtime
files__open('MD:/run_log.txt', 'AP', fl)
files__write_line('cycle complete', 'MD:/run_log.txt', fl)
files__close('MD:/run_log.txt', fl)
```

---

### Write a FANUC LS program file

The pattern used by `lib/TPE` to programmatically generate `.LS` teach-pendant programs:

```karel
VAR
  newfile : FILE
  prog    : STRING[36]

prog = 'MD:/MYPATH.LS'
files__open(prog, 'RW', newfile)
files__write_line('/PROG MYPATH', prog, newfile)
files__write_line('/MN', prog, newfile)
files__write_line(':  J P[1] 100% FINE ;', prog, newfile)
files__write_line(':  L P[2] 500mm/sec FINE ;', prog, newfile)
files__write_line('/POS', prog, newfile)
-- write position data here...
files__write_line('/END', prog, newfile)
files__close(prog, newfile)
```

---

### Read a file into a string array (batch load)

```karel
VAR
  fl      : FILE
  lines   : ARRAY[200] OF STRING[127]
  found   : BOOLEAN

found = files__read_to_sarr('MD:/config.txt', fl, lines, '/END')
-- found=TRUE means '/END' was hit; found=FALSE means array filled first
```

---

### Parse a delimited line into typed values

```karel
VAR
  fl      : FILE
  line    : STRING[127]
  result  : t_DATA_TYPE
  dummy_r : ARRAY[1] OF REAL
  dummy_i : ARRAY[1] OF INTEGER
  dummy_s : ARRAY[1] OF STRING

files__open('MD:/data.csv', 'RO', fl)
files__read_line_ref('MD:/data.csv', fl, line)
-- parse field at offset 2 as REAL (e.g. second column of CSV)
files__read_type(line, ',', 2, 1, C_REAL, result, dummy_r, dummy_i, dummy_s)
-- result.rl holds the value
files__close('MD:/data.csv', fl)
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Reading/writing without opening first | `karelError` 12328 abort | Call `files__open` first; check device prefix (`MD:/`, `RD:/`, etc.) |
| Using `'RW'` when you want to append | File data is overwritten | Use `'AP'` to add to end; use `'UD'` for in-place updates |
| Calling `files__create` and assuming the file stays open | Next write fails with not-opened error | `files__create` closes the file before returning; always follow with `files__open` |
| Calling `files__read_to_sarr` on an already-open file | Double-open warning or incorrect read position | `read_to_sarr` opens the file internally; do not pre-open |
| Calling `files__is_LS` on an already-open file | Same double-open issue | `is_LS` opens and closes the file internally |
| Sharing a `FILE` variable across concurrent tasks | Interleaved reads/writes, corrupt data | Each task must own its own `FILE` variable |
| Line exceeds 127 characters | Silent truncation with `STRING[127]` buffers | Keep source lines under 127 chars or use `read_bytes` with a sized buffer |

---

## Error Codes

| Constant | Value | Meaning |
|----------|-------|---------|
| `FILE_SUCCESS` | 0 | No error |
| `FILE_RAM_FULL` | 2021 | Controller DRAM full |
| `FILE_UNINIT` | 12311 | File handle uninitialized |
| `FILE_MULTIPLE_ACCESS` | 12326 | File open in another program |
| `FILE_NOT_OPENED` | 12328 | File handle not open |
| `FILE_OPEN_FAILED` | 12327 | `OPEN FILE` returned error |
| `FILE_NOT_OPEN_PREDEF` | 12335 | Cannot open predefined (system) file |
| `FILE_READ_FAILED` | 12334 | `READ` failed |
| `FILE_READ_TO_SHORT` | 12332 | Buffer too small for data |
| `FILE_READ_IO_FAILED` | 12347 | Hardware I/O error on read |
| `FILE_READ_TIMEOUT` | 12358 | Read timed out |
| `FILE_ILLEGAL_ASCII` | 12333 | Illegal character in read data |
| `FILE_WRITE_FAILED` | 12330 | `WRITE` failed |
| `FILE_WRITE_IO_FAILED` | 12348 | Hardware I/O error on write |
| `FILE_CLOSE_FAILED` | 12338 | `CLOSE FILE` failed |
| `FILE_NOT_CLOSE_PREDEF` | 12336 | Cannot close predefined file |

---

## Build Flow

`files` is compiled as part of the normal rossum/ninja build:

```shell
cd lib/files
mkdir build && cd build
rossum .. -w -o    # resolves depends (errors, Strings, ktransw-macros, systemlib)
ninja              # compiles files.kl → files.pc
kpush              # deploys files.pc to the robot controller
```

Modules that depend on `files` (`csv`, `xmlib`, etc.) will automatically include it in their dependency graph. See the [Ka-Boost readme](../../readme.md) for full build instructions.
