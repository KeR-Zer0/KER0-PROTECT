---
name: ker0-obfuscation
description: "Ker0 (Ker0 obfuscator / Ker0 Protect / Lua-Luau protector) compile-time macros, macro stubs for unprotected/dev runs, script protection, and post-obfuscation debugging. Use when the user mentions Ker0, ker0-protect.pp.ua, KER0_OBFUSCATED, KER0_ENC_STRING, KER0_ENC_NUM, KER0_ENC_FUNC, KER0_ENC_FUNC_SEED, KER0_ASSUME, KER0_OPAQUE, KER0_SECURE_CALL, KER0_SECURE_CALLBACK, KER0_INLINE, KER0_NO_VIRTUALIZE, KER0_JIT, KER0_JIT_MAX, KER0_NO_UPVALUES, KER0_IS_CALLER_KER0, KER0_GET_RNG_SEED, KER0_LINE, KER0_BEGIN_CLIENT_LINES, KER0_CRASH, or asks to obfuscate/protect/harden a Lua script, wrap sensitive code (API keys, tokens, license checks, secrets), stub macros so the script runs without the obfuscator, or debug a crash that only happens after obfuscation (e.g. '[string "Ker0"]:269: attempt to call a number value')."
metadata:
  {
    "openclaw":
      {
        "notes": "Ker0 macros are compile-time only. They exist ONLY in a protected build; in an unprotected run they are nil and must not be reached."
      }
  }
---
# Ker0 Obfuscation Skill

Ker0 is a compile-time protector for Lua/Luau scripts. It rewrites a script at
build time, virtualizes and encrypts selected regions, and exposes a set of
**compile-time macros** that the protector expands while it processes the
script.

Two jobs this skill covers, and they are almost always the same job seen from
two sides:

1. **Debug** a script that broke *after* protection (`[string "Ker0"]:NNN: ...`).
2. **Harden** a script: wrap its sensitive parts (keys, tokens, license logic)
   in the right Ker0 macros.

The macro catalog below is the single source of truth for what each macro
does and **what type it evaluates to**. The type column is the fastest way to
diagnose the classic Ker0 crash: `attempt to call a number value`.

---

## 0. Start here — routing

| Situation | Go to |
|---|---|
| "После обфускации ошибка", `attempt to call a number value`, any `[string "Ker0"]` trace | **§3** Workflow A |
| "Защити ключ/секреты/лицензию" | **§4** Workflow B |
| "Как ускорить / как сделать детерминированно" | **§5** Workflow C |
| "Скрипт падает без обфускатора" / "нужен dev-режим" | **§7** Stubs |
| "Проверить, что защита не слита" | **§8** Verification |
| "Обфусцируй / защити / собери билд" | **§8.0** build step — ker0-protect.pp.ua |
| "Не знаю, что делает макрос" | **§2** Catalog + **§2.7** calibration harness |

### 0.1 When NOT to use this skill

* Another obfuscator (MoonVeil, Prometheus, IronBrew, hide.lat, LuaObfuscator) —
  those have different macro sets. Say so instead of guessing Ker0 semantics.
* Plain Lua/Luau questions with no Ker0 macro in sight.
* General "how do I protect my Roblox game" questions — that needs a threat
  model, not macros (see **§11** on client-side secrets).

### 0.2 Operating modes

| Mode | When | What you may change |
|---|---|---|
| **Diagnose** | The user reports a crash | Propose a diff; do not rewrite unrelated code |
| **Protect** | The user asks to harden a script | Add/remove macros per the inventory (§4, Step 1) |
| **Audit** | The user asks whether an existing protected script is safe | Report only; no edits without approval |

Pick the mode from the request. If the request is ambiguous ("почини и заодно
защити"), ask one focused question before editing.

### 0.3 Output language

**Answer in the user's language.** If the user writes in Russian, answer in
Russian; in English, answer in English; in Ukrainian/Spanish/etc., answer in
that language. If the language is unclear or the request is a bare code dump
with no prose, default to English.

What stays verbatim regardless of language:

* macro names (`KER0_ENC_STRING`), code, file paths, diffs;
* error messages and stack traces — quote them exactly as the executor printed
  them, do not translate or reformat them;
* section references like (§8).

The user-facing handoff line in §12 is written in both languages — pick the
one that matches the user's language and use it verbatim.

---

## 1. Non-negotiable invariants

Read these before touching any script. Violating them is what produces
"worked in the editor, broke after obfuscation" bugs.

1. **Macros are compile-time constructs.** They are expanded by Ker0, not
   executed. In an unprotected run they are `nil` identifiers, so touching one
   gives `attempt to call a nil value` (or `attempt to index a nil value`).
2. **A protected script must be protected.** Standard header:

   ```lua
   if not KER0_OBFUSCATED then
       error("Script must be obfuscated!")
   end
   ```

   Put this at the very top of the file, before any macro use.
3. **A macro changes the type of the expression it wraps.** Always check the
   "Evaluates to" column before using a macro in a position that expects a
   different type. This is the #1 cause of `attempt to call a number value`.
4. **Macro names are case-sensitive and reserved.** `KER0_*` is Ker0's
   namespace. Never define your own variable with a `KER0_` prefix.
5. **Macros must be written literally.** `KER0_ENC_STRING("x")` works;
   `_G["KER0_ENC_STRING"]("x")`, `local f = KER0_ENC_STRING; f("x")` and
   `"KER0_ENC_STRING(\"x\")"` do **not** — the protector only recognizes the
   literal call form in the source text.
6. **Macro arguments are compile-time values.** For `ASSUME`, `ENC_*`,
   `OPAQUE`, `LINE`, `GET_RNG_SEED` the argument is consumed by the
   protector, not at runtime. A runtime-varying expression there is either an
   error or undefined behaviour. `ASSUME`'s first argument is the only one
   that may reference runtime state — and it is still *discarded* (§2.2).
7. **One protection layer per function by default.** Do not stack
   `KER0_ENC_FUNC(KER0_SECURE_CALL(fn))` unless the user asks for it — nested
   layers change call semantics and are a common source of "works in dev,
   fails in prod".
8. **Do not delete or rewrite user logic.** Protection changes representation,
   not behavior. A fix must preserve observable behavior.
9. **Ker0 is an external tool.** You edit source; the **user** builds it at
   <https://ker0-protect.pp.ua/>. The site sits behind a Cloudflare
   Turnstile human check, so it cannot be automated — the handoff to the user
   is mandatory, not optional. **Never state that a fix is verified in a
   protected build unless you actually ran Ker0.** Say
   "I can't run Ker0 here — protect it at ker0-protect.pp.ua and check these
   three things".
10. **A stub must behave like the macro it replaces.** If the dev stub
    returns something different from the protected macro, the dev build
    becomes a liar and you ship the "works in dev, fails in prod" bug. This
    is the single most important rule in this file.

---

## 2. The macro catalog

`Evaluates to` = the runtime type of the value the macro expands to.
`Status` = how confident we are about the described semantics:

* `doc` — taken from Ker0's own macro description.
* `inferred` — deduced from the macro's name/description, not stated outright.
* `unverified` — ambiguous or undocumented; resolve it with the calibration
  harness in **§2.7** before relying on it.

### 2.1 Build and identity

| Macro | Evaluates to | Status | What it does |
|---|---|---|---|
| `KER0_OBFUSCATED` | `boolean` | doc | Build-time flag that becomes `true` after the script is processed by Ker0. Use it to hard-fail an unprotected run. |
| `KER0_IS_CALLER_KER0()` | `boolean` | doc | Resolves to `true` at compile time — lets internal guards verify they run inside protected code. |
| `KER0_GET_RNG_SEED()` | `number` | doc | Replaced at compile time with a fresh random seed — per-build keys that change on every protection run. |
| `KER0_LINE` | `number` | doc | Replaced at compile time with the current line number. Works **without** enabling line tracking. |
| `KER0_CRASH()` | *(aborts)* | doc | Aborts execution. Intended for anti-debug / anti-tamper branches only. |

```lua
if not KER0_OBFUSCATED then
    error("Script must be obfuscated!")
end

if not KER0_IS_CALLER_KER0() then
    error("Run this inside Ker0 protection!")
end

local firstLine = KER0_LINE
print("Current line: " .. KER0_LINE)

local buildKey = KER0_GET_RNG_SEED()

if isDebuggerAttached() then
    KER0_CRASH()
end
```

### 2.2 Compile-time constants and predicates

| Macro | Evaluates to | Status | What it does |
|---|---|---|---|
| `KER0_ASSUME(expr, expected)` | `number` | **unverified** | Compile-time assumption. The whole macro is replaced by the numeric constant `expected`; `expr` is consumed by the protector and never appears in the protected output. `expr` must be a side-effect-free numeric expression. See the note below — run the harness in §2.7 to confirm. |
| `KER0_OPAQUE(bool)` | `boolean` | doc | Generates an opaque predicate that always resolves to the passed true/false value. Control-flow checkers and data-flow analyzers cannot tell which branch runs. Pair it with `KER0_ASSUME` for stronger predicates. |

```lua
local placeId = KER0_ASSUME(game.PlaceId, 123456789)
local token   = KER0_ASSUME(someResult(), 0xABCDEF)

if KER0_OPAQUE(true) then
    validateWhitelist()
else
    -- never runs, analyzers cannot prove it
end
```

> **`KER0_ASSUME` returns a number, never a function.** The macro expands to
> the constant `expected` (or, if Ker0 instead snapshots `expr`, to a number
> either way). The original function is gone. If the code then calls it →
> `attempt to call a number value`. This is the single most common Ker0 bug
> in user scripts.
>
> **Read the two arguments as: "assume `expr` equals `expected`, and inline
> `expected`".** That is the only reading consistent with the examples above:
> `game.PlaceId` is not knowable at compile time, so the substituted constant
> has to be `123456789`. The stub in §7 mirrors this reading. If the
> calibration harness in §2.7 prints `2` for `KER0_ASSUME(1 + 1, 42)` instead
> of `42`, Ker0 actually snapshots `expr` — change the stub in §7 to
> `function(expr) return expr end` and note the correction here.

### 2.3 Encryption

| Macro | Evaluates to | Status | What it does |
|---|---|---|---|
| `KER0_ENC_STRING(str)` | `string` | doc | Adds enhanced dynamic encryption for sensitive strings. Decrypted at runtime. |
| `KER0_ENC_NUM(num)` | `number` | doc | Encrypts numeric constants with expressions that unfold at runtime. |
| `KER0_ENC_FUNC(fn)` | `function` | doc | Encrypts the wrapped function body with a compiler-generated key so its logic is never stored in readable form. |
| `KER0_ENC_FUNC_SEED(fn, seed)` | `function` | doc | Same as `KER0_ENC_FUNC`, but with a fixed compile-time seed — use it when the function must decrypt identically across builds. |

```lua
local apiKey = KER0_ENC_STRING("secret")
local magic  = KER0_ENC_NUM(0xDEADBEEF)

local hidden = KER0_ENC_FUNC(function()
    return "sensitive logic"
end)

local stable = KER0_ENC_FUNC_SEED(function()
    return 42
end, 0xC0FFEE)
```

> `KER0_ENC_NUM` expands to a **number**, so it must never sit in a callee
> position (`KER0_ENC_NUM(1)()` → `attempt to call a number value`).
>
> `KER0_ENC_STRING` returns a **plain string value** at runtime — the
> decryption happens when the macro is evaluated, not when the string is
> later used. Store it in a local and reuse the local.

### 2.4 Call protection, inlining and performance

| Macro | Evaluates to | Status | What it does |
|---|---|---|---|
| `KER0_SECURE_CALL(fn)` | `function` | doc | Restricts function calls to only protected internal VM calls. |
| `KER0_SECURE_CALLBACK(fn)` | `function` | doc | `KER0_SECURE_CALL` for callbacks: a function handed to other code (event handlers, hooks) that may only be invoked from protected code. |
| `KER0_INLINE(fn)` | `function` | doc | Forces the protected build to inline the function at its call sites instead of emitting a standalone copy. |
| `KER0_NO_VIRTUALIZE(fn)` | `function` | doc | Excludes a hot function from virtualization if it needs maximum speed. |
| `KER0_JIT(fn)` | `function` | doc | Enables first-level bytecode optimization for protected execution regions. |
| `KER0_JIT_MAX(fn)` | `function` | doc | Maximum acceleration for critical inner loops. |
| `KER0_NO_UPVALUES(fn)` | `function` | doc | Compiles the function without upvalue capture — nothing from the enclosing scope leaks into it, removing a common hooking target. |

```lua
local internal = KER0_SECURE_CALL(function()
    return true
end)

table.insert(handlers, KER0_SECURE_CALLBACK(function(payload)
    return validate(payload)
end))

local twice = KER0_INLINE(function(x)
    return x * 2
end)

local fastAdd = KER0_NO_VIRTUALIZE(function(a, b)
    return a + b
end)

local compute = KER0_JIT(function(arr)
    local sum = 0
    for i = 1, #arr do sum = sum + arr[i] end
    return sum
end)

local renderLoop = KER0_JIT_MAX(function(dt)
    -- high performance loop
end)

local pure = KER0_NO_UPVALUES(function(x)
    return x + 1
end)
```

**Choosing between `KER0_SECURE_CALL` and `KER0_SECURE_CALLBACK`:**

| Who calls the function | Use |
|---|---|
| Only other protected code in the same script | `KER0_SECURE_CALL` |
| Roblox engine, an executor, another system, or a third-party hook | `KER0_SECURE_CALLBACK` |
| A function stored in a table and invoked from several places | `KER0_SECURE_CALLBACK` — it is the more permissive wrapper |

### 2.5 Line numbering / error reporting

| Macro | Evaluates to | Status | What it does |
|---|---|---|---|
| `KER0_BEGIN_CLIENT_LINES()` | *(block opener)* | doc (opener) / **unverified** (closer) | Starts a client line-numbering block: defines the base constant used to remap line numbers in client-visible error traces. |

```lua
KER0_BEGIN_CLIENT_LINES()

-- Everything below belongs to the client-visible line-numbering block.
-- Runtime stack traces now report THESE line numbers instead of the
-- protected chunk's virtual ones.
local function validate(payload)
    if type(payload) ~= "table" then
        error("validate: bad payload")     -- trace shows the real source line
    end
    return payload.value
end

local function dispatch(kind, payload)
    if kind == "validate" then
        return validate(payload)            -- line 269 in the old trace = this line
    end
    error("unknown kind: " .. tostring(kind))
end
```

```lua
-- Companion use: report the exact source line yourself, no line tracking needed
local function checkLicense(key)
    if key == nil then
        error("license check failed at source line " .. KER0_LINE)
    end
end
```

Rules of thumb for this macro:

* Put it **at the top of the file**, right after the `KER0_OBFUSCATED` guard,
  before any code you want line-accurate.
* It affects **client-visible error traces only** — it does not change program
  behavior and does not affect `KER0_LINE`.
* `KER0_LINE` gives you the current line inside your own `error()` calls;
  `KER0_BEGIN_CLIENT_LINES()` makes the *runtime* stack trace point at the
  real source line. Use both for maximum diagnosability.

*Reference note:* the upstream description of this macro stops at "defines the
base constant used to remap line numbers in client-visible error traces". It
is documented as the **opener** of a line-numbering block, so a matching
block-closer is expected by the runtime. **Verify the exact closing macro
name in the current Ker0 documentation before emitting it — do not guess a
name into user code.** See §10 (Known unknowns).

### 2.6 Cheat sheet: macro → resulting type

Use this table to reason about type errors without re-reading the catalog.

| Macro | Result type | Safe in a callee position? |
|---|---|---|
| `KER0_OBFUSCATED` | boolean | n/a (not callable) |
| `KER0_LINE` | number | **No** |
| `KER0_ASSUME(e, x)` | number | **No** |
| `KER0_OPAQUE(b)` | boolean | **No** |
| `KER0_GET_RNG_SEED()` | number | **No** |
| `KER0_ENC_STRING(s)` | string | **No** |
| `KER0_ENC_NUM(n)` | number | **No** |
| `KER0_ENC_FUNC(f)` | function | Yes |
| `KER0_ENC_FUNC_SEED(f, s)` | function | Yes |
| `KER0_SECURE_CALL(f)` | function | Yes |
| `KER0_SECURE_CALLBACK(f)` | function | Yes (but see restrictions) |
| `KER0_INLINE(f)` | function | Yes |
| `KER0_NO_VIRTUALIZE(f)` | function | Yes |
| `KER0_JIT(f)` | function | Yes |
| `KER0_JIT_MAX(f)` | function | Yes |
| `KER0_NO_UPVALUES(f)` | function | Yes |
| `KER0_IS_CALLER_KER0()` | boolean | **No** |
| `KER0_CRASH()` | *(aborts)* | n/a |
| `KER0_BEGIN_CLIENT_LINES()` | *(block opener)* | n/a (not callable) |

### 2.7 Calibration harness — resolve the `unverified` semantics once

Ker0's exact semantics are not fully pinned down by its macro descriptions.
Run this **once** in a protected build (a throwaway script, not the user's
code) and keep the output next to your notes. Every `unverified` row in the
catalog is settled by this.

```lua
-- ker0_selftest.lua — run ONCE inside a protected build.
-- Do NOT include KER0_CRASH here: it aborts the process.
local function t(name, f)
    local ok, res = pcall(f)
    print(string.format("%-24s %-6s %s", name, tostring(ok), tostring(res)))
end

t("OBFUSCATED",          function() return KER0_OBFUSCATED end)
t("IS_CALLER_KER0",      function() return KER0_IS_CALLER_KER0() end)
t("GET_RNG_SEED",        function() return KER0_GET_RNG_SEED() end)
t("LINE",                function() return KER0_LINE end)
-- ASSUME discriminators: 2 => Ker0 snapshots expr, 42 => Ker0 uses `expected`
t("ASSUME(1+1, 42)",     function() return KER0_ASSUME(1 + 1, 42) end)
t("ASSUME(0, 7)",        function() return KER0_ASSUME(0, 7) end)
t("OPAQUE(false)",       function() return KER0_OPAQUE(false) end)
t("ENC_STRING",          function() return KER0_ENC_STRING("abc") end)
t("ENC_NUM",             function() return KER0_ENC_NUM(7) end)
t("ENC_NUM type",        function() return type(KER0_ENC_NUM(7)) end)
t("ENC_FUNC type",       function() return type(KER0_ENC_FUNC(function() return 1 end)) end)
t("ENC_FUNC_SEED type",  function() return type(KER0_ENC_FUNC_SEED(function() return 1 end, 1)) end)
t("SECURE_CALL type",    function() return type(KER0_SECURE_CALL(function() return 1 end)) end)
t("SECURE_CALLBACK type",function() return type(KER0_SECURE_CALLBACK(function() return 1 end)) end)
t("INLINE type",         function() return type(KER0_INLINE(function() return 1 end)) end)
t("NO_VIRTUALIZE type",  function() return type(KER0_NO_VIRTUALIZE(function() return 1 end)) end)
t("JIT type",            function() return type(KER0_JIT(function() return 1 end)) end)
t("JIT_MAX type",        function() return type(KER0_JIT_MAX(function() return 1 end)) end)
t("NO_UPVALUES type",    function() return type(KER0_NO_UPVALUES(function() return 1 end)) end)
t("BEGIN_CLIENT_LINES",  function() KER0_BEGIN_CLIENT_LINES(); return "ok" end)
```

How to read the two `ASSUME` rows:

| Output | Meaning | Action |
|---|---|---|
| `42` and `7` | Ker0 substitutes `expected`; `expr` is discarded | The §7 stub `function(_expr, expected) return expected end` is correct |
| `2` and `0` | Ker0 snapshots `expr` at compile time | Change the §7 stub to `function(expr) return expr end` and correct §2.2 |

Also use the harness to discover the `KER0_BEGIN_CLIENT_LINES()` closer: if
protecting a file with the opener fails with "unterminated block", the error
message usually names the closer — use that exact name.

---

## 3. Workflow A — "it broke after obfuscation"

Trigger phrases: *"после обфускации ошибка"*, *"after obfuscation I get
`attempt to call a number value`"*, *"only crashes in the protected build"*,
any `[string "Ker0"]` stack trace.

### Step 1 — Collect the facts (never guess before this)

Ask for / read from the user:

- the **exact error text**, including the chunk name and the line number;
- the **full stack**, every frame;
- the **source**: a file path (read it) or a pasted script.

If the user attached the script as a file in the workspace, read it from disk.
If they pasted it, work from the paste. Do not ask for a path you already have.

### Step 2 — Read the stack trace anatomically

Example executor output:

```
[string "Ker0"]:269: attempt to call a number value
  Stack Begin
    Script '[string "Ker0"]', Line 269
    Script '[string "Ker0"]', Line 251
  Stack End
```

| Part | Meaning |
|---|---|
| `[string "Ker0"]` | Chunk name of the **protected** chunk (compiled from a string named `Ker0`). It is not a file path — you cannot open it. |
| `:269:` | The line inside the protected chunk where the failing call happened (**innermost** frame). |
| `Line 269` / `Line 251` | The same protected-chunk lines for each frame, innermost first. `251` is the **caller** of the function that failed at `269`. |
| `attempt to call a number value` | Luau/Lua tried to call a value whose type was `number`. The *callee* is a number, not the arguments. |

Two consequences:

- The reported lines are **protected-chunk lines, not your source lines**, unless
  client line remapping is enabled. Do not try to `sed` line 269 of the source.
- The innermost frame is the crime scene; outer frames are only the call path.

### Step 3 — Get real line numbers (do this early, it pays for itself)

Ask the user to add, near the top of the script, right after the guard:

```lua
KER0_BEGIN_CLIENT_LINES()
```

and to re-protect + re-run. Now the stack shows the **source** lines and the
diagnosis becomes mechanical. Meanwhile, use `KER0_LINE` to pin exact
locations in the source you are reading:

```lua
error("license check failed at source line " .. KER0_LINE)
```

If the user cannot re-run right now, proceed with a structural analysis of the
source (Step 5) — the taxonomy below is usually enough.

### Step 4 — Classify the error

Common Luau runtime errors seen after Ker0 protection, with the most likely
cause and the macro-level fix:

| Error | Most likely cause | Fix direction |
|---|---|---|
| `attempt to call a number value` | A number landed in a callee position (macro expansion, or a table lookup returning a number). | See §3, Step 5. |
| `attempt to call a nil value` | Missing global, key mismatch (encrypted string vs literal), function defined in another chunk, or protection removed the definition. | Encrypt *both* sides of a lookup; keep definitions and uses in the same file. |
| `attempt to call a table value` / `attempt to call a table` | A table (not a function) is being invoked — e.g. `obj.method` where `method` is a table of handlers. | Call the entry: `obj.method[kind](...)`. |
| `attempt to index a nil value` | The table itself is missing. | Check that the table is assigned before use and is not shadowed by a macro expansion. |
| `attempt to index a number value` | A numeric constant replaced the table. | Look for `KER0_ASSUME` / `KER0_ENC_NUM` where a table was expected. |
| `attempt to concatenate a nil value` | A string slot is `nil` — often an encrypted string used before decryption or a missing field. | Ensure `KER0_ENC_STRING` result is stored in a local first, then used. |
| `attempt to compare number with nil` | Type mismatch introduced by protection of a comparison operand. | Keep both sides of a comparison behind the same protection mode. |
| `attempt to perform arithmetic on a table value` | A constant table was protected into an opaque form and arithmetic is applied to it. | Keep numeric math on plain locals, not on table fields. |
| `stack overflow` | Deep recursion through a virtualized function, or a protected callback re-entering protected code. | Use `KER0_NO_VIRTUALIZE` for the recursive function; remove `KER0_SECURE_CALLBACK` from code external systems call. |
| `C stack overflow` / `error in error handling` | Debug/hook-based anti-tamper code fighting the VM. | Remove or narrow `KER0_CRASH` / anti-debug blocks. |

### Step 5 — `attempt to call a number value`: the checklist

The callee was a number. Walk this list **in order** against the actual source:

1. **Macro misuse in callee position.** Search the source for
   `KER0_ASSUME(`, `KER0_ENC_NUM(`, `KER0_LINE`, `KER0_GET_RNG_SEED()` used
   where a function is then called, including inside a table constructor:
   ```lua
   -- WRONG: expands to a numeric constant; the function is gone
   local actions = { start = KER0_ASSUME(startAction, 0) }
   actions.start()            -- attempt to call a number value

   -- RIGHT: keep a real callable
   local actions = { start = KER0_ENC_FUNC(startAction) }
   actions.start()
   ```
2. **Double-call pattern.** `f()()` where `f()` returns a number instead of a
   function; or `t[k]()` where `t[k]` is a numeric field.
3. **Numeric index into a handler table.** `local fn = handlers[kind]; fn()` where
   `kind` is a number and `handlers` is a *flat* table of numbers, or the
   handler table was replaced by protected constants.
4. **Detached method on a numeric object.** `local m = obj.method` where `obj`
   was replaced by a number; `obj:method()` where `obj` is `nil`/number.
5. **Arithmetic folding.** `a + b()` — parenthesization changed by the
   protector, or `a .. b()` with a numeric `b`.
6. **Protection artifact.** The protector replaced a function with a numeric
   handle/VM slot and a call site still calls the handle directly. Mitigations:
   - wrap the callee in `KER0_ENC_FUNC(fn)` so a real callable is emitted,
   - wrap the hot function in `KER0_NO_VIRTUALIZE(fn)` to keep a native copy,
   - wrap the caller in `KER0_INLINE(fn)` so the callee is inlined as code
     instead of referenced indirectly.
7. **Reflection removed.** `getfenv`, `debug.getinfo`, `string.dump` style
   code that used to find a function by name now resolves to a number or
   `nil` after protection. Replace reflection with a direct reference, or keep
   the reflective code outside the protected region.

To confirm the hypothesis cheaply, have the user insert a probe in the source
(not in the protected output) right before the call:

```lua
print("[probe] callee type:", type(fn), tostring(fn))
```

`type(fn) == "number"` confirms the diagnosis and the printed value often
identifies the exact source construct (a magic constant, a table field, a
macro expansion).

### Step 6 — Patch and protect

Apply the minimal change, then ask: *does this region need protection?* If the
callee is sensitive (license logic, token check), wrap it with
`KER0_ENC_FUNC` / `KER0_SECURE_CALL` rather than leaving it naked. If it is a
hot path, use `KER0_NO_VIRTUALIZE` or `KER0_JIT`.

### Step 7 — Verify and report

- Re-protect with Ker0 at <https://ker0-protect.pp.ua/> and run the script
  (see §8.0 — the user does this step, not you).
- Confirm the previous error is gone **and** that no new error appeared.
- Report: root cause → evidence (line mapping) → the exact diff → which macros
  were added and why → remaining risks.
- Close with the handoff line from §12.1, in the user's language.

---

## 4. Workflow B — "protect my key system / sensitive parts"

Trigger phrases: *"нужно защитить ключ систему и её чувствительные места"*,
*"protect my license system"*, *"hide my API key"*, *"secure my token"*.

### Step 1 — Inventory the sensitive assets

Read the script and list what must not be readable in a dumped/hooked build:

| Sensitive asset | Typical shape | Macro |
|---|---|---|
| API keys, tokens, secrets, passwords | string literals | `KER0_ENC_STRING` |
| Magic constants, hashes, build IDs, secret offsets | numeric literals | `KER0_ENC_NUM` |
| License / entitlement validation logic | whole function | `KER0_ENC_FUNC` (or `KER0_ENC_FUNC_SEED` for reproducible builds) |
| License server URL, webhook endpoint | string literal | `KER0_ENC_STRING` |
| Anti-tamper / anti-debug checks | function + branch | `KER0_OPAQUE` + `KER0_ASSUME` + `KER0_CRASH` |
| Callback handed to an external system (event handler, hook) | function | `KER0_SECURE_CALLBACK` |
| Key derivation / HMAC helper | function | `KER0_ENC_FUNC` |
| Pure helper that must not see outer scope | function | `KER0_NO_UPVALUES` |
| Hot loop inside protected code | function | `KER0_JIT` / `KER0_JIT_MAX` |
| Per-build randomization | value | `KER0_GET_RNG_SEED()` |
| Reproducible key across builds | value | `KER0_ENC_FUNC_SEED(fn, seed)` |

**Do not protect everything.** Encrypting every string costs startup time,
makes diffs unreadable, and buys nothing. Protect the assets above and leave
the rest in the clear. See also §11 before promising the user anything about
secrets.

### Step 2 — Apply in a consistent order

1. **Strings first** — `KER0_ENC_STRING` on every secret literal.
2. **Numbers** — `KER0_ENC_NUM` on magic constants and secret offsets.
3. **Functions** — `KER0_ENC_FUNC` on the sensitive logic.
4. **Guards** — `KER0_OBFUSCATED` header, `KER0_IS_CALLER_KER0()` inside
   protected-only entry points.
5. **Callbacks** — `KER0_SECURE_CALLBACK` for anything external code invokes.
6. **Performance** — `KER0_NO_VIRTUALIZE` / `KER0_JIT` / `KER0_JIT_MAX` for
   hot paths so protection does not cost frame time.
7. **Diagnostics** — `KER0_BEGIN_CLIENT_LINES()` near the top.

### Step 3 — The encryption pattern that actually works

Encrypted strings must be **the same value** wherever the key is needed.
Decrypt once into a local and reuse the local:

```lua
local KEY_HEADER = KER0_ENC_STRING("X-Ker0-Key")
local KEY_BODY  = KER0_ENC_STRING("sk_live_9f2b...")

local function request(url)
    return HTTP.request({
        Url = url,
        Headers = { [KEY_HEADER] = KEY_BODY },
    })
end
```

```lua
-- RISKY: the read side uses a raw literal, the write side an encrypted one
local CONFIG = {}
CONFIG[KER0_ENC_STRING("api_key")] = "value"
local v = CONFIG["api_key"]      -- nil → later "attempt to call a nil value"
```

```lua
-- SAFE: one local, one value, used on both sides
local k = KER0_ENC_STRING("api_key")
local CONFIG = {}
CONFIG[k] = "value"
local v = CONFIG[k]
```

### Step 4 — Keep the unprotected build usable

Because macros only exist inside a protected build, a script that uses them
**cannot run unprotected**. Options, in order of preference:

- Keep the dev version as a separate file without macros.
- Guard at the top so the failure is loud and obvious:
  ```lua
  if not KER0_OBFUSCATED then
      error("Script must be obfuscated!")
  end
  ```
- **Install macro stubs** so the same file runs in both modes — see §7 for the
  stub pattern, the ready-made stub block, the shadowing trap and the three
  placement options (dev header / guarded global injection / dev copy).
- Put macros only inside a branch that unprotected runs never reach, and make
  the unprotected branch explicit:
  ```lua
  if not KER0_OBFUSCATED then
      error("Script must be obfuscated!")
  end
  ```

Tell the user which option you applied and why. Never silently produce a file
that cannot run outside Ker0 without saying so.

### Step 5 — Report the protection plan

Produce a table the user can review:

| Asset (file:line) | Macro applied | Effect | Risk / note |
|---|---|---|---|
| `apiKey` literal, L14 | `KER0_ENC_STRING` | secret not present in plaintext | must be decrypted before use as a table key |
| license check, L88-140 | `KER0_ENC_FUNC` | body not stored readable | re-obfuscating changes the key — fine, body is decrypted at runtime |
| `isDebuggerAttached` branch, L22 | `KER0_OPAQUE` + `KER0_CRASH` | anti-debug | never place on a path a normal user can reach |
| hot loop, L200 | `KER0_JIT_MAX` | speed under protection | semantics must stay identical |

Then close with the handoff line from §12.1: the user protects the script at
<https://ker0-protect.pp.ua/>, obfuscates it, and checks that it works.

---

## 5. Workflow C — performance, determinism, and build hygiene

| Goal | Macro | Note |
|---|---|---|
| Keep a hot function fast | `KER0_NO_VIRTUALIZE(fn)` | Excluded from virtualization; fastest option. |
| Speed up a protected region | `KER0_JIT(fn)` | First-level bytecode optimization. |
| Max speed for a critical inner loop | `KER0_JIT_MAX(fn)` | Use sparingly; verify output is identical. |
| Avoid a standalone copy | `KER0_INLINE(fn)` | Inlined at call sites. Watch for recursion → infinite expansion. |
| Remove hooking surface | `KER0_NO_UPVALUES(fn)` | The body must not reference enclosing locals — it compiles with no capture. |
| Same key every build | `KER0_ENC_FUNC_SEED(fn, seed)` | Deterministic compile-time seed. |
| Fresh key every build | `KER0_GET_RNG_SEED()` | New random seed per protection run. |
| Traceable errors | `KER0_BEGIN_CLIENT_LINES()` + `KER0_LINE` | Client-visible line numbers; self-reported source lines. |

**Measuring the cost.** Compare the same loop before and after protection:

```lua
local t0 = os.clock()
for i = 1, 100000 do acc = acc + i end
print("elapsed:", os.clock() - t0)
```

Run it once unprotected and once protected. If the protected number is an
order of magnitude worse, the hot function needs `KER0_NO_VIRTUALIZE` or
`KER0_JIT` — measure before and after each change, never assume.

---

## 6. Pitfalls catalogue

| Pitfall | Symptom | Prevention |
|---|---|---|
| `KER0_ASSUME` on a function | `attempt to call a number value` | `KER0_ASSUME` yields a number; never wrap a callable. |
| `KER0_ENC_NUM` used as a callee | `attempt to call a number value` | Only numbers as data. |
| `KER0_ASSUME` on a runtime-varying value | Wrong branch, weird behavior | Only constant-ish values; remember "undefined" if it differs at runtime. |
| Encrypted string written but read as a literal | `nil` value, then `attempt to call a nil value` | Decrypt into one local and reuse it. |
| `KER0_CRASH` on a reachable path | Script dies for legit users | Only in anti-tamper branches that a normal user cannot reach. |
| `KER0_SECURE_CALLBACK` on a callback external code calls | External call refused / runtime error | Use it only for callbacks invoked exclusively by protected code. |
| `KER0_NO_UPVALUES` on a body using outer locals | `nil` inside the function | Rewrite the body to take parameters, or drop the macro. |
| `KER0_INLINE` on a recursive function | Build hang or stack explosion | Use `KER0_NO_VIRTUALIZE` for recursion. |
| Macros in a non-protected build | `attempt to call a nil value` | `KER0_OBFUSCATED` guard + separate dev file. |
| `KER0_ENC_FUNC` on a function whose behavior must match across builds | Builds differ | Use `KER0_ENC_FUNC_SEED` for a fixed seed. |
| Stacking protection layers | Confusing call semantics | One layer per function unless documented. |
| `KER0_OPAQUE` used as a business condition | Logic becomes unreachable-looking and untestable | Keep `KER0_OPAQUE` for security checks only; real conditions stay plain. |
| Macros used with no stub in an unprotected run | `attempt to call a nil value` on the first protected line | Install stubs (§7) or use a separate dev copy of the script. |
| Local-shadow stub (`local K = K or fn`) left in the protected file | Protection silently disappears; secrets ship in plaintext | Never stub inside the protected file — use a dev header (§7.4 Option A) and verify the protected output (§8.3). |
| `local KER0_LINE = KER0_LINE or 0` | Every `KER0_LINE` reports `0` | Same rule: stubs never live in the protected file. |
| `KER0_CRASH` stubbed to a no-op in a shipped build | Anti-tamper is a no-op | Stubs are dev-only; the shipped build keeps the real macro. |
| **Stub returns a different value than the macro** | Works in dev, fails in prod | The stub must mirror the macro's semantics (§1 invariant 10). `KER0_ASSUME` is the usual offender. |
| **Stub block missing a macro** | `attempt to call a nil value` in the dev build only | Use the single canonical stub block from §7.2 — do not hand-roll a partial one. |
| **`KER0_ENC_FUNC_SEED` without a fixed seed** | Value changes every build | Use the `_SEED` variant when the value must be reproducible. |
| **Macro used across `require()` boundaries** | Macro missing in the other file | Macros are per-chunk — see §9. |

---

## 7. Macro stubs (unprotected / dev runs)

A script that uses Ker0 macros, executed **without** the Ker0 compiler, sees
every macro identifier as `nil`:

```
attempt to call a nil value
```

`KER0_NO_VIRTUALIZE(fn)` is then a call to `nil` and the script dies on the
very first protected line — long before any real logic runs. The fix is a
**stub**: a no-op replacement with the same signature that keeps the script
runnable in a plain editor or plain executor.

### 7.1 The stub pattern

```lua
local KER0_NO_VIRTUALIZE = KER0_NO_VIRTUALIZE or function(fn) return fn end
```

Read it as "use the real macro if the protector provided one, otherwise fall
back to an identity function". The right-hand side resolves to the *global*
(a local is not in scope inside its own initializer), so:

* protected build → the real macro wins;
* unprotected build → the global is `nil` → the identity fallback is used.

### 7.2 The canonical stub block — use this one, do not hand-roll

This block covers **every** macro in the catalog. Copy it as a whole; a
partial stub is the #1 cause of "the dev build breaks on a macro the prod
build handles".

```lua
-- --- function-wrapper macros: identity is a faithful stub ---
local KER0_NO_VIRTUALIZE   = KER0_NO_VIRTUALIZE   or function(fn) return fn end
local KER0_JIT             = KER0_JIT             or function(fn) return fn end
local KER0_JIT_MAX         = KER0_JIT_MAX         or function(fn) return fn end
local KER0_INLINE          = KER0_INLINE          or function(fn) return fn end
local KER0_NO_UPVALUES     = KER0_NO_UPVALUES     or function(fn) return fn end
local KER0_ENC_FUNC        = KER0_ENC_FUNC        or function(fn) return fn end
local KER0_SECURE_CALL     = KER0_SECURE_CALL     or function(fn) return fn end
local KER0_SECURE_CALLBACK = KER0_SECURE_CALLBACK or function(fn) return fn end
local KER0_ENC_FUNC_SEED   = KER0_ENC_FUNC_SEED   or function(fn, _seed) return fn end

-- --- value macros: return the input unchanged ---
local KER0_ENC_STRING = KER0_ENC_STRING or function(s) return s end
local KER0_ENC_NUM    = KER0_ENC_NUM    or function(n) return n end
local KER0_OPAQUE     = KER0_OPAQUE     or function(b) return b end

-- KER0_ASSUME mirrors the protected semantics: the macro expands to
-- `expected`, NOT to expr. See the note in the catalog (section 2.2) and
-- the calibration harness (section 2.7) if the two disagree.
local KER0_ASSUME     = KER0_ASSUME     or function(_expr, expected) return expected end

-- --- predicates / flags ---
local KER0_IS_CALLER_KER0 = KER0_IS_CALLER_KER0 or function() return true end
local KER0_GET_RNG_SEED   = KER0_GET_RNG_SEED   or function() return 0 end
local KER0_LINE           = KER0_LINE           or 0
local KER0_OBFUSCATED     = KER0_OBFUSCATED     or false

-- --- structural / abort ---
local KER0_BEGIN_CLIENT_LINES = KER0_BEGIN_CLIENT_LINES or function() end
local KER0_CRASH = KER0_CRASH or function()
    error("KER0_CRASH reached in an unprotected build (dev stub)")
end
```

| Macro | Stub | Dev-build behavior |
|---|---|---|
| `KER0_ENC_STRING` | `function(s) return s end` | The secret stays plaintext — never ship the dev file. |
| `KER0_ENC_NUM` | `function(n) return n end` | Constant stays a plain number. |
| `KER0_ENC_FUNC` | `function(fn) return fn end` | Body stays readable — that is the point of a dev build. |
| `KER0_ENC_FUNC_SEED` | `function(fn, _seed) return fn end` | Same as `ENC_FUNC`; the seed is meaningless without the protector. |
| `KER0_ASSUME` | `function(_expr, expected) return expected end` | Mirrors the protected build: the macro yields `expected`. If your calibration run showed Ker0 snapshots `expr` instead, change this stub to `function(expr) return expr end`. |
| `KER0_OPAQUE` | `function(b) return b end` | The branch is decided normally. |
| `KER0_IS_CALLER_KER0` | `function() return true end` | Protection guards must not fire in dev. |
| `KER0_GET_RNG_SEED` | `function() return 0 end` | Deterministic dev seed. |
| `KER0_LINE` | `0` (a number, **not** a function) | It is a constant, not a call — a function stub breaks `"line " .. KER0_LINE`. |
| `KER0_OBFUSCATED` | `false` | Keeps the "must be obfuscated" guard loud. Use `true` only in a dedicated dev file. |
| `KER0_CRASH` | `error(...)` | Keeps anti-tamper meaningful in dev. Use `function() end` only when you explicitly want the script to survive for UI testing. |
| `KER0_BEGIN_CLIENT_LINES` | `function() end` | No-op — no line remapping in dev. |
| `KER0_SECURE_CALLBACK` | identity | In dev the callback can be called by anyone — expected. |
| `KER0_NO_UPVALUES` | identity | The body keeps its upvalues in dev — harmless. |

### 7.3 The shadowing trap (read before shipping a stub)

`local K = K or fallback` creates a **local** that shadows the macro name for
the rest of the chunk. Two consequences:

1. **In a protected build the protection can silently disappear.** If the
   protector does not publish the macro as a global, the right-hand side is
   `nil`, the local becomes the identity fallback, and from that line on
   every `KER0_NO_VIRTUALIZE(...)` in the file resolves to the *local* — no
   longer a macro. The function ships unprotected, with no error and no
   warning. (Some Ker0 versions instead *reject* the build because they see
   a macro identifier in a non-call position — a build error is the good
   outcome; silence is the bad one.)
2. **`local KER0_LINE = KER0_LINE or 0` freezes the line number to 0** for
   the whole file, so every later `KER0_LINE` reports `0`.

Therefore:

* **Never put the local-shadow stub in the file you protect.** Keep stubs in a
  dev-only header, or in a dev-only copy of the script.
* If you must ship one file, use the guarded **global** injection form (§7.4)
  and verify the protected output.

### 7.4 Three places to put stubs

**Option A — dev header file (recommended; the protected file stays clean).**

`ker0_dev_stub.lua` (dev only, never included in the protected build). Use the
canonical block from §7.2, but in **global** form so the macro call sites
below are still seen as macros by the protector:

```lua
-- ker0_dev_stub.lua — dev only, never included in the protected build.
-- Makes a Ker0 script runnable without the protector.
local function identity(fn) return fn end

KER0_NO_VIRTUALIZE   = KER0_NO_VIRTUALIZE   or identity
KER0_JIT             = KER0_JIT             or identity
KER0_JIT_MAX         = KER0_JIT_MAX         or identity
KER0_INLINE          = KER0_INLINE          or identity
KER0_NO_UPVALUES     = KER0_NO_UPVALUES     or identity
KER0_ENC_FUNC        = KER0_ENC_FUNC        or identity
KER0_SECURE_CALL     = KER0_SECURE_CALL     or identity
KER0_SECURE_CALLBACK = KER0_SECURE_CALLBACK or identity
KER0_ENC_FUNC_SEED   = KER0_ENC_FUNC_SEED   or function(fn, _seed) return fn end

KER0_ENC_STRING      = KER0_ENC_STRING      or function(s) return s end
KER0_ENC_NUM         = KER0_ENC_NUM         or function(n) return n end
KER0_OPAQUE          = KER0_OPAQUE          or function(b) return b end
KER0_ASSUME          = KER0_ASSUME          or function(_expr, expected) return expected end

KER0_IS_CALLER_KER0  = KER0_IS_CALLER_KER0  or function() return true end
KER0_GET_RNG_SEED    = KER0_GET_RNG_SEED    or function() return 0 end
KER0_LINE            = KER0_LINE            or 0
KER0_OBFUSCATED      = KER0_OBFUSCATED      or false

KER0_BEGIN_CLIENT_LINES = KER0_BEGIN_CLIENT_LINES or function() end
KER0_CRASH = KER0_CRASH or function()
    error("KER0_CRASH reached in an unprotected build (dev stub)")
end
```

Main script (one file, both modes):

```lua
if not KER0_OBFUSCATED then
    -- dev only; adjust the loader to your environment
    -- (load / loadstring / require — executors differ)
    load("ker0_dev_stub.lua")()
end
```

The stubs are installed as **globals**, so the macro call sites below are
still seen as macros by the protector, and in the protected build the branch
is never taken.

**Option B — guarded global injection inside the same file.**

```lua
if not KER0_OBFUSCATED then
    KER0_NO_VIRTUALIZE   = KER0_NO_VIRTUALIZE   or function(fn) return fn end
    KER0_ENC_STRING      = KER0_ENC_STRING      or function(s) return s end
    -- ... the rest of the stubs (use the global form from Option A)
end
```

Safe **iff** the protector folds `KER0_OBFUSCATED` to `true`, drops the
branch, and still expands the macro call sites. If it does not drop the
branch, the build either fails or the stub shadows the macro. Verify in the
protected output (§8.3).

**Option C — separate dev copy of the script** with the local-shadow stubs at
the top (the block from §7.2). Simplest and zero risk to the protected file,
but the two copies drift apart over time — keep them in version control and
diff them when the protected copy changes.

### 7.5 When to stub at all

Ask the user first. Stubbing is for:

* local development / debugging in a plain executor;
* keeping a shared "core" script runnable in both modes.

Do **not** stub in the shipped build: `KER0_OBFUSCATED = true`, a no-op
`KER0_CRASH`, or a `KER0_ENC_STRING` that returns plaintext defeat the
entire point of the protection.

---

## 8. Verification

Four levels. Do as many as the environment allows, and tell the user which
ones you actually performed.

### 8.0 The build step — the user protects the script

The protected artifact is produced **by the user**, at:

**<https://ker0-protect.pp.ua/>**

The site is behind a Cloudflare Turnstile human check, so you cannot submit
the script yourself — do not try, and do not claim you did. Your job ends at
"here is the fixed source, now protect it".

The handoff sequence the user follows:

1. Open <https://ker0-protect.pp.ua/> and pass the human check.
2. Paste the **fixed, unprotected** source.
3. Obfuscate / protect it (Ker0 expands the macros here).
4. Take the protected output and run it (inject into the executor, or load it
   where the script normally runs).
5. Check that it works — the original error is gone and nothing new appeared.

Always spell out steps 2–5 in your answer. A one-line "fixed" without the
build instruction is useless, because the user cannot reach a working script
without the build step.

### 8.1 Syntax-check the dev copy

The stubbed copy is the one that must parse:

```bash
luau-analyze script.lua      # or: luac -p script.lua   (Lua 5.1)
```

A malformed macro call or a broken stub shows up here, before Ker0.

### 8.2 Calibrate the macro semantics once

Run the harness from §2.7 in a protected build and record the output. It
settles `KER0_ASSUME` and reveals the `KER0_BEGIN_CLIENT_LINES` closer.

### 8.3 Audit the protected output

After protecting, check the protected artifact:

| Check | How | Fails when |
|---|---|---|
| No plaintext secrets | search the output for the API key string | `KER0_ENC_STRING` was stubbed, so the secret ships in the clear |
| No stub leaked | search for `function(fn) return fn end` | A local-shadow stub shipped in the protected file |
| Macro call sites gone | search for `KER0_ENC_`, `KER0_ASSUME`, `KER0_JIT` | The protector did not expand them, or a stub shadowed them |
| Guard intact | the output still hard-fails when unprotected | The `KER0_OBFUSCATED` guard was optimized away |
| Behavior unchanged | run the same input through dev and protected builds | A macro changed semantics |

If a secret appears in plaintext in the protected output, the stub shadowed
the macro → drop the local stub and use Option A or B.

### 8.4 Check the performance delta

See §5 — measure the same loop before and after protection, before and after
adding `KER0_NO_VIRTUALIZE` / `KER0_JIT`.

---

## 9. Multi-file scripts and `require()`

Ker0 macros are **per-chunk**. A macro used in file A does not exist in
file B.

| Situation | Result | What to do |
|---|---|---|
| Protected file `require`s an unprotected file | The unprotected file has no macros — its strings and functions stay plaintext | Protect every file separately, or bundle into one file before protecting |
| Protected file `require`s a protected file | Each file is protected independently; seeds and keys are per-file | Use the same protection settings across files, or `KER0_ENC_FUNC_SEED` when a value must match in both |
| A value must be identical in two files | `KER0_GET_RNG_SEED()` differs per build *and* per file | Use `KER0_ENC_FUNC_SEED` / a fixed constant instead |
| `load()` / `loadstring()` of a runtime string | The string is not a chunk Ker0 processed — no macros inside it | Inline the code, or protect the source string separately |
| `KER0_OBFUSCATED` in a required file | The flag is per-chunk too | Put the guard in every protected file |

---

## 10. Known unknowns

Do not guess these into user code. Resolve them with the calibration harness
(§2.7) or the current Ker0 documentation.

| Item | Status | How to resolve |
|---|---|---|
| Exact `KER0_ASSUME(expr, expected)` semantics | **unverified** | §2.7 harness — `42` vs `2` settles it |
| `KER0_BEGIN_CLIENT_LINES()` block closer | **unverified** | Protect a file with the opener; the error message usually names the closer |
| Does `KER0_ENC_STRING` decrypt at macro site or at use? | **inferred** (macro site) | Harness: encrypt, then check the value is correct in the same expression |
| Exact restriction semantics of `KER0_SECURE_CALL` | **inferred** | Harness: call a `SECURE_CALL`ed function from unprotected code and observe |
| Whether `KER0_INLINE` keeps the original binding usable | **inferred** (yes) | Harness: call the inlined function normally |
| How Ker0 handles a macro identifier in a non-call position | **unverified** | Try `local KER0_JIT = ...` in a throwaway file and see if the build errors |

---

## 11. Security reality check — read this before promising protection

Ker0 makes a script **harder to read**. It does not make a client-side secret
safe.

* A secret that reaches the client (an API key, a license token, a webhook
  URL) is **extractable at runtime** by anyone who runs the script — the
  obfuscated form is only a speed bump for them. Encryption delays the
  reading; it does not prevent it.
* The correct architecture for a real secret is **server-side**: the client
  asks your server for what it needs, the server authenticates the user, and
  the secret never ships.
* Client-side Ker0 protection is genuinely useful for **raising the cost of
  casual theft and re-distribution**, for hiding business logic from casual
  readers, and for anti-tamper tripwires — not for protecting credentials.
* Anti-tamper built on `KER0_CRASH` / `KER0_OPAQUE` is a deterrent too. A
  determined attacker with a debugger can strip it. Never let anti-tamper be
  the only thing standing between a user and the paid functionality.

When a user asks to "protect my API key", do both: wrap it with
`KER0_ENC_STRING` **and** tell them to move the key behind a server endpoint.
State the tradeoff plainly — a skill that implies client-side encryption
solves credential theft is worse than no skill.

---

## 12. Response contract

**Answer in the user's language** (§0.3) — Russian if the user writes Russian,
English if they write English or the language is unclear. Code, macro names,
error strings and stack traces stay verbatim in any language.

When you answer, structure it like this (adapt length to the request):

1. **What the user reported** — the error text / the protection goal, restated
   in one line.
2. **Root cause** — the concrete reason, tied to the stack frames and to the
   exact source construct (file:line or snippet). Name the macro involved.
3. **The fix** — the patched code (or a precise diff) with the Ker0 macros
   added and why each one was chosen. Keep the user's logic intact.
4. **Stubs, if the user wants a dev-runnable build** — say which placement
   you used (dev header / guarded global injection / dev copy) and warn that
   local-shadow stubs must never ship in the protected file (§7.3).
5. **How to verify** — re-protect, re-run, and what to look for (§8). If you
   could not run Ker0 yourself, say so explicitly instead of claiming the fix
   works.
6. **Next steps / risks** — e.g. "enable `KER0_BEGIN_CLIENT_LINES()` so future
   stack traces point at real source lines", "this script now requires a
   protected build", "the API key should really live on a server" (§11).
7. **The handoff line** — close with the build instruction, in the user's
   language, using one of the two exact forms below.

### 12.1 The handoff line (use verbatim, in the user's language)

**English:**

> Fixed. Now go to https://ker0-protect.pp.ua/, paste the script, obfuscate it,
> then inject it into your executor and check that it works.

**Russian:**

> Починил. Теперь зайди на https://ker0-protect.pp.ua/, вставь скрипт,
> обфусцируй, затем заинжекти в экзекутор и проверь, что всё работает.

Rules for this line:

* Always include the URL. It is the only way the user can produce a protected
  build.
* Do not claim you protected it yourself — the Turnstile check is manual.
* If the request was diagnostic only (no code change), still end with the
  instruction to re-protect and re-check, since the fix must be validated in
  a protected build.
* If the user uses a different build pipeline than the web UI, keep their
  pipeline but still name the URL as the default.

Never invent a macro that is not in §2. Never change behavior beyond what the
user asked for. When uncertain, state the assumption and the check that would
confirm it.

---

## 13. Quick patterns (copy-paste starting points)

**Protected header + line mapping**

```lua
if not KER0_OBFUSCATED then
    error("Script must be obfuscated!")
end
KER0_BEGIN_CLIENT_LINES()
```

**License check as a protected function**

```lua
local validateLicense = KER0_ENC_FUNC(function(key)
    local EXPECTED = KER0_ENC_NUM(0x5A17BEEF)
    if key == nil then return false, "missing key" end
    return key == "USER-SUPPLIED-KEY", "invalid key"
end)
```

**Opaque anti-tamper branch**

```lua
if KER0_OPAQUE(true) then
    if not KER0_IS_CALLER_KER0() then
        KER0_CRASH()
    end
end
```

**Protected callback handed to the host**

```lua
table.insert(handlers, KER0_SECURE_CALLBACK(function(payload)
    return validate(payload)
end))
```

**Hot path kept fast**

```lua
local fastAdd = KER0_NO_VIRTUALIZE(function(a, b)
    return a + b
end)
```

**Per-build key vs stable key**

```lua
local buildKey   = KER0_GET_RNG_SEED()                       -- changes every build
local stableKey  = KER0_ENC_FUNC_SEED(function() return 42 end, 0xC0FFEE) -- same every build
```

**Self-reporting source line inside an error**

```lua
local function checkLicense(key)
    if key == nil then
        error("license check failed at source line " .. KER0_LINE)
    end
end
```
