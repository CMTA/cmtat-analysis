# RuleEngine documentation — improvements for the CMTAT / Rules related content

Scope: `RuleEngine/README.md`, `RuleEngine/doc/README.md`, `RuleEngine/doc/technical/RuleEngine-with-CMTAT.md`, `RuleEngine/doc/technical/RuleEngine-with-ERC3643.md`, `RuleEngine/doc/technical/TokenBinding-module.md`, `RuleEngine/CLAUDE.md` / `AGENTS.md`. Checked against RuleEngine `bf7a203` (`v3.0.0-rc6`), CMTAT `5f07359` (`v3.3.0-rc3`, the commit RuleEngine pins), Rules `a3fbc27` (`v0.6.0`), and the vendored CMTAT `v3.0.0` in `lib/CMTATv3.0.0`. The RuleEngine test suite was built and run here (348/348), so the build-instruction claims below are verified, not inferred.

Companion to `FEEDBACK.md` (cross-project review) and `CMTAT_DOC.md`. IDs from `FEEDBACK.md` are referenced where a doc item is the counterpart of a finding there.

Priority key: **P1** — a reader acting on the text ends up with a wrong or broken deployment/rule · **P2** — wrong or stale, caught quickly · **P3** — polish.

---

## Summary

| # | File | Priority | Problem |
|---|---|---|---|
| E-1 | `doc/README.md` §"CMTAT v3.3.0 — mint and burn use the spender path" | P1 | "The 3-argument overload … does not occur in normal CMTAT v3.3.0 flows" — every plain `transfer` uses it (`FEEDBACK.md` RE-2) |
| E-2 | `doc/README.md` §Compatibility, §Dependencies; `doc/technical/RuleEngine-with-CMTAT.md` §Version compatibility | P1 | Three compatibility/dependency tables disagree with each other and with the pins: rc4 is the newest row, "CMTAT v3.3.0" tag does not exist, OZ stated as 5.6.1 (submodule is 5.7.0). Dev comment: CMTAT v3.3.0 tag will be created very soon onece the final version is approved |
| E-3 | `doc/technical/RuleEngine-with-CMTAT.md` §2, `README.md` §How it works | P1 | Entry-point table lists `transfer` / `transferFrom` / `mint` / `burn` only; `batchTransfer` (`spender == from`), `burnFrom`, `burn(uint256)`, `crosschainMint/Burn`, `burnAndMint` and the `forcedTransfer` bypass are missing (`FEEDBACK.md` R-4, C-2) |
| E-4 | `doc/README.md` §Gasless, §Requirement warning; `TokenBinding-module.md` §4 | P1 | Nothing says the bound-token gate is resolved through `_msgSender()`, i.e. the trusted forwarder can impersonate a bound token (`FEEDBACK.md` RE-1) — must be documented as a trust anchor or the code changed. Dev: no change to code for the moment |
| E-5 | `doc/README.md` §Available Rules, §Rules details; `README.md` §Rules | P2 | Rule lists are two Rules releases behind: `RuleSanctionList` (wrong name), `RuleMintAllowance`, `RuleReceiverWhitelist`, `RuleMaxBalance`, `RuleChainlinkPoR(ERC3643)`, `RuleMaxTotalSupplyERC3643`, `RuleConditionalTransferLightMultiToken` absent |
| E-6 | `doc/README.md` §Available Rules, §Motivation | P2 | "Directly on CMTAT (single-rule setup)" / "directly set the rule in the token" lack the two caveats: never for ERC-3643, always for `…MultiToken` |
| E-7 | `README.md` §Quick start, `CLAUDE.md` §Build | P2 | `git submodule update --init --recursive` and `cd lib/CMTAT && npm install` are both unnecessary (verified: `--init` alone + `forge test` passes with no `node_modules`) — `--recursive` pulls CMTAT's, CMTATv3.0.0's and ERC-3643's own submodules (`FEEDBACK.md` RE-5) |
| E-8 | `doc/README.md` §Gasless | P2 | Names `ERC2771ContextUpgradeable`; the engine uses `ERC2771Context` (`FEEDBACK.md` RE-3) |
| E-9 | `doc/README.md` §How to include it | P2 | Links `IRuleEngine.sol` at CMTAT commit `23a1e59` (2023, v2.x) — that interface has no 4-arg `transferred` and no `canTransferFrom` |
| E-10 | `doc/technical/RuleEngine-with-ERC3643.md` §4.8 | P2 | "The reference T-REX token cannot be compiled into this test suite" — Rules v0.6.0 does exactly that (separate `[profile.erc3643]` at solc 0.8.30, context-scoped onchain-id stubs); link to it instead of declaring it impossible |
| E-11 | `doc/technical/RuleEngine-with-CMTAT.md` §2 "Read-only path", `doc/README.md` §How it works step 2 | P2 | Says CMTAT "exposes the ERC-1404 view path" for everything; CMTAT documents that its token-level `detectTransferRestriction*` does **not** accept the `address(0)` mint/burn encoding the engine and every rule use (`FEEDBACK.md` C-1) |
| E-12 | `doc/technical/RuleEngine-with-CMTAT.md` §4.6, `doc/README.md` | P2 | Rule-authoring invariant behind CMTAT's six unguarded variants ("`transferred*` must not make non-view external calls") is not stated (`FEEDBACK.md` C-3) |
| E-13 | `doc/technical/RuleEngine-with-ERC3643.md` §4 | P2 | Post-update ordering (T-REX calls compliance **after** `_mint` / `_transfer`) is never mentioned although it is why three Rules need ERC-3643 variants |
| E-14 | `doc/README.md` §Production Deployment Checklist | P3 | Missing the two steps a CMTAT deployment actually needs after construction: `engine.bindToken(cmtat)` (CMTAT never self-binds) and `cmtat.setRuleEngine(engine)`; forwarder trust statement. Dev: to check but some CMTAT |
| E-15 | `doc/README.md` §ERC-165 table, `RuleInterfaceId` NatSpec | P3 | `IRuleEngine` id (`0x20c49ce7`) excludes `supportsInterface`, `IRule` id includes it — state the two conventions side by side (`FEEDBACK.md` X-3) |
| E-16 | `README.md`, `CLAUDE.md`, `doc/README.md` | P3 | "Version: 3.0.0" while the tag is `v3.0.0-rc6`; the ERC-3643 guide says OZ 5.7.0 and the README dependency list says 5.6.1 (`FEEDBACK.md` X-1, X-2) |
| E-17 | `doc/README.md` §Compatibility, §Functionality | P3 | Headings "CMTAT v3.0.0" used as a label for the current `setRuleEngine` API; "RuleEngine can be removed … CMTAT v3.0.0: `setRuleEngine`" — say "CMTAT ≥ v3.0.0" |



---

## E-1 · The README says the 3-arg overload is dead code (P1)

`doc/README.md` §"CMTAT v3.3.0 — mint and burn use the spender path":

> The 3-argument overload is only called when `spender == address(0)`, which does not occur in normal CMTAT v3.3.0 flows.

and the table row `transfer / transferFrom | caller / approved spender`. Every plain `transfer(to, value)` passes `address(0)` (`CMTAT/contracts/modules/0_CMTATBaseCommon.sol:75`) and `ValidationModuleRuleEngine._callRuleEngineTransferred` then calls the 3-argument `transferred(from, to, value)`. The root `README.md`, `CLAUDE.md` and `doc/technical/RuleEngine-with-CMTAT.md` §2 all say this correctly; only this paragraph is wrong, and it is the one under the "How to include it › Like CMTAT" heading that a rule author reads first. A rule author who trusts it will leave the 3-arg `transferred` unimplemented or unscreened (a no-op 3-arg hook lets every holder-initiated transfer through).

Fix: split the row into `transfer | — | 3-arg` and `transferFrom | _msgSender() | 4-arg`, and replace the sentence with: "A plain `transfer` carries no spender and takes the 3-argument overload; every other state-changing operation takes the 4-argument one."

## E-2 · Three version tables, three answers (P1)

| Where | Says | Reality |
|---|---|---|
| `doc/README.md` §Compatibility | newest row **rc4** → "CMTAT ≥ v3.0.0, target v3.3.0-rc1"; rc3 → "target [v3.3.0](…/releases/tag/v3.3.0)" | engine is rc6; rc5 and rc6 target CMTAT `v3.3.0-rc3` (pinned `658672f`); there is no CMTAT `v3.3.0` tag — the link 404s |
| `doc/technical/RuleEngine-with-CMTAT.md` §Version compatibility | rc5 → "≥ v3.0.0, target v3.3.0-rc3"; no rc6 row | rc6 is the current release and has the same target |
| `doc/README.md` §Dependencies | OZ (submodule) **v5.6.1**; CMTAT **v3.3.0** | `lib/openzeppelin-contracts` is `cab1993` = **v5.7.0** (`package.json` in the submodule); CMTAT is `v3.3.0-rc3` |
| `doc/technical/RuleEngine-with-ERC3643.md` §4.8 | "this project uses **5.7.0**" | correct |

The rc3 row is also chronologically inverted (an older engine targeting a newer CMTAT than the rc4 row).

Suggested single table, kept in `doc/README.md` and linked from the technical guide:

| RuleEngine | CMTAT supported | CMTAT tested against | Rules | Notes |
|---|---|---|---|---|
| v3.0.0-rc6 (current) | ≥ v3.0.0 | v3.3.0-rc3 (`lib/CMTAT`) and v3.0.0 (`lib/CMTATv3.0.0`) | v0.6.0 | Token-binding split; error renames `TokenBinding_*` |
| v3.0.0-rc5 | ≥ v3.0.0 | v3.3.0-rc3 | v0.5.0 | |
| v3.0.0-rc4 | ≥ v3.0.0 | v3.3.0-rc1 | v0.3.0 – v0.4.0 | |
| v3.0.0-rc3 | ≥ v3.0.0 | v3.2.0 | | |
| v3.0.0-rc2 | ≥ v3.0.0 | v3.2.0 | | |
| v1.0.2.1 | v2.3.0 | v2.3.0 | — | audited line |

And, right under it, the behavioural note that matters more than the version: **on CMTAT v3.0.0–v3.2.0 mint and burn arrive on the 3-arg overload** (verified in `lib/CMTATv3.0.0/contracts/modules/0_CMTATBaseCommon.sol:169-185`), so spender-keyed rules (`RuleMintAllowance`, minter screening in `RuleBlacklist` / `RuleSanctionsList` / `RuleERC2980`) are inert there. The "≥ v3.0.0" support claim is true for the engine and misleading for the rules; say both.

## E-3 · The CMTAT entry-point table is incomplete (P1)

`doc/technical/RuleEngine-with-CMTAT.md` §2 and `README.md` §How it works list `transfer`, `transferFrom`, `mint`, `burn`. CMTAT reaches the engine from more places, and two of them behave differently from anything in the table:

| CMTAT entrypoint | `spender` | `from` | `to` | Overload | Note |
|---|---|---|---|---|---|
| `transfer` | — | holder | recipient | 3-arg | |
| `transferFrom` | `_msgSender()` | holder | recipient | 4-arg | `spender == from` when a holder calls `transferFrom` on itself |
| `mint`, `batchMint`, `crosschainMint`, mint leg of `burnAndMint` | operator | `address(0)` | recipient | 4-arg | |
| `burn`, `batchBurn`, `burnFrom`, `burn(uint256)`, `crosschainBurn`, burn leg of `burnAndMint` | operator | holder | `address(0)` | 4-arg | `burn` and `burnFrom` are indistinguishable to the engine |
| `batchTransfer` (MINTER_ROLE, own tokens) | minter | **minter** | recipient | 4-arg | **`spender == from`** since CMTAT 3.3.0-rc3 (NM-5): `RuleSpenderWhitelist` and any `checkSpender` rule screen the minter as a delegated spender |
| `forcedTransfer` / forced burn | — | — | — | **not called** | the enforcer bypasses the engine; stateful rules are not notified |

`CMTAT/doc/technical/ruleengine-integration.md` §"Spender / Operator Semantics" already lists all of these; copy the table or link it. The `spender == from` row is the one that bites: `Rules/src/rules/validation/abstract/core/RuleNFTAdapter.sol` normalises `spender == from` to the direct hook on the ERC-7943 path but deliberately not on the CMTAT 4-arg path, so the engine's guide is where an integrator would expect to learn it.

## E-4 · Bound-token gate is forwarder-resolvable and nobody says so (P1)

`TokenBindingModule._checkBoundToken` compares `_msgSender()`, and all three deployables are `ERC2771Context`. With a forwarder configured, a call `forwarder → engine.transferred(...)` with the bound token appended as ERC-2771 sender passes `onlyBoundToken` (PoC in `FEEDBACK.md` RE-1: a mint quota in a `RuleMintAllowance`-style rule is consumed with no mint). Three places should say this, none does:

- `doc/README.md` §"Gasless support (ERC-2771)": add "the forwarder is trusted to the same degree as a bound token: it can reach `transferred` / `created` / `destroyed` on any bound token's behalf".
- `doc/README.md` §Requirement › Warning ("The `RuleEngine` has to restrict the access of `transferred` to only the CMTAT token contract") and `doc/technical/RuleEngine-with-ERC3643.md` §4.5 ("Only bound tokens may call the callbacks"): qualify with "or the trusted forwarder".
- `doc/technical/TokenBinding-module.md` §4 Operational warnings: add a bullet; the module is advertised as reusable, so the hook's `_msgSender()` dependency is part of its contract.

Better: fix the code to `msg.sender` (a token never calls through a forwarder — CMTAT's `onlyTokenBridge` makes the same choice, `ERC20CrossChainModule.sol:30-40`, and the Rules repo removed ERC-2771 from every rule that uses the sender as a binding identity, `CLAUDE_AUDIT.md` F-11) and then document that `bindToken` / `unbindToken` remain relayable while the callbacks are not.

## E-5 · Rule lists are stale (P2)

`doc/README.md` §"Available Rules" families table, §"Rules details" table, and `README.md` §Rules all list the same nine rules and all spell `RuleSanctionList` (contract: `RuleSanctionsList`). Rules v0.6.0 ships in addition: `RuleMintAllowance` (operation rule, the only one that *uses* the spender the engine forwards on mint — it belongs in every CMTAT-facing table here), `RuleReceiverWhitelist`, `RuleMaxBalance`, `RuleChainlinkPoR`, `RuleChainlinkPoRERC3643`, `RuleMaxTotalSupplyERC3643`, `RuleConditionalTransferLightMultiToken`, plus `IdentityRegistryWhitelist` (not a rule). The families table also has `RuleMaxTotalSupply` but not its ERC-3643 twin, which is the one an ERC-3643 integrator reading *this* repo needs.

Either keep one short table with a "last synced with Rules vX" line, or drop the per-rule table and keep only the two-family description with a link — the "source of truth" sentence already says the Rules repo is authoritative, so a stale copy here is worse than none.

## E-6 · "Set the rule directly on the token" needs its two exceptions (P2)

`doc/README.md` §Motivation ("directly set the rule in the token contract instead of using a RuleEngine") and §Available Rules ("Directly on CMTAT (single-rule setup)"). Add, in both places:

- **Never for ERC-3643.** A bare rule implements neither `created` nor `destroyed`, so `Token.mint` / `Token.burn` revert; only the engine implements the full `ICompliance` surface (`Rules/doc/README.md` §Topology B).
- **Always for `RuleConditionalTransferLightMultiToken`.** It keys approvals on `msg.sender`, so behind an engine it either reverts every transfer or strands approvals (`Rules/doc/technical/contracts/RuleConditionalTransferLightMultiToken.md`). It nevertheless advertises `IRule`, so `addRule` accepts it (`FEEDBACK.md` R-3) — worth a warning in §Available Rules until that changes.

## E-7 · Build instructions ask for two steps that are not needed (P2)

`README.md` §Quick start and `CLAUDE.md` §"Build & Test Commands":

```bash
git submodule update --init --recursive   # or: forge install
cd lib/CMTAT && npm install && cd ../..   # CMTAT's own OpenZeppelin deps
```

- `--recursive` recurses into `lib/CMTAT`, `lib/CMTATv3.0.0` and `lib/ERC-3643`, cloning their `forge-std`, `openzeppelin-contracts-upgradeable`, etc. None of that is used: `remappings.txt` resolves `@openzeppelin/contracts/` and `@openzeppelin/contracts-upgradeable/` to this repo's own submodules for every file, including the CMTAT sources under `lib/`.
- `npm install` in `lib/CMTAT` is likewise unused for the same reason (Hardhat is the only consumer, and `hardhat.config.js` here uses `hardhat-foundry`).

Verified: `git submodule update --init` (non-recursive) followed by `forge test` → 348 passed, no `node_modules` anywhere. Replace with `git submodule update --init` and note that `--recursive` is only needed if a contributor wants to run CMTAT's own Hardhat suite from inside `lib/CMTAT`.

## E-8 · `ERC2771ContextUpgradeable` (P2)

`doc/README.md:607`. The engine is not upgradeable (§Upgradeable says so a few lines later) and inherits `ERC2771Context` through `ERC2771ModuleStandalone`. Also the sentence "the parameter `forwarder` inside the RuleEngine contract constructor" — the parameter is `forwarderIrrevocable` in all three constructors.

## E-9 · `IRuleEngine.sol` linked at a 2023 CMTAT commit (P2)

`doc/README.md` §"How to include it" links `github.com/CMTA/CMTAT/blob/23a1e59f…/contracts/interfaces/engine/IRuleEngine.sol`. That revision predates the 4-argument `transferred`, `canTransferFrom` and the ERC-165 inheritance; a reader copying the interface from it gets a v2 engine that CMTAT ≥ v3.0.0 cannot call for `transferFrom`. Link the interface at the pinned CMTAT tag (`v3.3.0-rc3`) or, better, the local copy `lib/CMTAT/contracts/interfaces/engine/IRuleEngine.sol`. The same 2023 commit is linked from CMTAT's own README (`CMTAT_DOC.md` D-6).

## E-10 · "The reference T-REX token cannot be compiled" — Rules did it (P2)

`doc/technical/RuleEngine-with-ERC3643.md` §4.8 lists three blockers (exact `0.8.17` pragma, OZ 4.8, `onchain-id`). Rules v0.6.0 vendors a newer T-REX (`4.2.0-beta1`, pragma `0.8.30`) and compiles it in a second Foundry profile (`[profile.erc3643]`, `solc = "0.8.30"`, `out-erc3643/`, context-scoped remapping `lib/ERC-3643/:@onchain-id/solidity/contracts/=test/utils/onchainid/`) — see `Rules/CLAUDE.md` §Toolchain and `Rules/test/ERC3643Real/`. The engine's integration tests still use `ERC3643TokenMock`. Either adopt the same setup here (the `ICompliance` ordering claims in §2 would then be pinned against the real token) or replace "cannot" with "is not, see Rules for how".

Same section, the OZ version is stated correctly as 5.7.0 — fix §Dependencies to match (E-2).

## E-11 · Read-only path: the token does not accept the engine's `address(0)` encoding (P2)

`doc/technical/RuleEngine-with-CMTAT.md` §2 "Read-only path" says CMTAT exposes `detectTransferRestriction` / `canTransfer` / `…From` and the engine "answers by iterating the same rules". True for the engine. On the **token**, CMTAT documents (`ValidationModuleERC1404.sol` header, `doc/technical/ruleengine-integration.md`) that `detectTransferRestriction(address(0), to, v)` is *not* a mint predictor and answers `TRANSFER_REJECTED_PAUSED` while paused even though `mint` proceeds while paused; only `canTransfer` / `canTransferFrom` on the token branch on the zero address. Every Rules view *does* treat `from == address(0)` as a mint. Add one sentence: "to pre-flight a mint or burn, call `canTransfer*` on the token, or `detectTransferRestriction*` on the engine directly — not `detectTransferRestriction*` on the token."

## E-12 · State the rule-side reentrancy invariant (P2)

`doc/technical/RuleEngine-with-CMTAT.md` §4.6 "Rules are trusted code" says rules "can revert or consume arbitrary gas". The stronger obligation comes from CMTAT: six of the nine RuleEngine-capable variants (Debt, DebtEngine, Permit, UUPS, ERC1363, HolderList) have **no** reentrancy guard around `transferred`, and CMTAT's stated trust assumption is that the engine "MUST NOT hand control to untrusted code during `transferred(...)`" (`CMTAT/doc/modules/controllers/validationRuleEngine.md`). The engine forwards to every rule, so the obligation lands on rule authors: `transferred*` hooks must not make non-view external calls (all `CMTA/Rules` hooks are `view`, so their oracle/registry/feed calls are `STATICCALL`s). Add it to §4.6 and to the "Adding a new rule" recipe in `CLAUDE.md`.

## E-13 · ERC-3643 guide: mention post-update ordering (P2)

`doc/technical/RuleEngine-with-ERC3643.md` §2 quotes `Token.sol` for *which* hooks are called but not *when*: T-REX runs `_transfer` then `transferred`, `_mint` then `created`, so a rule observing `totalSupply()` / `balanceOf()` sees the amount already applied. That is why Rules ships `RuleMaxTotalSupplyERC3643` and `RuleChainlinkPoRERC3643` and marks `RuleMaxBalance` unsupported on ERC-3643 (`RULE_SEMANTICS.md` §6). A §4.9 "Compliance is notified after the state change" with a pointer there would stop an integrator from putting the stock cap rules behind an engine bound to a T-REX token — a silent, fail-closed misconfiguration.

## E-14 · Production deployment checklist is missing the wiring steps (P3)

`doc/README.md` §"Production Deployment Checklist" has variant, forwarder, "bind in constructor or later", rules source, permissions. Add:

- `engine.bindToken(cmtat)` — CMTAT never self-binds (`RuleEngine-with-CMTAT.md` §3.2 says so; the checklist does not), so an engine deployed with `tokenContract = address(0)` silently rejects every transfer with `TokenBinding_UnauthorizedCaller` until this is done;
- `cmtat.setRuleEngine(engine)` by the token's `DEFAULT_ADMIN_ROLE`;
- for ERC-3643: `setTokenSelfBindingApproval(token, true)` **before** `setCompliance`;
- "if a forwarder is set, it must be a signature-verifying ERC-2771 forwarder; it can reach the bound-token callbacks" (E-4).

## E-15 · Two ERC-165 conventions in one table (P3)

`doc/README.md` §"ERC-165 Support" lists `IRuleEngine = 0x20c49ce7` next to the note that `IRule = 0x2497d6cb` is "computed over the full hierarchy". They are not computed the same way: `0x20c49ce7` (CMTAT's constant) excludes `IERC165.supportsInterface`; `0x2497d6cb` includes it (`RuleInterfaceId.sol` NatSpec lists `IERC165` explicitly). Both are correct and test-pinned; say so in the notes so a rule author does not "fix" one of them.



## E-17 · Headings that use "CMTAT v3.0.0" as a name for the current API (P3)

`doc/README.md` §"How to set it › CMTAT v3.0.0" and §"Functionality › CMTAT v3.0.0: `setRuleEngine(address ruleEngine)`". `setRuleEngine` exists unchanged from v3.0.0 through v3.3.0-rc3; "CMTAT ≥ v3.0.0" avoids the reading that later versions differ. (The signature is `setRuleEngine(IRuleEngine)`, not `address`.)

---

## Suggested order

1. E-1, E-3, E-4 — the three statements a rule author or deployer acts on directly.
2. E-2, E-16 — one compatibility table, one dependency list, both matching the pins.
3. E-5, E-6, E-9, E-11, E-12, E-13 — rule-list and caveat content that currently lives only in the Rules or CMTAT repos.
4. E-7, E-8, E-10, E-14, E-15, E-17 — build instructions and polish.
