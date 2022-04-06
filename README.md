## Primer for Review of CHIPS upgrade, Changes in Bitcoin Core (v0.16.0 to 0.21.0)

<h3 id="contents"> Contents:</h3>

- [Introduction](#intro)
- [Changes to API code](#api)
- [Relocation of `komodo_connectblock`](#connectblock)
- [Accessing chainstate and blockindex](#chainstate)
- [Changes to Keystore/ScriptPubKey Manager](#keystore)

<h3 id="intro"> Introduction </h3>


The following document will describe the changes to CHIPS codebase in upgrading from BTC(KMD) source version v0.16.0 to the most current v22.0.  We  will focus predominantly on addition of dPoW to the upgraded codebase, and requisite changes as they pertain specifically to integration with KMD infrastructure/code.  In doing so, we will also examine the ways BTC has changed in terms of: class inheritance, JSON-RPC API, access points for blockchain history, wallet code, and more.  All of which have direct impacts on the way dPoW must be integrated into the upgraded codebase(s).

___

<h3 id="api">Changes to API code</h3>

Since v0.16.0, RPC code in Bitcoin Core has undergone an overhaul.  This changes the way in which both client-side commands, and help text associated with them, are written into the codebase.  Coupled with wallet separation, and the added capability of storing multiple wallets (which may be loaded/unloaded individually), these changes require a re-structuring of many RPC's used by dPoW.

We will examine the differences in code via an accompanying video walkthrough, as it is easier than manually adding code snippets here.  Apart from the backend changes to API structure, wallet separation is a primary difference between v0.16.0 and v0.21.0 of Bitcoin source code.  Wallet separation does impact DPoW-utilized API endpoints.  We will discuss those impacted below, and propose a recommendation to accomodate the newly required `loadwallet` command into existing KMD infrastructure.

**API endpoints required for dPoW:**
  - getinfo
  - getblockchaininfo
  - validateaddress
  - listunspent
  - calc_MoM
  - getblockhash
  - getblock
  - getbestblockhash
  - sendrawtransaction
  - signrawtransactionwithwallet
  - decoderawtransaction
  
#### Wallet separation impact on RPC methods

Any method called via API that displays or accesses restricted wallet information now requires a loaded wallet, to complete successfully. Calling any of these without a wallet loaded will yield the following error: `No wallet is loaded. Load a wallet using loadwallet or create a new one with createwallet. (Note: A default wallet is no longer automatically created)`. 

Affected RPCs required for dPoW are as follows:

- `validateaddress`
- `getinfo`
- `listunspent`
- `signrawtransactionwithwallet`

All other dPoW-required methods are unaffected.

It should be noted, however, for codebases supporting NSPV comms: these changes may complicate wallet function substantially.

#### `validateaddress` RPC

DPoW requires an `ismine` key in the validateaddress response.  The function `isMine` is a member of `CWallet` class.  In order to call it, we need to access wallet information.  To preserve backward compatiblity, [`validateaddress` has been moved from `util` category of v22 API into the `wallet` category](https://github.com/who-biz/chipschain/commit/aa9642ea6a532ba25e3f792fe606a32b6970a98c).

#### `getinfo`/`getblockchaininfo` RPCs

Since v0.16.0, some RPCs require a loaded wallet to successfully provide a result.  Reasons for this change were discussed briefly, above.  On the positive side, the changes to wallet implementation indicate some added security.  On the negative side, it makes many dPoW-related requirements a bit more complicated to meet.  `getinfo` is one such RPC that requires a loaded wallet due to displaying wallet-based information.

`iguana`, the engine behind Komodo's dPoW, queries `getinfo` first, and then `getblockchaininfo` if the former fails.  As a result, we need to add notarization-based KVs to responses for both methods.  In the event a wallet is not loaded, `getblockchaininfo` must return all values required by iguana, for querying blockchain state of the dPoWed chain.  This does not appear to substantially interfere with efficiency of iguana.


#### Recommendations

**1. Add loadwallet to NN startup scripts for v0.21.0+ coins**

New BTC wallet functionality requires naming a wallet on creation with [`createwallet`](https://bitcoincore.org/en/doc/0.21.0/rpc/wallet/createwallet/).  This wallet can then be loaded by name with [`loadwallet main`](https://bitcoincore.org/en/doc/0.21.0/rpc/wallet/loadwallet/) where `main` is the name provided on wallet creation.  Once a wallet is loaded, it remains that way until daemon is powered down, or [`unloadwallet main`](https://bitcoincore.org/en/doc/0.21.0/rpc/wallet/unloadwallet/) is called.  For ease of integration with existing KMD infrastructure, it is recommended that standard practices are established for wallet management among elected notary nodes.  Startup scripts for `iguana` should be amended to include a call to `loadwallet` on v0.21.0+ daemon startup.

**2. Give special consideration to `wallet` category of API**

Since wallet separation seems to indicate some added security, special consideration should be given to which methods are included in `wallet` category of API.  These methods should be the only ones accessing sensitive wallet info. As in case of `validateaddress`, method should be moved to wallet category if access to wallet is required.

___


<h3 id="connectblock"> Relocation of `komodo_connectblock` and `komodo_disconnect`</h3>

Due to changes in the way blocks are added to index in new BTC source code, `ConnectBlock` in validation.cpp is no longer the proper place to call `komodo_connectblock()`.  If placed at this location, dPoW logic was attempting to validate *just before* the block was actually added to index.  This was resolved by [moving the `komodo_connectblock()` call to the end of `ConnectTip` in validation.cpp](https://github.com/who-biz/chipschain/commit/ed28788232390491d02c5425566b186bca7b98d6).  This resolved all issues in display of notarization values in `getinfo/getblockchaininfo` and their recording in `notarizations` file.

Currently, the location of `komodo_disconnect()` [function call has not been changed](https://github.com/who-biz/chipschain/blob/da385d1eff85f736921f91dfc8bfe90a98805802/src/validation.cpp#L1692), as `DisconnectTip()` calls `DisconnectBlock()`.  However, further testing should be performed to verify the proper placement of `komodo_disconnect`.  Relocation of `komodo_connectblock` strongly suggests `komodo_disconnect` may need moved as well, for dPoW to work properly.

___


<h3 id="chainstate"> Accessing chainstate and blockindex</h3>

Bitcoin has removed global access to chainstate and blockindex, requiring them to be accessed more indirectly than in 0.16.0 code.  This has been worked around for dPoW functions by addition of a global `NodeContext* g_rpc_node` pointer.  Address for pointer is populated on daemon init, and dPoW functions access it via global pointer.

CHIPS v22 upgrade was posted as a bounty in KMD discord.  Submitter of bounty used this method, which appears to run in contrast to practices in BTC.  Therefore, it is my recommendation that use of a global for accessing these values be avoided (and practices established by Bitcoin developers are more closely adhered to).  Impact(s) of defining a global for NodeContext struct, and accessing the variables within it, remains unknown.

Examples of an alternate approach, using Bitcoin-supplied `Ensure...` functions, for access to each variable in `NodeContext` can be seen here: https://github.com/who-biz/chipschain/commit/da385d1eff85f736921f91dfc8bfe90a98805802.  This commit has not been tested for proper functioning - it is merely meant as an example.

In addition to the removal of globals such as `chainActive`, naming of these access points for blockchain history have (in some instances) changed.  For example, `chainactive` must now be accessed through `chainman->ActiveChain()`.  It should be noted that `ActiveChain()` functions exactly the same as legacy `chainActive` in that: we can access `CBlockIndex` entries by using block height as an index.  This takes the form of `chainman->ActiveChain()[nHeight]`, which looks unwieldy but works.

---

<h3 id="keystore"> Changes to Keystore/ScriptPubKey Manager</h3>

The manner in which scripts are handled has changed substantially as well, with the upgrades to wallet code.  For examining these differences, we can compare the `ImportScript` function [from v0.16.0 code](https://github.com/who-biz/chips/blob/archive-0.16.0/src/wallet/rpcdump.cpp#L208-L231) to the [rewritten version for v22.0](https://github.com/who-biz/chipschain/commit/ab875f23fbe6b316745e8b0836e6a45f802a69c3#diff-c455c7835ec6d7a6d846c1ce3d3c2ad704b01d0c2215bffb5c28c9a5a1b33369R223-R248).  This, of course, does not encapsulate all differences, but serves as a nice high-level overview of the degree to which things have changed.

In v22.0 of Bitcoin core, a fair portion of legacy wallet functionality must be accessed through [`pwallet->GetLegacyScriptPubKeyMan()`](https://github.com/who-biz/chipschain/commit/ab875f23fbe6b316745e8b0836e6a45f802a69c3#diff-c455c7835ec6d7a6d846c1ce3d3c2ad704b01d0c2215bffb5c28c9a5a1b33369R231).  In this particular case, functions related to `WatchOnly` wallets are accessed through the legacy manager. Previusly, these could be accessed directly through `CWallet` pointer. 

It appears as though legacy functionality was preserved to make upgrading modified codebases easier.  Security implications of using legacy `ScriptPubKeyMan` are unclear.  It is recommended that this code is reviewed more thoroughly to discern impacts of using it.  At worst, security should still be comparable to v0.16.0 wallets.
