# mining: rename gbt_force and gbt_force_name (#32386)

## Background

In Bitcoin Core, the `getblocktemplate` (GBT) RPC provides miners with a template for constructing new blocks. This template includes version bits signaling for softfork deployments under [BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki#getblocktemplate-changes). To communicate activation rules to clients, some softfork names in the template are prefixed with a `!` character to denote changes that must be understood before using the template (e.g., BIP34 and BIP141). 

However, the term **"force"** was introduced in the code as a variable (`gbt_force`) and name (`gbt_vb_name`, later `gbt_force_name`) to represent this prefix behavior. This choice of terminology has led to confusion, for the following reasons:

### Ambiguity with BIP9 Force Terminology

[BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki#getblocktemplate-changes) does not define a "force" concept; instead, it uses rule prefixes to indicate non-backward-compatible template rules. Introducing `force` in variable names obscures the relationship to the `!` rule prefix.

## Poor Readability

The condition:
```cpp
if (!gbt_force) { s.insert(s.begin(), '!'); }
```
is difficult to read and understand at a glance, conflating a Boolean flag with the insertion of a rule prefix.

## Name Inconsistency and Renaming

Originally named `gbt_vb_name`, the variable was renamed to `gbt_force_name` in PR [#29039](https://github.com/bitcoin/bitcoin/pull/29039), further distancing the name from the underlying version‐bits semantics.

---

Together, these issues make the code harder understand and review, even though the existing behavior remains unchanged.

## Motivation for Renaming `gbt_force` to `gbt_optional_rule`

The PR renames the `gbt_force` variable and related naming instances to `gbt_optional_rule` across the codebase. This change improves clarity by:

**Aligning terminology with BIP9 semantics**:
  BIP9 specifies that certain rules must be explicitly acknowledged by clients if they are prefixed with a `!`. These are **not** optional. Conversely, rules without `!` are considered optional, which more accurately reflects the purpose of the renamed field.

**Improving readability**:
  Replacing `gbt_force` with `gbt_optional_rule` makes the logic surrounding rule handling in `getblocktemplate` more understandable at a glance.

**Simplifying mental model**: 
  By using a direct term like `optional_rule`, it becomes immediately obvious what the boolean flag controls.

---

## Files Touched and Role of Each File

### 1. `src/deploymentinfo.cpp`

This file defines the version bits deployments and their properties used throughout the codebase. Each deployment is initialized with a name and whether it is an optional rule for GBT(GetBlockTemplate).

#### Change:
Renamed `.gbt_force = true` to `.gbt_optional_rule = true` for deployments like `taproot` and `testdummy`.

```cpp
const std::array<VBDeploymentInfo, Consensus::MAX_VERSION_BITS_DEPLOYMENTS> VersionBitsDeploymentInfo{
    VBDeploymentInfo{
        .name = "testdummy",
        .gbt_optional_rule = true,
    },
    VBDeploymentInfo{
        .name = "taproot",
        .gbt_optional_rule = true,
    },
};
```

---

### 2. `src/deploymentinfo.h`

This header declares the `VBDeploymentInfo` structure. The renamed field clarifies that the GBT client can ignore rules marked as optional.

#### Change:
Renamed `gbt_force` to `gbt_optional_rule`:

```cpp
struct VBDeploymentInfo {
    const char *name;
    bool gbt_optional_rule;
};
```

---

### 3. `src/rpc/mining.cpp`

This file implements the `getblocktemplate` RPC, which generates a block template for miners. It processes active/locked-in/signaling version bits and uses `gbt_optional_rule` to determine whether a rule needs explicit client acknowledgment.

#### Change:
Replaced all logic using `gbt_force` with `gbt_optional_rule`, improving clarity in rule prefix logic:

```cpp
vbavailable.pushKV(gbt_rule_value(name, info.gbt_optional_rule), info.bit);

if (!info.gbt_optional_rule && !setClientRules.count(name)) {
    block.nVersion &= ~info.mask;
    // ...
}

if (!info.gbt_optional_rule && !setClientRules.count(name)) {
    throw JSONRPCError(RPC_INVALID_PARAMETER, strprintf("Support for '%s' rule requires explicit client support", name));
}
```

---

### 4. `src/versionbits.cpp`

Contains logic for determining the state of version bits and interfacing with deployment information. This file indirectly references `gbt_optional_rule` through the deployment metadata.

- No functional change here beyond renaming, ensuring consistency across the codebase.

---

### 5. `src/versionbits.h`

Header for `versionbits.cpp`,  pulling in `VBDeploymentInfo` from `deploymentinfo.h`.

---
## Miscellaneous: What I Learned

While reviewing this PR, I was able to deepen my understanding of two important concepts in the Bitcoin Core codebase and in modern C++:

---

### 📘 The `RPCHelpMan` Class

The `RPCHelpMan` class is a key component used to define and document Bitcoin Core’s RPC methods. It provides a structured way to define:

- The method name
- A description
- List of arguments (`RPCArg`)
- Return type (`RPCResults`)
- Examples (`RPCExamples`)
- And finally, the function (`RPCMethodImpl`) that handles the actual RPC logic

#### Key Features

1. **Argument Handling**
   - `Arg(key)` is used to access **required** arguments or those with **default values**.
   - `MaybeArg(key)` is used to access **optional** arguments that do **not** have a default.

2. **Type Safety with Templates**
   - The class uses template programming to return arguments in the correct type (`int`, `bool`, `std::string`, etc.)
   - For numbers (integral and floating point), values are returned directly.
   - For other types (like strings or objects), references or pointers are returned for efficiency.

3. **Automatic Validation and Mapping**
   - It checks whether a required number of arguments is passed (`IsValidNumArgs`).
   - It provides a map of arguments to be converted from string to JSON (`GetArgMap`).
   - It supports both **positional** and **named** arguments and can determine the index of a parameter by its name (`GetParamIndex`).

4. **Help Generation**
   - The `ToString()` method generates the help message automatically from the documentation provided in the constructor. This keeps user-facing RPC help messages in sync with the code logic.


---

### Understanding `std::any` in C++

While exploring the internals and template mechanisms used in `RPCHelpMan`, I also learned more about `std::any`, a type-safe container introduced in C++17.

#### What is `std::any`?

`std::any` is a type-safe container for single values of any type. Unlike `void*`, it knows the type of the value stored inside and ensures safe casting.

