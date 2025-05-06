# mining: rename gbt_force and gbt_force_name (#32386)

## Background

In Bitcoin Core, the `getblocktemplate` (GBT) RPC provides miners with a template for constructing new blocks. This template includes version bits signaling for softfork deployments under [BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki#getblocktemplate-changes). To communicate activation rules to clients, some softfork names in the template are prefixed with a `!` character to denote changes that must be understood before using the template (e.g., BIP34 and BIP141). 

However, the term **"force"** was introduced in the code as a variable (`gbt_force`) and name (`gbt_vb_name`, later `gbt_force_name`) to represent this prefix behavior. This choice of terminology has led to confusion, for the following reasons:

### Ambiguity with BIP9 Force Terminology

[BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki#getblocktemplate-changes) does not define a "force" concept; instead, it uses rule prefixes to indicate non-backward-compatible template rules. Introducing `force` in variable names obscures the relationship to the `!` rule prefix.

### Poor Readability

The condition:
```cpp
if (!gbt_force) { s.insert(s.begin(), '!'); }
```
is difficult to read and understand at a glance, conflating a Boolean flag with the insertion of a rule prefix.

### Name Inconsistency and Renaming

Originally named `gbt_vb_name`, the variable was renamed to `gbt_force_name` in PR [#29039](https://github.com/bitcoin/bitcoin/pull/29039), further distancing the name from the underlying version‐bits semantics.

---

Together, these issues make the code harder to maintain and to review, even though the existing behavior remains unchanged.
