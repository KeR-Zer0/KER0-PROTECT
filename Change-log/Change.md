## [1.2.0] - 2026-10-19

### Added

- **`KER0_OPAQUE(bool)`** — opaque predicate macro. Resolves at runtime to the passed `true`/`false` value, but is emitted as 32-bit bitwise arithmetic (`bit32.band`) that static analyzers cannot fold, so both branches of an `if` remain equally plausible to deobfuscators.
- **`KER0_ASSUME(expression, expected)`** — compile-time value snapshot. The macro is replaced by the expected numeric constant and the original expression is removed from the protected output entirely (e.g. `game.PlaceId` no longer appears as a runtime lookup). Behavior is undefined if the assumption does not hold at runtime.
- **`KER0_LINE` as a value** — the macro now works both as `KER0_LINE()` and as a plain value (`local ln = KER0_LINE`).

### Fixed

- Macros inside `while`/`repeat` conditions and `for` loop bounds were silently left unprocessed in the output.
- Statement macros (`KER0_CRASH()`, `KER0_BEGIN_CLIENT_LINES()`) inside nested blocks such as `if ... then ... end` were not processed.
- Macros nested inside function bodies wrapped by `KER0_JIT`, `KER0_JIT_MAX`, `KER0_NO_VIRTUALIZE`, `KER0_INLINE`, `KER0_SECURE_CALL`, `KER0_SECURE_CALLBACK`, `KER0_NO_UPVALUES`, `KER0_ENC_FUNC` or `KER0_ENC_FUNC_SEED` were lost.
- `KER0_BEGIN_CLIENT_LINES` emitted a bogus `: any` type annotation on the generated local.

## [1.1.2] - 2026-10-15

### Added

- New dashboard GUI.

### Fixed

- Various Register VM stability issues.

## [1.1.1] - 2026-06-09

### Fixed

- Macro processing errors.

## [1.1.0] - 2026-02-09

### Added

- Custom anti-tamper protection system.

### Removed

- VM architecture: `none`.
- Security grade: `debug`.
