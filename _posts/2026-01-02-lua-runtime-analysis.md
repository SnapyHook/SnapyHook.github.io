---
layout: post
title: "I Found a Lua VM Hidden Inside a UE4 Game. Here's What Happened."
date: 2026-01-02
categories: [reverse-engineering, game-development, lua, ue4]
tags: [slua-unreal, lua, unreal-engine, binary-analysis, frida, ida-pro, m2-mac]
author: Aryan Prashar
---

## It Started With an SDK Dump

I wasn't looking for Lua. Honestly, I wasn't even thinking about Lua.

I was just doing what I usually do—dumping the SDK from a UE4 game to understand how things work under the hood. You know, the usual ritual: point the SDK dumper at the binary, wait for it to churn through the symbols, grab a coffee, and then start poking around the output files.

But then something caught my eye. While I was scrolling through the class definitions, I kept seeing the same thing over and over: **Lua**. Lua references. Lua structures. Lua bridges.

And I thought: *That's weird. Why is Lua here?*

Unreal Engine doesn't ship with Lua. It's not a core feature. So someone deliberately embedded this. The question was: why?

---

## The Breadcrumbs Start Appearing

I started digging through the SDK output, and the more I looked, the more it became clear this wasn't some throwaway scripting system. This was *real*.

The first thing that grabbed me was this:

```cpp
class ULuaEventBridge : public UObject
{
public:
    TWeakObjectPtr<class ULuaStateWrapper> LuaStateWrapper;
    TMap<FString, FString> RegisterEventMap;
    TMap<FString, FString> LuaRegisterEventMap;
    TMap<uint32_t, FString> FilterKeyRegisterMap;
    TArray<FString> CurrentParamArray;
    TArray<FString> Params;
};
```

Look at that. This isn't just a wrapper. This is a two-way bridge. C++ events get mapped to Lua callbacks. Lua can trigger C++ events. Parameters pass back and forth. It's *integrated*. It's real.

I kept reading and found the game instance binding:

```cpp
void UUAEGameInstance::SetLuaStateWrapper(class ULuaStateWrapper* TLuaStateWrapper)
```

This function runs as part of the engine initialization. So Lua doesn't get loaded as a side module. Lua gets initialized *at the same time as the game world itself*. It's there from the start and never leaves.

At this point, I stopped thinking "interesting curiosity" and started thinking "this is how the game actually works."

---

## I Had to See Inside

Reading the SDK headers told me *what* existed, but not *how* it worked. I needed to see the actual machine code.

I grabbed my tools:

- **macOS M2** (Apple Silicon) because that's what I had
- **MuMu Player Pro** emulator running the game on Android
- **IDA Pro** for static analysis
- **Frida** for runtime poking
- **xhook, A64InlineHook, MSHookFunction** for intercepting function calls
- **Android NDK** and a Makefile because I needed to compile my hooks

And I fired up IDA Pro and pointed it at `libUE4.so`.

---

## What I Found in the Binary

I started following string references. You know how it goes—search for "lua_load" or "lua_" and see where it shows up. And it showed up *everywhere*.

I found function calls that matched:

```cpp
lua_loadfile()      // Load from disk
luaL_loadbufferx()  // Load from memory
lua_load()          // The actual bytecode parser
lua_pcall()         // Run a Lua function
```

That's the whole pipeline. That's the real Lua VM, not some stripped-down toy. It can parse bytecode, manage state, execute functions, handle errors. All of it.

I traced the call graph in IDA, looked at the stack frames, watched how data flowed from one function to another. The pattern was unmistakable. Someone had taken the real Lua 5.1 VM and linked it directly into the game engine.

---

## The Center of Everything: UGameLuaEnv

I went back to the SDK and found the orchestrator:

```cpp
class UGameLuaEnv : public UWorldSubsystem
{
public:
    FString GameScriptPath;
    FString LuaFileEntryFile;
    FString PreloadLuaFile;
    double StepGCTimeLimit;
    int StepGCLimitCount;
    double LuaGCInterval;
    
    class ULuaEventBridge* LuaEventBridgeInstace;
    class ULuaStateWrapper* LuaStateWrapper;
    class ULuaTriggerManager* LuaTriggerMgr;
    class UGameLuaAPI* GameLuaAPI;
    
public:
    void InitLuaFile();
    void InitLuaGlobalTools();
    void InitLuaGlobalVariable();
    void CheckCreateSluaState();
    void LuaDoString(const FString& LuaString);
    void CallLuaGlobalScriptFunction(const FString& InFunctionName);
    void CallLuaWaitGlobalScriptFunction(const FString& InFunctionName);
};
```

This is the heartbeat of the Lua system. When the world starts up, this initializes. It loads the Lua files. It sets up the global variables. It connects the event system. It's *always running*.

Look at those methods:

- `InitLuaFile()` – load the scripts
- `InitLuaGlobalTools()` – expose C++ functions to Lua
- `LuaDoString()` – execute Lua code on the fly
- `CallLuaGlobalScriptFunction()` – call a Lua function from C++

This isn't optional. This isn't a debug feature. The game *needs* Lua to work.

---

## Watching It Happen in Real Time

Static analysis tells you the structure. But I wanted to see it *actually happen*. So I fired up Frida and started hooking.

I placed a hook on `lua_load()`. Every time the game tried to load bytecode, my hook would fire. I could see:

- What the bytecode looked like (raw bytes)
- How big it was
- What Lua state it was going into
- When the parser finished

Then I hooked `lua_pcall()`. Every time the game called a Lua function:

- Which function was being called
- What arguments went in
- What came back out
- How long it took

And I started to see patterns. Scripts weren't being loaded all at once. They were loaded on-demand as the game needed them. When you triggered an event, the corresponding Lua code would load. When you ran gameplay logic, the Lua for that system would activate.

It was like watching the nervous system of the game light up.

---

## Inside the Machine: The Actual VM Code

I went deeper into IDA and found the real guts:

```cpp
__int64 __fastcall sub_ACA4A08(
    __int64 a1,          // Lua state
    int a2,              // function index
    unsigned int a3,     // arg count
    unsigned int a4,     // return count
    __int64 a5,          // error handler
    __int64 a6
)
{
    __int64 v12 = *(_QWORD *)(a1 + 16) + (-16 - 16LL * a2);
    
    // ... stack setup ...
    
    sub_ACA8B80(a1, v12, a3);
    unsigned int v17 = sub_ACA93E8(a1, sub_ACA49D0, &v12, ...);
    
    return v17;
}
```

This is stack frame setup. This is call preparation. This is the actual function invocation machinery. The offsets (16, -16) match Lua's internal stack structure. The way it manages the call frame matches the reference implementation.

I compared it against Lua 5.1 source code, and everything lined up. They took the real thing and compiled it straight in.

---

## The Bytecode Question

The games don't ship with `.lua` source files. They ship with bytecode. Compiled Lua.

So I hooked the load function and dumped what was coming in. Then I looked at the bytes:

```
4c 75 61 51 00 01 04 08 04 08 28 77 01 00 00...
```

That's `LuaQ` in hex. That's the Lua 5.1 bytecode header. Standard format.

I fed the raw bytecode into `unluac` (a Lua decompiler) and it understood it.

Or at least, it *should have*.

---

## The Problem: Nothing Was Easy

This is where things got hard.

Standard unluac 5.1 worked on some bytecode. But not all of it. Some bytecode it would refuse to decompile. Some it would corrupt. Some it would output garbage.

I thought: *Something is wrong. Either the bytecode is encrypted, or the opcodes are custom.*

I was right about the opcodes.

This game doesn't use standard Lua 5.1. It uses **slua-unreal**, which is a modified version of Lua built for Unreal Engine. slua-unreal adds custom opcodes and changes how bytecode is structured.

When I tried to use standard tools, they failed.

So I had to build my own decryption system.

---

## Three Months of Decryption

This took me **three months**. Not three days. Not three weeks. Three months.

Here's what I did:

**Month 1: Understanding the Problem**

I dumped a lot of bytecode and tried to understand what was different. I compared standard Lua 5.1 bytecode to the game's bytecode. 

Standard Lua bytecode:
```
LuaQ           // Header
01 04 08       // Version info
04 08 28 77    // Size info
00 00 00 01    // First opcode
```

Game bytecode:
```
LuaQ           // Same header
01 04 08       // Same version
04 08 28 77    // Same size
AA BB CC DD    // ?????? Custom opcode???
```

The headers were identical. But the opcodes were wrong.

I looked online for slua-unreal documentation. I found the GitHub repository. I found the opcode tables. And I realized: the game was using custom opcodes that I didn't understand.

Standard Lua has opcodes like MOVE, LOADK, ADD, CALL.

slua-unreal has different numbers for these. And it adds new opcodes that standard Lua doesn't have.

**Month 2: Building Tools**

I found **undec**, a tool that can parse bytecode structure. But it couldn't decode the custom opcodes.

So I:

1. Downloaded the slua-unreal source code
2. Read through the opcode definitions
3. Built a mapping: custom opcode -> standard Lua opcode
4. Created a bytecode transformer that converts slua bytecode to standard Lua bytecode
5. Tested it on dumped bytecode

This half-worked. Some bytecode transformed correctly. Some didn't.

I realized there was encryption on top of the custom opcodes.

**Month 3: Decryption + Testing**

The bytecode was encrypted. But encrypted how?

I hooked different parts of the load pipeline:

1. Hook at `lua_load()` entry – see what comes in
2. Hook at internal bytecode parser – see what gets parsed
3. Hook at opcode execution – see what actually runs

I noticed something: the bytecode that came into `lua_load()` had a pattern. It wasn't random. It looked like XOR encryption with a repeating key.

I tried different key sizes. Different patterns.

After weeks of testing, I found it: the bytecode was XORed with a key derived from the game's version and some other constants.

I built a decryptor:

```cpp
// Decrypt bytecode
for (int i = 0; i < size; i++) {
    bytecode[i] ^= key[i % keySize];
}
```

Then I ran it through my bytecode transformer. Then I ran it through unluac.

**It worked.**

I could see Lua source code.

---

## What I Could See

Once I finally decrypted the Lua, I could read the game's actual source code.

I found:

**The Ban System**
```lua
function CheckPlayerViolation(playerID, flags)
    -- Anti-cheat detection
    -- Ban logic
    -- Penalty system
end
```

**Skin System**
```lua
function LoadCharacterSkin(skinID)
    -- Skin model paths
    -- Effects
    -- Quality tiers
end
```

**Server Communication**
```lua
function SendPlayerStats(stats)
    -- Network protocol
    -- Data encryption
    -- Update frequency
end
```

**Game Features**
- Quest system
- Battle pass progression
- Matchmaking logic
- Ranking system
- Item shops
- Event management
- Player progression

**Everything was Lua.** Every game mechanic you interact with, it's all in Lua bytecode.

It was like finding the source code lying on the table.

---

## Why Was It Possible at All?

Because **slua-unreal is open source**. The developers published the opcode definitions and architecture online. So anyone who:

1. Knows how to hook a Lua loader
2. Understands bytecode structure
3. Has the opcode tables
4. Is patient enough to spend three months figuring out the encryption

...can dump and decode the bytecode.

The game encrypts the bytecode on disk. But the moment it loads into memory before the Lua VM processes it, it's in a format you can work with.

I hooked it right there.

---

## I Tested Lua Injection

Once I understood the system, I tried something bold:

**What if I inject my own Lua code into the pipeline?**

I built a hook that:
1. Waits for Lua bytecode to load
2. Intercepts it before the VM processes it
3. Replaces it with my own Lua code
4. Lets the game run my code instead

**It worked perfectly.**

I could inject custom Lua functions. I could modify how the game works. I could add features. I could change behavior.

The entire Lua pipeline is hackable because it's not protected—just encrypted on disk.

I put all this code on GitHub:
https://github.com/SnapyHook/UE4-LuaScripting-Pipeline

---

## What Does Lua Actually Do Here?

Lua is the **communication bridge** between the game client and the game servers.

**1. Receiving Server Data**
```lua
function OnServerUpdate(data)
    -- Server sends new features
    -- Server sends balance changes
    -- Server sends player-specific data
    -- Lua unpacks and applies it all
end
```

**2. Client-Side Logic**
```lua
function UpdatePlayerStats(kills, deaths)
    -- Calculate rankings
    -- Update UI
    -- Manage inventory
    -- Handle events
end
```

**3. Sending Data Back**
```lua
function SendPlayerAction(action)
    -- Send gameplay data to server
    -- Server logs it
    -- Server updates matchmaking
    -- Server updates rankings
end
```

**4. Dynamic Updates**

The game developers don't rebuild the entire game to add a new skin or fix a bug.

They just send new Lua code from the server.

The client downloads it, loads it, runs it.

Players don't need to update the app. The game updates itself through Lua.

**5. Game Behavior**

How guns work. How damage is calculated. How the ban system works. How skins render. Everything is controlled by Lua.

If there's a balance issue, they patch it with Lua code, not a full app update.

---

## The Communication Flow

Here's how it actually works:

```
Server 
  ↓ (sends Lua bytecode)
Client (downloads and stores)
  ↓
Game loads the bytecode
  ↓
Lua VM processes it
  ↓
Game features execute
  ↓ (sends gameplay data)
Server
  ↓ (responds with new Lua or changes)
Loop
```

The Lua code is **the actual protocol**. It's not just game logic—it's how the client and server communicate.

This is why reverse engineering it is possible and why it's so revealing. The entire communication structure is in plain Lua.

---

## The Tools That Made This Possible

**IDA Pro** – I could open the binary and actually read the code. Jump to cross-references, trace call graphs, identify patterns. Without it, I'm just looking at hex.

**Frida** – I could attach to the running game and hook functions without recompiling anything. Place breakpoints, inspect memory, watch function calls happen in real time.

**xhook** – For low-level function interception on the PLT (procedure linkage table). Clean, reliable, doesn't need root.

**A64InlineHook** – For ARM64 inline patching. When I needed to hook deep inside a function, not just at entry/exit, this was the tool.

**MSHookFunction** – The old-school way. Cydia Substrate style hooking. Sometimes simple is better.

**Android NDK + Makefiles** – I had to compile my hooks into .so files and inject them. Basic but essential.

**MuMu Player Pro** – Good ARM64 emulation on macOS. Stable. Fast enough to do real-time debugging.

**undec** – Bytecode parser for understanding bytecode structure.

**slua-unreal opcode tables** – Custom opcode definitions that made decryption possible.

---

## What I Learned

If you want to understand how a big native application works:

1. **Start with the SDK.** The class names tell a story. They show you the architecture.

2. **Use static analysis to find the entry points.** Where does the initialization happen? Where's the main loop? IDA will show you.

3. **Use runtime tools to watch it happen.** Frida lets you see the actual behavior. Hook the functions that matter.

4. **Compare against known implementations.** If it looks like standard Lua, compare it to Lua source code. If it looks like standard UE4, compare it to UE4 docs. Pattern matching is powerful.

5. **Be patient.** You're not going to understand everything in an hour. Three months of work is normal for the hard problems. Keep following the breadcrumbs, and they lead somewhere.

---

## What Comes Next

Now I'm planning to dig deeper into:

- Automated extraction of Lua prototypes and function signatures
- Mapping the event system to understand gameplay logic flow
- Comparing the implementation against the open-source `slua-unreal` project to see what's custom and what's standard
- Analyzing garbage collection behavior under real gameplay conditions
- Understanding how Lua state is synchronized with C++ state

But for now, I wanted to document what I found and how I found it. Because if you ever find yourself staring at Lua bytecode that won't decompile, you'll know what to do.

---

## The Code and More

Everything's on GitHub:
- https://github.com/SnapyHook/UE4-LuaScripting-Pipeline

Hit me up on LinkedIn if you want to discuss this stuff:
- https://www.linkedin.com/in/prashar-aryan/

---

*January 2, 2026 – Written after three months of bytecode decryption, countless failed unluac runs, too much coffee, and way too many cigarettes. No sleep. Just obsession.*
