# MacOS Roblox Address Finding Guide (COSINE)

## Overview
The hardcoded addresses in the original injection code inside cosine are outdated and need to be updated for current Roblox versions. This just explains how to find the current addresses.

## Required Tools
1. **Hopper Disassembler** - For static analysis
2. **IDA Pro** or **Ghidra** - Alternative disassemblers
3. **lldb** - For dynamic analysis
4. **class-dump** - For Objective-C headers
5. **otool** - For Mach-O analysis

## Address Finding Guide

### 1. luau_load Function
**What it does**: Loads Luau bytecode into the Lua state
**Signature**: `int luau_load(lua_State* L, const char* chunkname, const char* data, size_t size, int env)`

**Step-by-step finding process**:
1. **Search for error strings**:
   ```
   "bytecode version mismatch"
   "malformed bytecode"
   "bytecode too old"
   ```
2. **Find the function that contains these strings**:
   - Open Hopper/IDA and search for these strings
   - Look at cross-references to find the function
   - This function is `luau_load`

3. **Verify the function**:
   - Should take 5 parameters (lua_State*, chunkname, data, size, env)
   - Should have bytecode validation logic
   - Should call other Lua functions like `lua_settop`

**Example pattern to look for**:
```assembly
; Check bytecode version
cmp     eax, 4          ; Expected version
jne     version_error   ; Jump if version mismatch
```

### 2. lua_settop Function
**What it does**: Sets the stack top to a specific index
**Signature**: `void lua_settop(lua_State* L, int idx)`

**Step-by-step finding process**:
1. **Look for stack manipulation**:
   - Search for functions that modify `L->top`
   - Look for index calculations with stack base

2. **Find the pattern**:
   ```assembly
   ; Typical lua_settop pattern
   mov     rdi, [rdi+stack_top_offset]  ; Load current top
   lea     rax, [rdi+rsi*8]             ; Calculate new top (idx * 8)
   mov     [rdi+stack_top_offset], rax  ; Set new top
   ```

3. **Verify by cross-references**:
   - Should be called by many other Lua functions
   - Often called with negative indices (-1, -2, etc.)
   - Used in error handling and cleanup

**Alternative method**:
- Search for string "stack overflow" - functions near this often use `lua_settop`

### 3. lua_tolstring Function
**What it does**: Converts Lua value to string with length
**Signature**: `const char* lua_tolstring(lua_State* L, int idx, size_t* len)`

**Step-by-step finding process**:
1. **Search for type checking patterns**:
   ```assembly
   ; Check if value is string type
   cmp     byte ptr [rax+type_offset], LUA_TSTRING
   je      already_string
   ```

2. **Look for string conversion logic**:
   - Functions that handle number-to-string conversion
   - Functions that check Lua value types
   - Functions that return both pointer and length

3. **Find by string references**:
   - Search for format strings like "%.14g" (number formatting)
   - Look for functions that handle string metatable

**Verification**:
- Should handle multiple Lua types (string, number, boolean)
- Should return NULL for non-convertible types
- Should set length parameter if provided

### 4. lua_pcall Function
**What it does**: Protected call for Lua functions
**Signature**: `int lua_pcall(lua_State* L, int nargs, int nresults, int errfunc)`

**Step-by-step finding process**:
1. **Search for error handling strings**:
   ```
   "error in error handling"
   "stack overflow"
   "C stack overflow"
   ```

2. **Look for setjmp/longjmp patterns**:
   ```assembly
   ; Protected call setup
   call    setjmp          ; Set error recovery point
   test    eax, eax        ; Check if returning from error
   jnz     error_handler   ; Handle error case
   ```

3. **Find by call patterns**:
   - Look for functions that save/restore stack state
   - Functions that handle multiple return values
   - Functions with error function parameter handling

**Verification**:
- Should have error recovery mechanism
- Should handle stack unwinding on errors
- Should be called by script execution functions

### 5. lua_newthread Function
**What it does**: Creates a new Lua thread (coroutine)
**Signature**: `lua_State* lua_newthread(lua_State* L)`

**Step-by-step finding process**:
1. **Search for memory allocation patterns**:
   ```assembly
   ; Allocate new lua_State
   mov     edi, lua_State_size
   call    malloc_function
   ```

2. **Look for thread initialization**:
   - Functions that copy global state
   - Functions that initialize stack
   - Functions that set up thread-specific data

3. **Find by string references**:
   - Search for "thread" in error messages
   - Look for coroutine-related strings

**Verification**:
- Should allocate memory for new lua_State
- Should initialize the new thread's stack
- Should link to parent state's global state

### 6. task.defer Function
**What it does**: Roblox's task scheduling function
**Signature**: Custom Roblox function

**Step-by-step finding process**:
1. **Search for Roblox-specific strings**:
   ```
   "task.defer"
   "task.spawn"
   "task.wait"
   "Heartbeat"
   ```

2. **Look in the task scheduler**:
   - Find the RunService or task scheduler
   - Look for functions that queue tasks
   - Search for deferred execution logic

3. **Find by job system references**:
   ```
   "WaitingHybridScriptsJob"
   "TaskScheduler"
   "DeferredTaskQueue"
   ```

**Pattern to look for**:
```assembly
; Task queuing pattern
lea     rax, [task_queue]
mov     [rax+queue_end], rdi    ; Add task to queue
inc     dword ptr [queue_count] ; Increment count
```

### 7. print Function
**What it does**: Roblox's print function with identity checking
**Signature**: Custom Roblox function

**Step-by-step finding process**:
1. **Search for identity strings**:
   ```
   "current identity is %d"
   "identity"
   "print"
   ```

2. **Look for output functions**:
   - Functions that write to console/output
   - Functions that format print arguments
   - Functions that check script permissions

3. **Find by cross-references**:
   - Look for functions called by Lua print
   - Search for output formatting code

**Verification pattern**:
```assembly
; Identity check in print
mov     eax, [rdi+identity_offset]
cmp     eax, required_identity
jl      permission_denied
```

### 8. Script Context / getgenv
**What it does**: Main script execution context and global environment
**Signature**: Various context-related functions

**Step-by-step finding process**:
1. **Search for context strings**:
   ```
   "ScriptContext"
   "GlobalEnvironment"
   "getgenv"
   "script context"
   ```

2. **Look for job scheduler references**:
   ```
   "WaitingHybridScriptsJob"
   "ScriptJob"
   "LuaSourceContainer"
   ```

3. **Find by script execution flow**:
   - Functions that set up script environment
   - Functions that manage global variables
   - Functions that handle script lifecycle

**Pattern identification**:
```assembly
; Script context access
mov     rax, [script_context_global]
mov     rdi, [rax+environment_offset]
call    setup_globals
```

## Practical Address Finding Walkthrough

### Step 1: Prepare Your Environment
```bash
# Create working directory
mkdir roblox_analysis
cd roblox_analysis

# Find and copy Roblox binary
find /Applications -name "RobloxPlayer*" -type d
cp "/Applications/RobloxPlayer.app/Contents/MacOS/RobloxPlayer" ./RobloxPlayer

# Check binary architecture and info
file RobloxPlayer
otool -h RobloxPlayer
otool -l RobloxPlayer | grep -A5 LC_SEGMENT_64
```

### Step 2: Initial String Analysis
```bash
# Extract all strings and filter for relevant ones
strings RobloxPlayer > all_strings.txt

# Search for Lua-related strings
grep -i "lua" all_strings.txt
grep -i "bytecode" all_strings.txt
grep -i "task" all_strings.txt
grep -i "script" all_strings.txt

# Look for specific error messages
grep "version mismatch" all_strings.txt
grep "stack overflow" all_strings.txt
grep "identity" all_strings.txt
```

### Step 3: Disassembly Analysis with Hopper

1. **Open RobloxPlayer in Hopper Disassembler**
2. **Wait for analysis to complete** (this may take several minutes)
3. **Use the search function** to find specific strings

**Finding luau_load**:
```
1. Search for "bytecode version mismatch"
2. Double-click the string to see references
3. Follow the reference to the function
4. This function is luau_load
5. Note the function address (e.g., 0x1234567890)
```

**Finding lua_settop**:
```
1. Search for "stack overflow"
2. Look at functions that reference this string
3. Find functions with stack manipulation patterns
4. Look for functions called frequently by other Lua functions
```

### Step 4: Cross-Reference Analysis

**Using Hopper's cross-reference feature**:
1. Right-click on a function name
2. Select "References to [function]"
3. Analyze calling patterns to verify function identity

**Example for lua_pcall**:
- Should be called by script execution functions
- Should have error handling code paths
- Should manipulate the Lua stack extensively

### Step 5: Dynamic Analysis with LLDB

```bash
# Start LLDB with Roblox
lldb RobloxPlayer

# Set breakpoints on suspected functions
(lldb) br set -a 0x1234567890  # Address from static analysis
(lldb) br set -n malloc        # Memory allocation
(lldb) br set -r "lua.*"       # Any function starting with "lua"

# Run and analyze
(lldb) run
(lldb) bt                      # Backtrace when breakpoint hits
(lldb) register read           # Check register values
(lldb) memory read $rdi        # Read memory pointed to by RDI
```

### Step 6: Pattern Verification

**For each function, verify these patterns**:

**luau_load verification**:
```assembly
; Should see bytecode validation
mov     eax, [rsi]              ; Load first 4 bytes
cmp     eax, 0x6C754C00         ; Compare with Luau signature
jne     invalid_bytecode

; Version check
movzx   eax, byte ptr [rsi+4]   ; Load version byte
cmp     eax, 4                  ; Expected version
jne     version_mismatch
```

**lua_settop verification**:
```assembly
; Stack bounds checking
mov     rax, [rdi+0x10]         ; Load stack base
mov     rcx, [rdi+0x18]         ; Load stack top
lea     rdx, [rax+rsi*8]        ; Calculate new top
cmp     rdx, [rdi+0x20]         ; Check against stack end
ja      stack_overflow
mov     [rdi+0x18], rdx         ; Set new top
```

**task.defer verification**:
```assembly
; Task queuing pattern
mov     rax, [task_scheduler]   ; Load scheduler
lea     rcx, [rax+queue_offset] ; Get queue
mov     [rcx+tail], rdi         ; Add task to queue
inc     dword ptr [queue_count] ; Increment count
```

### Step 7: Address Calculation

**Calculate final addresses**:
```bash
# Get base address (ASLR slide)
otool -l RobloxPlayer | grep -A5 LC_SEGMENT_64 | grep vmaddr

# Your found offset from disassembler
FUNCTION_OFFSET=0x1234567890

# In your C++ code, calculate runtime address:
# uintptr_t slide = _dyld_get_image_vmaddr_slide(0);
# uintptr_t function_addr = slide + FUNCTION_OFFSET;
```

## Advanced Techniques

### Automated Pattern Scanning
Create patterns for automatic detection:

```cpp
// Example pattern for luau_load
struct FunctionPattern {
    const char* name;
    const char* pattern;
    const char* mask;
    size_t offset;
};

FunctionPattern patterns[] = {
    {
        "luau_load",
        "\x48\x89\x5C\x24\x08\x48\x89\x6C\x24\x10\x48\x89\x74\x24\x18",
        "xxxxxxxxxxxxxxx",
        0
    },
    {
        "lua_settop",
        "\x48\x85\xD2\x78\x00\x48\x8B\x41\x10",
        "xxxx?xxxx",
        0
    }
};

uintptr_t find_function_by_pattern(const FunctionPattern& pattern) {
    // Implementation here
}
```

### Using IDA Pro Scripts
Automate address finding with IDAPython:

```python
import ida_search
import ida_bytes

def find_luau_load():
    # Search for bytecode version mismatch string
    addr = ida_search.find_text(0, 0, 0, "bytecode version mismatch", ida_search.SEARCH_DOWN)
    if addr != ida_idaapi.BADADDR:
        # Find function that references this string
        xrefs = ida_xref.get_first_dref_to(addr)
        while xrefs != ida_idaapi.BADADDR:
            func_addr = ida_funcs.get_func(xrefs)
            if func_addr:
                print(f"Found luau_load at: 0x{func_addr.start_ea:x}")
                return func_addr.start_ea
            xrefs = ida_xref.get_next_dref_to(addr, xrefs)
    return None

def find_lua_settop():
    # Search for stack manipulation patterns
    pattern = "48 85 D2 78 ?? 48 8B 41 10"  # Common lua_settop pattern
    addr = ida_search.find_binary(0, ida_idaapi.BADADDR, pattern, 16, ida_search.SEARCH_DOWN)
    if addr != ida_idaapi.BADADDR:
        func_addr = ida_funcs.get_func(addr)
        if func_addr:
            print(f"Found lua_settop at: 0x{func_addr.start_ea:x}")
            return func_addr.start_ea
    return None

# Run the searches
luau_load_addr = find_luau_load()
lua_settop_addr = find_lua_settop()
```

### Memory Signature Scanning
Implement runtime signature scanning:

```cpp
class SignatureScanner {
private:
    uintptr_t base_address;
    size_t module_size;

public:
    SignatureScanner() {
        // Get module info
        base_address = _dyld_get_image_vmaddr_slide(0);
        // Calculate module size
    }

    uintptr_t scan(const char* pattern, const char* mask) {
        for (size_t i = 0; i < module_size - strlen(mask); i++) {
            bool found = true;
            for (size_t j = 0; j < strlen(mask); j++) {
                if (mask[j] == 'x' &&
                    ((char*)(base_address + i))[j] != pattern[j]) {
                    found = false;
                    break;
                }
            }
            if (found) {
                return base_address + i;
            }
        }
        return 0;
    }
};

// Usage
SignatureScanner scanner;
uintptr_t luau_load = scanner.scan(
    "\x48\x89\x5C\x24\x08\x48\x89\x6C\x24\x10",
    "xxxxxxxxx"
);
```

## Troubleshooting Common Issues

### Issue 1: Function Not Found
**Problem**: String search doesn't find the expected function
**Solutions**:
1. **Check for string obfuscation**: Roblox may obfuscate error strings
2. **Look for similar strings**: Search for partial matches
3. **Use alternative methods**: Try pattern scanning instead
4. **Check different Roblox versions**: Strings may change between versions

### Issue 2: Wrong Function Identified
**Problem**: Found function doesn't behave as expected
**Solutions**:
1. **Verify function signature**: Check parameter count and types
2. **Analyze calling convention**: Ensure correct parameter passing
3. **Check cross-references**: Verify the function is called appropriately
4. **Test with simple cases**: Try basic function calls first

### Issue 3: Addresses Change After Updates
**Problem**: Addresses become invalid after Roblox updates
**Solutions**:
1. **Implement pattern scanning**: Use signatures instead of hardcoded addresses
2. **Create update detection**: Check for version changes
3. **Automate address finding**: Use scripts to find addresses automatically
4. **Version-specific configs**: Maintain address sets for different versions

### Issue 4: Anti-Debugging Detection
**Problem**: Roblox detects debugging attempts
**Solutions**:
1. **Use static analysis only**: Avoid runtime debugging
2. **Patch anti-debug checks**: Disable detection mechanisms
3. **Use VM or sandbox**: Isolate analysis environment
4. **Stealth debugging**: Use advanced debugging techniques

## Updating LibCosineMain.cpp

Once you've found the addresses, update the code:

```cpp
// In find_lua_functions()
bool find_lua_functions() {
    if (mock_mode) {
        // Keep mock mode for testing
        return true;
    }

    uintptr_t base_address = _dyld_get_image_vmaddr_slide(0);

    // Update these addresses with your findings
    g_addresses.luau_load = base_address + 0x123456789;  // Your found offset
    g_addresses.lua_settop = base_address + 0x234567890; // Your found offset
    g_addresses.lua_tolstring = base_address + 0x345678901;
    g_addresses.lua_pcall = base_address + 0x456789012;
    g_addresses.lua_newthread = base_address + 0x567890123;
    g_addresses.task_defer = base_address + 0x678901234;
    g_addresses.print_func = base_address + 0x789012345;
    g_addresses.script_context = base_address + 0x890123456;

    // Verify addresses are valid
    if (g_addresses.luau_load == base_address) {
        last_error = "Failed to find luau_load address";
        return false;
    }

    addresses_found = true;
    return true;
}
```

## Testing Your Addresses

Create test functions to verify addresses work:

```cpp
bool test_luau_load() {
    // Test with simple Luau bytecode
    const char* test_bytecode = "\x00\x4C\x75\x61\x04"; // Luau header

    // Create test Lua state (you'll need to implement this)
    // lua_State* L = create_test_state();

    // Test the function
    // int result = ((luauload_t)g_addresses.luau_load)(L, "test", test_bytecode, 5, 0);

    // return result == 0; // Success
    return true; // Placeholder
}

bool test_all_addresses() {
    return test_luau_load() &&
           test_lua_settop() &&
           test_lua_pcall();
}
```

## Version Compatibility

### Roblox Updates
- Addresses change with each Roblox update
- Pattern scanning is more reliable than hardcoded addresses
- Consider implementing automatic address resolution

### Testing
1. Test with current Roblox version
2. Verify all functions work correctly
3. Test script execution end-to-end
4. Check for crashes or memory corruption

## Security Considerations

### Code Signing
- Roblox may have code signing verification
- Injection may trigger anti-cheat detection
- Consider stealth techniques

### Anti-Cheat
- Roblox has built-in anti-cheat (Byfron)
- Avoid suspicious memory patterns
- Use legitimate API calls when possible

## Debugging Tips

### Common Issues
1. **Segmentation Faults**: Wrong addresses or calling conventions
2. **Access Violations**: Incorrect memory permissions
3. **Crashes**: Stack corruption or invalid parameters

### Debugging Commands
```bash
# Check crash logs
console | grep -i roblox

# Memory debugging
export MallocStackLogging=1
leaks RobloxPlayer
```

## Legal Notice
This information is for educational purposes only. Ensure compliance with Roblox Terms of Service and applicable laws.
