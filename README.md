# ET:QW SDK 1.5

Enemy Territory: Quake Wars game SDK, version 1.5.

## Branches

### `main`
All fixes required to compile and run `gamex86.so` on a modern Linux host
(GCC 14/15, Python 3, x86_64 host building i386 target). The vanilla SDK
boots, loads maps, runs bots, and accepts connections from retail Windows
clients without crashing.

## Building on modern Linux

### Prerequisites

```
sudo apt install gcc-multilib g++-multilib python3 scons
```

### Build

```
cd trunk
scons BUILD=debug PLATFORM=linux ARCH=x86 TARGET=dedicated gamedll=1 -j$(nproc)
```

The output is `trunk/build/debug/game/sys/scons/libgame.so`. Deploy it as
`gamex86.so` in your server's base directory or alongside the dedicated
server binary.

---

## Fixes applied (`modern-linux-build`)

### 1. Build system ported to Python 3
`sys/scons/scons_utils.py` and `SConstruct` updated for Python 3 syntax
(`print()`, `dict.items()`, `str` vs `bytes` for XML parsing). GCC 15
compile errors suppressed with targeted flags (`-fpermissive`,
`-fno-sized-deallocation`). `--allow-shlib-undefined` added to the game
`.so` link so it builds without the engine present.

### 2. Null-pointer UB fixes (GCC 14+)
Several files had implicit conversions and pointer casts that GCC 14+
rejects as undefined behaviour. Fixed across `Game_local.cpp`,
`gamesys/SysCvar.cpp`, and related headers.

`-fno-delete-null-pointer-checks` added to `SConstruct`: GCC 14+ removes
`return this ? x : 0` null guards as unreachable UB under the C++ standard's
"this is never null" rule. The SDK relies on these guards in `GetHandle()`,
`GetScriptObject()`, and `Class.h` cast methods; without this flag the server
crashes during map entity spawning.

### 3. `STACK_GET` macro corrected
The Linux `STACK_GET` macro in `game/script/Script_DLL.cpp` was changed
from a frame-pointer-relative expression to a direct ESP capture:

```c
#define STACK_GET( value ) __asm__ __volatile__ ( "movl %%esp, %0" : "=r" ( value ) : : )
```

This ensures all three call sites (init / save / restore) in
`Coroutine_Enter` capture the same ESP value relative to EBP, which is the
invariant the coroutine save/restore algorithm depends on.

### 4. i386 PIC GOT corruption in `Coroutine_Enter` (root crash fix)
**Problem.** On i386 PIC shared libraries, GCC caches the GOT base register
(`%ebx`) in a local stack slot (`[ebp-0x1c]`) in the prologue and reloads
it from that slot after every external call. `Coroutine_Enter`'s RESTORE
path runs with that slot corrupted: the coroutine that executed between
save and restore used the same physical stack memory and overwrote it.
Every subsequent global variable access or PLT call in `Coroutine_Enter`
loaded the wrong `%ebx` and faulted with `SEGV_ACCERR` (write into
read-only `.text`).

**Fix.** Four `__attribute__((noinline))` helper functions, each with its
own function prologue that calls `__x86.get_pc_thunk.*` to compute a fresh,
independent GOT base:

- `context_jump_is_set()` / `context_jump_test_and_clear()` /
  `context_jump_set()` — access the `context_jump` flag through their own
  fresh GOT, bypassing `Coroutine_Enter`'s corrupted `[ebp-0x1c]`.

- `coroutine_restore_path(char* stack_ptr)` — reads `s_current`, checks
  `IsStackSaved()`, runs the restore `memcpy`, and calls `FreeStack()`, all
  with a fresh GOT. `Coroutine_Enter` captures ESP via `STACK_GET` (inline
  asm, no GOT needed) and passes it in before the call. The call itself is
  a module-relative `e8` instruction (no PLT, no `%ebx` required). After
  the `memcpy` restores the stack, the epilogue's `pop`/`ret` sequence reads
  back the correct saved registers and return address.

### 5. Uninitialized `newData.teleporting` in `sdTransport` broadcast write
**Problem.** `sdTransport::WriteNetworkState` (broadcast mode) wrote
`newData.teleporting` to the network message via `WriteBool` but never
assigned it from `IsTeleporting()` before writing. On a freshly-allocated
`newData` buffer the field held uninitialized stack memory — typically `10`
on this build — which `WriteBool` rejected as `idBitMsg::WriteBits: value
overflow 10 1`. Bots never trigger `ServerWriteSnapshot`, so the bug stayed
dormant until the first human client connected.

**Fix.** Set `newData.teleporting = IsTeleporting();` in
`WriteNetworkState`. Add the matching `IsTeleporting() != baseData.teleporting`
check to `CheckNetworkStateChanges` so a teleport transition is detected as
a delta. Both edits are in `game/vehicles/Transport.cpp`.

### 6. `SD_PUBLIC_BUILD` define for retail-client wire compatibility (root protocol fix)
**Problem.** The retail Windows ETQW client cannot read snapshots produced
by an SDK Linux server built with the stock SCons configuration: the client
disconnects immediately on connect with
`ERROR: Entitynum (-1) out of range`.

The cause is a wire-protocol mismatch driven by the `ASYNC_SECURITY` macro
in `game/Common.h`:

```c
#ifndef SD_PUBLIC_BUILD
    #define ASYNC_SECURITY
#endif

#ifdef ASYNC_SECURITY
#define ASYNC_SECURITY_READ( STREAM )                                          \
    if ( STREAM.ReadLong() != ASYNC_MAGIC_NUMBER ) {                          \
        gameLocal.Error( "Network State Corrupted: %s(%d)", __FILE__, __LINE__ ); \
    }
#define ASYNC_SECURITY_WRITE( STREAM )                                         \
    STREAM.WriteLong( ASYNC_MAGIC_NUMBER );
#else
#define ASYNC_SECURITY_READ( STREAM )
#define ASYNC_SECURITY_WRITE( STREAM )
#endif
```

`ASYNC_MAGIC_NUMBER` is `12345678` (`0xBC614E`). When `ASYNC_SECURITY` is
on, `ASYNC_SECURITY_WRITE`/`ASYNC_SECURITY_READ` macros (active in
`idPlayer::WritePlayerStateBroadcast` and `ReadPlayerStateBroadcast`,
bracketing the inventory section in `Player.cpp:12076-12112`) inject 32-bit
sentinel longs into the stream so the read side can verify it stayed in sync.

The retail Windows client is built with `SD_PUBLIC_BUILD` defined (see
`_SDK.vsprops`), so `ASYNC_SECURITY` is **off** in the retail build and the
macros expand to nothing. The Linux SCons build did **not** define
`SD_PUBLIC_BUILD`, so `ASYNC_SECURITY` was **on**, and the server wrote
sentinels the client did not read. Each unread 32-bit sentinel shifted the
client's read cursor 32 bits behind the server's write cursor; after enough
broadcast sections the client's `ReadBits(GENTITYNUM_BITS)` call ran past
the end of the message and `BitMsg.cpp:481` returned `-1`, which the
caller surfaced as "`Entitynum (-1) out of range`".

**Fix.** Match the Windows `_SDK.vsprops` by appending `-DSD_PUBLIC_BUILD`
alongside the existing `-DSD_SDK_BUILD` in the SDK block of `SConstruct`:

```python
if ( g_sdk ):
    BASECPPFLAGS.append( '-DSD_SDK_BUILD' )
    BASECPPFLAGS.append( '-DSD_PUBLIC_BUILD' )
    BASECPPFLAGS.append( '-Igame' )
```

Building with `BETA=1` is **not** an acceptable substitute: `BETA=1` defines
`SD_PUBLIC_BETA_BUILD`, which transitively defines `SD_PUBLIC_BUILD` but
also enables `SD_ENCRYPTED_FILE_IO`, `SD_RESTRICTED_FILESYSTEM`, and
`SD_EXPIRE_BUILD` (the build refuses to run after a hardcoded date).

**How it was diagnosed.** A WB-level dump of every `WriteBits` call during
the first snapshot surfaced `WriteBits val=12345678 nb=32` for
`BC_<n>_idPlayer_STATE`. Grepping for `12345678` in the source led to
`ASYNC_MAGIC_NUMBER`, then to the `ASYNC_SECURITY_*` macros, then to the
`SD_PUBLIC_BUILD` gate. Comparing `_SDK.vsprops` (the Windows SDK build's
preprocessor definitions: `SD_SDK_BUILD;SD_PUBLIC_BUILD`) confirmed the
asymmetry between the SCons build and the retail build it must
interoperate with.
