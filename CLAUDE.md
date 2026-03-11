# CLAUDE.md — lib/files

## Purpose

`files` is Layer 4's foundational I/O abstraction. It wraps FANUC Karel's raw `OPEN FILE` / `CLOSE FILE` / `READ` / `WRITE` built-ins with validated, error-reporting versions. Every module above Layer 3 that touches disk — `csv`, `xmlib`, `socket`, `TPE`, `layout` — delegates its actual file handle operations here. Without `files`, a bad `OPEN` just silently fails; with it, `karelError` fires with the exact FANUC error code and the filename in the message, making field debugging practical.

---

## Repository Layout

```
lib/files/
├── package.json          # rossum manifest: name=files, depends=[errors,Strings,ktransw-macros,systemlib]
├── readme.md             # one-line description only
├── include/
│   ├── files.klh         # all ROUTINE declarations + namespace alias `file__`
│   └── files.klt         # 16 error code defines + MAX_PIPE=500 constant
├── src/
│   └── files.kl          # all 21 routine implementations (~441 lines)
└── test/
    └── test_files.kl     # stub — kunit_done only, no assertions yet
```

---

## Full API Reference

### Includes needed by callers

```karel
%include files.klt   -- error codes + MAX_PIPE
%include files.klh   -- ROUTINE declarations
```

### Alias

Every `files__*` routine is also accessible as `file__*` (short alias registered via `declare_function`).

---

### File Lifecycle

```karel
files__open(filename : STRING; access_typ : STRING; fl : FILE)
```
Opens `filename` with the given access mode. `access_typ` must be one of:
- `'RO'` — read-only
- `'RW'` — read-write from start (truncates on open)
- `'AP'` — append (write to end)
- `'UD'` — update from beginning (read/write, no truncate)

Calls `CLR_IO_STAT(fl)` before opening, then `files__check_open` after. Posts `karelError` on failure.

```karel
files__close(filename : STRING; fl : FILE)
```
Closes `fl`. Calls `files__check_closed` after. Posts `karelError` on failure.

```karel
files__create(filename : STRING; fl : FILE) : BOOLEAN
```
Creates the file if it does not exist. Implementation: tries `'RO'` open; if status ≠ 0, opens `'RW'`, writes empty string, closes. Always returns `TRUE`. Does NOT leave the file open.

```karel
files__clear(filename : STRING; fl : FILE)
```
Truncates the file to zero bytes. Opens `'RW'` then immediately closes. Safe to call whether file exists or not.

```karel
files__load_on_disk(filename : STRING; overwrite : BOOLEAN)
```
Calls the FANUC `LOAD` built-in to load a `.pc` binary onto the controller. `overwrite=TRUE` passes flag 1. Posts `karelError` if `LOAD` returns non-zero status.

---

### Read Operations

```karel
files__read_line(filename : STRING; fl : FILE) : STRING
```
Reads the next newline-terminated line. Returns `''` if the line is uninit (including at EOF). File must already be open; routine validates state first.

```karel
files__read_line_ref(filename : STRING; fl : FILE; out_str : STRING)
```
Same as `read_line` but writes into an `out_str` parameter instead of returning. Preferred when the caller needs to avoid a function-return copy. Sets `out_str = ''` on uninit.

```karel
files__read_bytes(filename : STRING; fl : FILE; buffer_size : INTEGER; out_str : STRING)
```
Reads `buffer_size` raw bytes. Returns `'EOF'` if `BYTES_LEFT(fl) = 0`. Used for binary/socket-style streaming reads. The 127-byte `STRING` limit applies.

```karel
files__purge_bytes(filename : STRING; fl : FILE; buffer_size : INTEGER)
```
Discards `buffer_size` bytes without storing. Used to skip headers or pad bytes.

```karel
files__read_to_sarr(filename : STRING; fl : FILE; out : ARRAY[*] OF STRING; break_str : STRING) : BOOLEAN
```
Opens the file, reads line-by-line into `out[]`. Stops and returns `TRUE` if `break_str` appears in any line. Returns `FALSE` if the array fills before `break_str` is found (and posts `STR_TOO_LONG` warning). Posts `CANT_READ` abort if any line is uninit. Closes the file before returning.

```karel
files__read_type(parse_str : STRING; split : STRING; offset : INTEGER; size : INTEGER;
                 typ : INTEGER; out_struct : t_DATA_TYPE;
                 out_real_arr : ARRAY[*] OF REAL; out_int_arr : ARRAY[*] OF INTEGER;
                 out_str_arr : ARRAY[*] OF STRING)
```
Parses a pre-read delimited string into typed output. `typ` is one of the `C_*` constants from `systemlib.codes.klt`:

| `typ` constant | Output target | Description |
|----------------|---------------|-------------|
| `C_INT`        | `out_struct.int` | Single integer field |
| `C_REAL`       | `out_struct.rl` | Single real field |
| `C_STR`        | `out_struct.str` | String range [offset, offset+size-1] |
| `C_BOOL`       | `out_struct.bool` | Boolean field |
| `C_VEC`        | `out_struct.vec` | 3-element VECTOR (x,y,z) |
| `C_VEC2D`      | `out_struct.vec` | 2-element VECTOR (x,y), z=0 |
| `C_POS`        | `out_struct.pos` | 6-element XYZWPR |
| `C_CONFIG`     | `out_struct.cnfg` | CONFIG string |
| `C_INT_ARR`    | `out_int_arr[]` | Integer array |
| `C_REAL_ARR`   | `out_real_arr[]` | Real array |
| `C_STR_ARR`    | `out_str_arr[]` | String array |

Requires `%ifdef systemlib_datatypes_t` guard — only compiled when `t_DATA_TYPE` is available.

---

### Write Operations

```karel
files__write(str : STRING; filename : STRING; fl : FILE)
```
Writes `str` with no trailing newline. Thin wrapper over `WRITE fl(str)`.

```karel
files__write_line(str : STRING; filename : STRING; fl : FILE)
```
Writes `str` followed by `CR`. The workhorse for line-oriented output (CSV, LS files, logs).

```karel
files__write_from_pipe(pipname : STRING; pip_fl : FILE;
                       filename : STRING; write_fl : FILE;
                       handle_file : BOOLEAN)
```
Drains a pipe file (in-memory buffer) to a disk file. Reads all bytes from `pip_fl` in a `BYTES_AHEAD` loop (127-byte chunks), writes to `write_fl`. If `handle_file=TRUE`, opens/closes the disk file; if `FALSE`, caller owns the disk handle. Always opens and closes the pipe.

```karel
files__write_to_display(filename : STRING; fl : FILE)
```
Dumps entire file to `TPDISPLAY`. Opens in `'RO'`, loops `BYTES_AHEAD` / `READ` / `WRITE TPDISPLAY` in 127-byte chunks, closes. Useful for debug inspection of generated files.

---

### Validation Routines (called internally, also callable by users)

```karel
files__check_open(filename : STRING; fl : FILE)
```
Calls `IO_STATUS(fl)` after an `OPEN`. Posts `karelError` for each error code:
- `FILE_NOT_OPENED` → abort
- `FILE_NOT_OPEN_PREDEF` → abort
- `FILE_OPEN_FAILED` → abort
- `FILE_MULTIPLE_ACCESS` → warning
- `FILE_RAM_FULL` → writes "EOF" to TPDISPLAY only

```karel
files__check_closed(filename : STRING; fl : FILE)
```
Validates post-`CLOSE`. Error codes handled: `FILE_NOT_OPENED` (warn), `FILE_CLOSE_FAILED` (abort), `FILE_NOT_CLOSE_PREDEF` (abort).

```karel
files__check_rw(filename : STRING; fl : FILE)
```
Validates mid-read/write state. Handles 10 error codes: `FILE_RAM_FULL`, `FILE_UNINIT`, `FILE_MULTIPLE_ACCESS`, `FILE_READ_FAILED`, `FILE_READ_IO_FAILED`, `FILE_READ_TO_SHORT`, `FILE_READ_TIMEOUT`, `FILE_ILLEGAL_ASCII`, `FILE_WRITE_FAILED`, `FILE_WRITE_IO_FAILED`.

---

### Utility

```karel
files__is_LS(filename : STRING; fl : FILE) : BOOLEAN
```
Opens the file, scans line-by-line for the string `/MN` in the first 3 characters of each line. Returns `TRUE` if found (it's a FANUC LS teach-pendant program file). Returns `FALSE` if EOF first. Closes the file before returning.

---

## Error Codes (files.klt)

| Constant | Value | Meaning |
|----------|-------|---------|
| `FILE_SUCCESS` | 0 | No error |
| `FILE_RAM_FULL` | 2021 | Controller memory full |
| `FILE_UNINIT` | 12311 | File handle uninitialized |
| `FILE_MULTIPLE_ACCESS` | 12326 | File open in another program |
| `FILE_NOT_OPENED` | 12328 | File handle not open |
| `FILE_OPEN_FAILED` | 12327 | `OPEN FILE` returned error |
| `FILE_NOT_OPEN_PREDEF` | 12335 | Cannot open predefined file |
| `FILE_READ_FAILED` | 12334 | `READ` failed |
| `FILE_READ_TO_SHORT` | 12332 | Read buffer too small |
| `FILE_READ_IO_FAILED` | 12347 | Hardware I/O error on read |
| `FILE_READ_TIMEOUT` | 12358 | Read timed out |
| `FILE_ILLEGAL_ASCII` | 12333 | Illegal character in read data |
| `FILE_WRITE_FAILED` | 12330 | `WRITE` failed |
| `FILE_WRITE_IO_FAILED` | 12348 | Hardware I/O error on write |
| `FILE_CLOSE_FAILED` | 12338 | `CLOSE FILE` failed |
| `FILE_NOT_CLOSE_PREDEF` | 12336 | Cannot close predefined file |

`MAX_PIPE = 500` — maximum lines the pipe buffer supports.

---

## Core Patterns

### Pattern 1: Standard open/read-loop/close

```karel
VAR
  fl     : FILE
  line   : STRING[127]

files__open('MD:/myfile.csv', 'RO', fl)

WHILE BYTES_LEFT(fl) > 0 DO
  files__read_line_ref('MD:/myfile.csv', fl, line)
  -- process line...
ENDWHILE

files__close('MD:/myfile.csv', fl)
```

### Pattern 2: Create-if-missing, append new data

```karel
VAR fl : FILE
files__create('MD:/log.txt', fl)       -- safe: noop if already exists
files__open('MD:/log.txt', 'AP', fl)
files__write_line('session started', 'MD:/log.txt', fl)
files__close('MD:/log.txt', fl)
```

### Pattern 3: Pipe flush to disk (used by csv and display modules)

```karel
VAR
  pip_fl   : FILE
  write_fl : FILE

-- pipe_name is a PIPE: device file (e.g. 'PIPE:mydata')
-- handle_file=TRUE means files module manages the disk file handle
files__write_from_pipe('PIPE:mydata', pip_fl, 'MD:/out.csv', write_fl, TRUE)
```

### Pattern 4: Parse a delimited line into typed struct (used by csv module)

```karel
VAR
  line     : STRING[127]
  result   : t_DATA_TYPE
  dummy_r  : ARRAY[1] OF REAL
  dummy_i  : ARRAY[1] OF INTEGER
  dummy_s  : ARRAY[1] OF STRING

files__read_line_ref('MD:/data.csv', fl, line)
-- parse the 3rd field (offset=3) as a REAL
files__read_type(line, ',', 3, 1, C_REAL, result, dummy_r, dummy_i, dummy_s)
-- result.rl now holds the value
```

### Pattern 5: LS file detection before processing

```karel
VAR fl : FILE
IF files__is_LS('MD:/MYPROG.LS', fl) THEN
  -- treat as TP program: skip to /MN before reading instructions
ELSE
  -- treat as plain text
ENDIF
```

### Pattern 6: Write an LS program file (pattern from tpefile module)

```karel
files__open('MD:/MYPROG.LS', 'RW', newfile)
files__write_line('/PROG MYPROG', 'MD:/MYPROG.LS', newfile)
files__write_line('/MN', 'MD:/MYPROG.LS', newfile)
files__write_line(':  PAUSE ;', 'MD:/MYPROG.LS', newfile)
files__write_line('/POS', 'MD:/MYPROG.LS', newfile)
files__write_line('/END', 'MD:/MYPROG.LS', newfile)
files__close('MD:/MYPROG.LS', newfile)
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling `files__read_line` / `files__write` without opening first | `karelError` 12328 `FILE_NOT_OPENED` abort | Always call `files__open` before any read/write; check that the path string matches exactly (device prefix `MD:/` etc.) |
| Opening a file in `'RW'` mode to append | Data is overwritten from the beginning | Use `'AP'` mode for appending; use `'UD'` to update without truncation |
| Forgetting `CLR_IO_STAT` before reusing a FILE variable | Stale error status causes `check_open` to fire on the second open | `files__open` calls `CLR_IO_STAT` internally — do not reuse a FILE variable across non-`files__open` opens |
| Passing `break_str = ''` or uninit to `files__read_to_sarr` | The `search_str` call is never triggered, entire file is read | Pass a meaningful sentinel string (e.g. `'/END'`) or ensure `break_str` is initialized |
| Calling `files__create` and assuming the file is open afterward | Next `files__write_line` fails with not-opened error | `files__create` closes the file before returning; call `files__open` explicitly after |
| Using `files__read_line` when the string return value exceeds 127 chars | String is silently truncated | Use `files__read_bytes` with an appropriate `buffer_size`, or ensure source data fits in 127 characters |
| Opening a file that is already open in another program | `FILE_MULTIPLE_ACCESS` warning (not abort) allows silent double access | Design programs to own exclusive file handles; use named mutex via `multitask` if concurrent access is needed |

---

## Dependencies

### What `files` depends on

| Module | Used for |
|--------|---------|
| `errors` | `karelError`, `CHK_STAT`, named error constants (`CANT_READ`, `STR_TOO_LONG`, `INVALID_TYPE_CODE`) |
| `Strings` | `search_str` (in `read_to_sarr`), `extract_str`, `s_to_i`, `s_to_r`, `s_to_b`, `s_to_vec`, `s_to_xyzwpr`, `s_to_config`, `s_to_iarr`, `s_to_rarr`, `s_to_arr` (all used in `read_type`) |
| `systemlib` | `t_DATA_TYPE`, `C_INT`, `C_REAL`, etc. (used in `read_type`) |
| `ktransw-macros` | `declare_function`, `funcname`, namespace macros |

### What depends on `files`

| Module | How it uses `files` |
|--------|---------------------|
| `csv` | Delegates all `open/close/clear/create/read_line_ref/write_line/read_type` calls |
| `xmlib` | Wraps `files__open` / `files__close` after setting `ATR_XML` file attribute |
| `socket` | Uses file handle infrastructure for socket stream I/O |
| `TPE` (tpefile) | Line-by-line LS file reading and writing (`read_line_ref`, `write_line`, `open`, `close`) |
| `display` | `write_from_pipe` to flush display pipe to log file |
| `layout` | File buffering for layer-by-layer CSV data loading |

---

## Build / Integration Notes

- **Include order**: `%include files.klt` before `%include files.klh`. The `klh` uses `%ifdef systemlib_datatypes_t` to conditionally declare `files__read_type` — so `systemlib.datatypes.klt` must be included before `files.klh` if you want `read_type`.
- **Device prefix**: FANUC file paths require a device prefix: `MD:` (DRAM disk), `RD:` (RAM disk), `UD1:` (USB), `PIPE:` (in-memory pipe). Always pass the full prefixed path as `filename`.
- **File variable scope**: Declare `fl : FILE` at the program or routine level. Do not share a FILE variable between concurrent tasks without a mutex.
- **Pipe files**: `PIPE:` device files are FANUC in-memory buffers. `MAX_PIPE = 500` is a Ka-Boost–defined soft limit enforced by `display`; the FANUC kernel has its own hard limits.
- **`files__read_to_sarr`**: Opens and closes the file internally. Do not pre-open the file before calling this routine.
- **`files__is_LS`**: Also opens/closes internally. Do not pass an already-open handle.
- **The `test/test_files.kl`** is a stub — it contains no assertions. Testing is done implicitly via `csv` and `TPE` tests.
