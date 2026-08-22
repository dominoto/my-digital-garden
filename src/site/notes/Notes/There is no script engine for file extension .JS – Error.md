---
{"dg-publish":true,"dg-path":"Resources/There is no script engine for file extension .JS – Error.md","permalink":"/resources/there-is-no-script-engine-for-file-extension-js-error/","title":"Fixing \"There is no script engine for file extension .js\" on Windows","created":"2024-02-13","updated":"2026-08-22","dg-note-properties":{"type":["[[Notes/Article]]"],"topics":["[[Windows]]"],"title":"Fixing \"There is no script engine for file extension .js\" on Windows","created":"2024-02-13","modified":"2026-08-22","author":["Domagoj Zubac"]}}
---


When you try to run a `.js` file on Windows, you may hit one of these errors:

> Can't find script engine "JScript" for script "filename.js".

> There is no script engine for file extension ".js".

## What's actually happening

Windows treats `.js` files as **JScript** scripts, to be executed by Windows Script Host (WSH). For that to work, two registry pieces must be intact: the JScript engine itself must be registered, and the `.js` extension must be associated with it. The error means one of those is missing or broken, commonly after installing a code editor or dev tool that claimed the `.js` association for itself, after a registry "cleaner" pass, or because an admin or antivirus disabled it deliberately (WSH is a classic malware vector).

> [!note] If your script is actually Node.js
> Double-clicking a `.js` file never runs it with Node. Windows hands it to WSH, which can't understand modern JavaScript. If that's your situation, this error is a red herring: run the file with `node filename.js` from a terminal instead. The fixes below are only for scripts genuinely written for Windows Script Host.

## Fix 1: Re-register the JScript engine

Open an elevated Command Prompt (right-click Start → *Terminal (Admin)*) and run:

```
regsvr32 %systemroot%\system32\jscript.dll
```

You should get a "succeeded" dialog. Try the script again. For many people this alone solves it.

## Fix 2: Restore the .js file association

If the error persists, the extension-to-engine mapping is broken. Save the following as `jscript_fix.reg`, then double-click it and confirm:

```reg
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\.js]
@="JSFile"
"Content Type"="text/javascript"

[HKEY_CLASSES_ROOT\JSFile]
@="JScript Script File"

[HKEY_CLASSES_ROOT\JSFile\Shell\Open\Command]
@="\"%SystemRoot%\\System32\\WScript.exe\" \"%1\" %*"

[HKEY_CLASSES_ROOT\JSFile\Shell\Open2\Command]
@="\"%SystemRoot%\\System32\\CScript.exe\" \"%1\" %*"
```

This restores the default mapping: `.js` → `JSFile` → executed by `WScript.exe` (windowed) or `CScript.exe` (console).

## Fix 3: Clear a per-user override

A leftover per-user association (set when some app registered itself for `.js`) overrides the system-wide mapping and survives Fix 2. Delete it: open `regedit`, navigate to

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\FileExts\.js
```

and delete the `UserChoice` subkey if present, then sign out and back in (or restart Explorer).

## Verify

Create a file `test.js` containing:

```js
WScript.Echo("JScript works!");
```

Double-click it. A message box means WSH is healthy again. Or run `cscript test.js` in a terminal for console output.

If this guide saved you some time, you can say thanks with a coffee:

<a href="https://ko-fi.com/dominoto"><img src="https://storage.ko-fi.com/cdn/kofi2.png?v=3" width="200" alt="Buy me a coffee at ko-fi.com"></a>

---
Source: paraphrased from [Error "There is no script engine for file extension" when running .JS files](https://www.winhelponline.com/blog/error-there-is-no-script-engine-for-file-extension-when-running-js-files/) at WinHelpOnline.
