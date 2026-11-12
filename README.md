

## Summary

On 2026-11-12, a bot-delivered HTML widget launched Windows Calculator from the official signed Telegram Desktop 7.4.2 x64 process. Telegram remained alive and responsive. The proof uses the QuickJS `await using` use-after-return fixed by [`ca3594d`](https://github.com/TelegramMessenger/quickjs/commit/ca3594dae5fd0dd6d667be74a322e8e935a2d217), not the earlier RegExp bytecode UAF.

Desktop 7.4.2 embeds widget-sandbox `a74626d6768b5d14e0dc0b8435a7df44a980a8e7` (version 0.6.3) and QuickJS `8f0d3b6193a01bda29d69b0cab4f03320bcaa027`, which does not contain the fix. Widget-sandbox v0.6.4 incorporates the corrected engine.

## Root cause and chain

QuickJS accepts `for (await using ...)` inside a non-async class static block. Disposal emits `OP_await`; the normal interpreter frame exits through `done_generator` without `close_var_refs()`, leaving captured `JSVarRef` pointers into a returned C stack frame.

The PoC turns deterministic frame reuse into a forged string-backed `JSVarRef`, corrupts a string length to obtain a heap leak and address-selected `JSValue` access, and corrupts a `Uint8Array` for byte-precise arbitrary read/write. A native function pointer reveals the ASLR image base. After checking the exact Telegram image, a forged `JS_CLASS_C_CLOSURE` record calls Telegram's resolved `WinExec` import with `calc.exe`; the legitimate finalizer then frees the genuine donor allocation and returns cleanly.

## Confirmed target

- Telegram Desktop: `7.4.2.0`, official signed x64 build
- `Telegram.exe` SHA-256: `3864695188C5D564C7A6036B04ADB4E67D589F7843C51891155EA4905EB41963`
- Result: `CalculatorApp.exe` launched; Telegram remained responsive
- PoC SHA-256: `6B385FBF009B6923727B84448D3C460351733A8EEA953479279E30EBFF4FDF90`

## Reproduction

Use only in an isolated, authorized test environment. Send the exact contents of [`poc.html`](poc.html) as the raw `html` string of a Bot API `sendWidget` request to a private test chat, then display the resulting widget in the affected Desktop client. The payload executes when rendered and is restricted to launching Calculator on the exact tested image. No token, chat identifier, account data, or other secret is included.

## Remediation

Ship widget-sandbox v0.6.4 or a QuickJS revision containing `ca3594d` on every native client, rebuild the embedded engine, and retain a regression test that rejects suspending opcodes in normal frames.
