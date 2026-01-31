# Kits Kärneriks: Breaking the Glass on Client-Side Fraud Defense

## 1. The Objective

This repository chronicles a dual-pronged assault on Google’s login infrastructure. The primary target is "Botguard," a defensive mechanism often dismissed as a simple fingerprinting script. It is not. It is a hostile, obfuscated Virtual Machine (VM) running directly in the client’s browser.

In 2021, we proved this system could be bypassed. We did not do this by untangling the knot; we cut it. By utilizing a hardened headless browser, we generated valid security tokens on the legitimate domain and trafficked them to a Man-in-the-Middle (MITM) phishing context. This document details the architectural flaws of that client-side trust model and integrates a deep-dive reverse engineering analysis of the VM internals that make Botguard such a formidable adversary.

> 🚨 **Operational Note** 🚨
>
> The architectural analysis here applies to the core Botguard VM logic found in ReCaptcha v2 and Google's v2/v3 login flows. While specific variable names shift with every compile, the underlying "CPU-in-JavaScript" logic remains the engine of their defense.

## 2. The Defense: It's Not a Script, It's a CPU

To defeat the enemy, you must understand its biology. Botguard is not a static list of checks. It is a custom, register-based Virtual Machine written in JavaScript.

When the browser loads the login page, it doesn't just run code; it boots a processor. This VM executes a custom bytecode binary. This is the heavy armor. It beats standard Control Flow Flattening (CFF) because you cannot simply beautify the code to read it. You must write a disassembler, a decompiler, and a custom debugger just to see the instructions.

### The Virtual Machine Architecture

The VM emulates a modern CPU. It uses registers to hold variables and operational states. While older obfuscators might use stack-based logic (like Java or WASM), Botguard uses a register-based approach that mimics the x86 architecture sitting in your actual hardware.

The initialization sequence is frantic. It locates its own bytecode string, grabs a substring (often the first three characters + an underscore), and uses that dynamic key to locate the initialization function. This prevents static analysis tools from easily finding the entry point. Once the VM boots, it begins executing opcodes.

### The Opcodes

We identified the instruction set through painful trial and error. The VM utilizes a mix of standard logic and custom operations:

* **`328 (USHR) / 381 (ADD)`:** Standard bitwise and arithmetic operations.
* **`65 (SETPROP) / 467 (GETPROP)`:** Manipulation of object properties, used to bind the anti-tamper mechanisms to the DOM.
* **`220 (IN)`:** Checks for the existence of properties, likely looking for `webdriver` flags.
* **`289 (HALT)`:** Kills the execution if an anomaly is detected.

The most dangerous aspect is **Self-Modifying Code**. The VM brings in new opcodes at runtime. It constructs an array of integers (Register 274) and maps them to string definitions, using a `LOADSTRING` opcode to generate new instructions on the fly. An `EVAL` opcode (mapped as `LOADOP`) then compiles these into executable logic. The code you see at the start is not the code that runs at the end.

## 3. Anti-Tamper Mechanisms

Botguard actively fights back against analysis. It assumes it is being watched.

### Chronometric Defense (Anti-Debug)

The script is obsessed with time. It constantly polls `performance.now()` combined with `Date.now()`. If you set a breakpoint in DevTools, the execution pauses, but the clock keeps ticking. When you resume, the delta between timestamps is massive.

The VM uses this delta to mutate a **Seed** (stored in the VM context `K.U`). The seed determines the decryption key for the next block of bytecode. If the time delta suggests a debugger was active, the seed corrupts. The program doesn't crash immediately; it silently diverges into a garbage execution path, generating an invalid token that the server will reject. You think you are debugging it, but you are ghost-hunting in a dead timeline.

### The Anti-Logger

You cannot print the variables. The script aggressively binds to console methods. It creates a trap function that overrides properties using a `create` (or similar) method. If you attempt to inject a `console.log` or use a conditional logpoint, the trap triggers.

This trap modifies the `t.prototype` stack, which is linked to the memory reader function. By logging a variable, you inadvertently shift the memory pointer. The VM reads the wrong bytes, the instruction stream corrupts, and the session dies.

## 4. The Bypass: Environmental Spoofing

Given the complexity of the VM—self-modifying opcodes, rolling encryption keys, and time-based seed mutation—pure reverse engineering is a high-cost attrition war. The most efficient attack vector is not to break the lock but to steal the key.

### The "Puppet" Strategy

The server-side validation checks the token's integrity, not its origin. The token proves that *a* browser passed the checks. It does not prove *which* user is holding the token.

We deployed **go-rod**, a Golang-based browser automation framework, to act as the token factory.

1. **Stealth Layer:** Standard headless Chrome leaks its identity (e.g., `navigator.webdriver = true`). We patched this using stealth libraries to mask the user agent, override prototype methods, and spoof WebGL rendering contexts.
2. **Token Extraction:** The automated browser navigates to `accounts.google.com`. It inputs a target email. Botguard runs, validates the environment, and mints the `bgRequest` token.
3. **Interception:** We hook the request on the client side immediately after generation. We abort the network call to Google (preventing the token from being "spent") and extract the string.
4. **Injection:** This fresh, valid token is passed to our MITM proxy. The proxy attaches it to the victim's request. Google's server sees a valid token from a "clean" browser and authorizes the phishing session.

This method bypasses the VM entirely. We do not care about the opcodes or the registers because we are giving the VM exactly what it wants: a compliant browser environment.

## 5. The Reality Check

* **Client-Side Checks are Vulnerable to Environmental Spoofing:** While powerful, client-side defenses are fundamentally running on an attacker-controlled machine. With sufficient effort, the environment can be mimicked to satisfy the checks. The use of stealth plugins is a clear example of this.
* **Token Portability is a Potential Weakness:** The security of token-based systems like Botguard relies on the token being non-transferable. This research shows that if token generation can be outsourced to a "clean" environment, the token can then be used in a "dirty" one, defeating the purpose of the check.
* **The Importance of a Layered Defense:** This bypass works because it circumvents a single, albeit strong, layer of defense. More advanced defensive systems could correlate the token's fingerprint with other server-side signals (e.g., IP address reputation, historical session data) to detect this type of anomaly.

The industry creates these elaborate cat-and-mouse games, adding layers of obfuscation and behavioral analysis. Yet, as long as the logic relies on a token that can be ported from a clean environment to a dirty one, the defense is bypassable. It isn't about breaking the encryption; it is about abusing the architecture.

<table>
  <tr>
    <td><img width="1024" height="512" alt="botguard_white" src="https://github.com/user-attachments/assets/b2304d98-c1a0-4d9b-8c10-f62aceb2d1fe" /></td>
    <td><img width="1024" height="512" alt="botguard_dark" src="https://github.com/user-attachments/assets/c8515399-ef0f-4260-9e4b-998a8ff31104" /></td>
  </tr>
</table>

## 6. Technical Findings: The Memory Reader

For those intent on the pure reverse-engineering path, the heart of the beast is the memory reader function, often minified as `H`.

It reads bytes from the bytecode array but encrypts them before return.

* **Register 21:** Acts as a rolling key array.
* **`Z.W`:** A position tracker that increments linearly.
* **`Z.U`:** The seed, which mutates based on time and execution history.

The function `H(true, L, 8)` reads 8 bits. If the first argument is true, it calls an encryption routine using Register 21. A `SETPROP` opcode exists specifically to scramble these keys, resetting the position and pulling a new seed from the reader. This circular dependency (the reader relies on the seed, the seed relies on the reader) makes static analysis nearly impossible without perfect emulation.

## 7. Contact

I break things to understand how to build them better. I am looking for roles where I can apply this offensive mindset to defensive strategies. If you want to discuss web security, reverse engineering, or why client-side trust is a myth, reach out. Find me on LinkedIn.
