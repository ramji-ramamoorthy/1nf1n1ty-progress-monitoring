# pwn.college: Reverse Engineering -- Real World Applications
 

 
---
 
## Overview
 
This is the last lecture in the pwn.college reversing module. It covers real world applications of reverse engineering through three main case studies: license key checkers, DRM evolution, and game modding. The core theme throughout is that trusting a binary to keep secrets safe is a fundamentally flawed assumption.
 
---
 
## 1. License Checkers
 
### The Problem
 
Back before the internet, software developers had no way to verify purchases server-side. To combat piracy they came up with several approaches including uncopyable floppies and physically modified media. One of the most common methods was the **license key check**.
 
How it worked:
 
- Developer implements a secret algorithm inside the binary
- The algorithm verifies a long serial number and returns yes or no
- Developer has a way to generate many valid keys
- Pirates should not be able to generate valid keys
- Software shipped with a sticker on the CD containing the key
- Only purchasers with valid keys could install the software
### The Fundamental Flaw
 
The key insight here is that this approach and many other development practices at the time **trusted the binary itself to keep algorithms and data safe**. That is not a safe assumption. A reverse engineer can open that binary and extract the secrets inside.
 
---
 
## 2. The Rise of Keygens
 
### What is a Keygen
 
A keygen (key generator) is the product of reverse engineering a license key verification algorithm well enough to understand it and reproduce it. Once you understand the algorithm you can generate valid keys for software you did not purchase.
 
### Keygens as Culture
 
Keygens were not just piracy tools. In the 90s and early 2000s they became a form of art and expression similar to the demoscene. Groups would ship keygens with:
 
- Custom UI libraries and visual design
- Embedded soundtracks (example: a Nero Burning ROM keygen by a group called Orion shipped with a full music track for no functional reason)
- Group logos and branding
People took genuine pride in the craft. The reversing skill required was real and the community treated it seriously.
 
### Crackmes
 
The exercises in the pwn.college reversing module are inspired by this. A **crackme** is a small program designed specifically to test reversing skills, modelled after real license key verification systems. The workflow is:
 
1. Get the executable
2. Disassemble it (and optionally decompile it)
3. Analyze the verification algorithm
4. Figure out what makes a valid input
---
 
## 3. Alternatives to Keygens -- Patching
 
As internet access became widespread, server-side license checks became possible. This killed keygens because you cannot generate a valid key if the verification only ever happens on a remote server. Online games became largely piracy-resistant because of this.
 
The response from attackers shifted to **patching**:
 
- Reverse engineer the binary until you fully understand where the license check lives
- Replace that section of code with NOPs (no-operations) to disable it entirely
- Requires a thorough understanding of the control flow
---
 
## 4. Evolution of DRM
 
Developers responded to patching with increasingly complex **Digital Rights Management (DRM)** techniques:
 
### Code Obfuscation
Rewriting binary code so it is functionally identical but impossible to read or understand. Has a performance cost but makes static analysis extremely difficult. Similar in principle to the obfuscated shellcode from earlier in the course, but applied to the entire program.
 
### Anti-Debugging Techniques
Techniques that detect whether a debugger or emulator is attached and alter or stop execution accordingly. Designed to fight dynamic analysis.
 
### Virtual Machine Based Protection
A significant evolution in DRM:
 
- DRM developers create a custom CPU architecture that only they know
- They write an emulator for that architecture
- The emulator itself is heavily obfuscated and protected with anti-debugging
- The critical license logic runs inside that custom VM
- An attacker now has to reverse engineer the emulator AND the custom ISA before they can even read the protected code
This is directly relevant to some of the practice problems in this module.
 
### Trusted Execution Environments (TEEs)
The most recent trend: move DRM outside the CPU entirely into TEEs (like ARM TrustZone, present on most phones). The protected logic runs in hardware-isolated environments that the main OS cannot inspect. Not a perfect solution but significantly raises the bar.
 
---
 
## 5. Reverse Engineering for Game Modding
 
Game modding is another major real world application of reverse engineering, especially when modders hit the limits of what the developer officially exposed.
 
### SKSE -- Skyrim Script Extender
 
- A binary modification tool that loads alongside Skyrim
- Patches parts of the Skyrim binary at runtime to enable additional functionality for mods
- Many major Skyrim mods depend on it
- After every Skyrim update, the SKSE team has to re-reverse-engineer the new binary to find updated memory offsets and reposition their hooks correctly
- The official SKSE documentation tells users not to update Skyrim until SKSE has confirmed compatibility for exactly this reason
### DF Hack (Dwarf Fortress)
 
- Same concept applied to Dwarf Fortress
- Binary-level modding tool that hooks into the game executable
- Needs to be updated every time a new DF version ships because memory layouts change
### The Dark Side -- Cheat Engines and Aimbots
 
The same techniques used for legitimate modding are used for cheating in online games:
 
- An aimbot is essentially a game mod that runs alongside the game
- Uses a routine in a cheat engine to read game memory and auto-aim at opponents
- Anti-cheat software is the defensive side of this, also using reverse engineering
- This is an active, high-stakes cat and mouse war being fought on the reverse engineering and code modding battlefield right now
---
