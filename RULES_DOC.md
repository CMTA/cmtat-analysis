# Rules documentation — improvements for the CMTAT / RuleEngine related content

Scope: `Rules/README.md`, `Rules/doc/README.md`, `Rules/doc/technical/guides/RULE_SEMANTICS.md`, `Rules/doc/technical/guides/DEPLOYMENT_SCRIPTS.md`, the per-contract pages under `Rules/doc/technical/contracts/`, and `Rules/CLAUDE.md` / `AGENTS.md`. Checked against Rules `a3fbc27` (`v0.6.0`), the CMTAT (`v3.3.0-rc3`) and RuleEngine (`v3.0.0-rc6`) commits it pins, and the vendored CMTAT `v3.0.0` in `RuleEngine/lib/CMTATv3.0.0` for the historical claims. The Rules test suite was not run here (source review only); every statement below about code behaviour was read from the source and cross-checked against the RuleEngine suite where a CMTAT integration test covers it.

Companion to `FEEDBACK.md` (cross-project review), `CMTAT_DOC.md` and `RULEENGINE_DOC.md`. IDs from `FEEDBACK.md` are referenced where a doc item is the counterpart of a finding there.

Priority key: **P1** — a reader acting on the text ends up with a wrong or broken deployment/rule · **P2** — wrong or stale, caught quickly · **P3** — polish.

---

## Summary

| # | File | Priority | Problem |
|---|---|---|---|
| U-1 | `CLAUDE.md` / `AGENTS.md` | P1 | Three claims contradict the code and the contract pages: `approveAndTransferIfAllowed` "only works in Topology B"; "bind the RuleEngine to an operation rule, not the token"; `RuleIdentityRegistry` "screens the minter … minter must be identity-verified (F-1)" (`FEEDBACK.md` R-1) |
| U-2 | `doc/README.md` §"How rules differ" | P1 | Repeats the stale "`RuleIdentityRegistry` also screens [the minter], so the minter must itself be identity-verified" — `RULE_SEMANTICS.md` §1 and the code say the minter is exempt |
| U-3 | `README.md` §Compatibility, `doc/README.md` §Compatibility | P1 | "Every other rule … uses the 3-argument path only" / "work across the full CMTAT ≥ v3.0.0 range" is wrong: on CMTAT v3.0.0–v3.2.0 mint and burn arrive on the **3-arg** overload, so every mint/burn spender leg (`RuleBlacklist`, `RuleSanctionsList`, `RuleERC2980`, `checkSpender`) is silently inert there, not just `RuleMintAllowance` |
| U-4 | `RULE_SEMANTICS.md` §1–§3 | P1 | The semantics tables — the page every other page points to as the single source — omit `RuleMaxBalance`, `RuleMaxTotalSupplyERC3643` and `RuleChainlinkPoRERC3643` |
| U-5 | `RULE_SEMANTICS.md:89` | P2 | "single adds revert, batch adds skip it" for `address(0)` — batch adds revert the whole batch (`AddressSetBatchLib`, I-12, every contract page) |
| U-6 | `doc/README.md` §"Which rule should I use?", §"Available validation rules", §"Summary tab"; `README.md` §"The rules" | P2 | Rule inventories disagree with each other: `RuleMaxBalance` and the two ERC-3643 cap variants missing from three of the four lists; `RuleConditionalTransferLightMultiToken` missing from the root README table |
| U-7 | `RULE_SEMANTICS.md` §1 header, `RuleSpenderWhitelist.md`, `RuleWhitelist.md` | P2 | CMTAT `batchTransfer` (minter moving own tokens) and self-`transferFrom` reach rules on the 4-arg path with **`spender == from`**; not normalised to "direct" on the CMTAT path, so `RuleSpenderWhitelist` and every `checkSpender` rule screen the minter as a delegated spender — undocumented (`FEEDBACK.md` R-4) |
| U-8 | `RuleConditionalTransferLight.md`, `doc/README.md` §Operational Notes | P2 | `approveAndTransferIfAllowed` makes **the rule contract** the ERC-20 spender; sibling spender-screening rules and CMTAT's frozen-spender check must admit it (`FEEDBACK.md` R-5); the note "`transferred()` is restricted to the single token bound via `bindToken`" predates `bindRuleEngine` |
| U-9 | `RULE_SEMANTICS.md` §2, `RuleSanctionsList.md`, `RuleIdentityRegistry.md` | P2 | Only the *unset* oracle case is documented ("fail-open"); a **set-but-broken** oracle/registry makes the rule's, the engine's and the token's `canTransfer` / `detectTransferRestriction` **revert** — contradicting "views must not revert" (`FEEDBACK.md` R-2) |
| U-10 | `RULE_SEMANTICS.md` §6, `RuleMaxBalance.md`, `RuleWhitelist.md` | P2 | CMTAT `forcedTransfer` never notifies the engine — stateful and observing rules are bypassed by the enforcer; the page discusses only T-REX's `forcedTransfer` (`FEEDBACK.md` C-2) |
| U-11 | `CLAUDE.md` §Conventions, `RULE_SEMANTICS.md` | P2 | The reentrancy invariant that keeps six unguarded CMTAT variants safe — `transferred*` hooks make no non-view external calls — is satisfied by every rule but stated nowhere as an authoring rule (`FEEDBACK.md` C-3) |
| U-12 | `RuleConditionalTransferLightMultiToken.md`, `doc/README.md` §Binding model | P2 | The rule advertises `IRule`, so `RuleEngine.addRule` accepts it and the page's case C′ (approvals stranded permanently) is one call away; say so until the ERC-165 answer is changed (`FEEDBACK.md` R-3) |
| U-13 | `RULE_SEMANTICS.md` §2 "Authoritative pre-flight view" | P3 | Column does not say *on which contract*: CMTAT's token-level `detectTransferRestriction(address(0), …)` is not a mint predictor, the engine's and the rules' are (`FEEDBACK.md` C-1) |
| U-14 | `doc/README.md` §Summary tab | P3 | "Security Audit planned in the roadmap" column with ☑ / ✘ and no legend; no rule has had an external audit |
| U-15 | `RuleMaxBalance.md` | P3 | Self-transfer at the cap is rejected (`from == to` not special-cased) — add to the documented edge cases (`FEEDBACK.md` R-6) |
| U-16 | `README.md` §Compatibility, `CLAUDE.md` §Toolchain | P3 | `git submodule update --init --recursive` clones `lib/chainlink-doc` (the whole Chainlink documentation site) and `lib/chainlink-ace`, plus the nested submodules of CMTAT, RuleEngine and ERC-3643; say which ones a build needs |
| U-17 | `doc/README.md` §Compatibility vs `README.md` §Compatibility | P3 | Two compatibility sections with different wording; keep one and link |

---

## U-1 · `CLAUDE.md` / `AGENTS.md` — three stale claims in the agent-facing guide (P1)

These two files are byte-identical by convention and are the first thing an AI agent or a new contributor reads; a stale claim there is more likely to be acted on than one in a spec.

1. **Line 23** — *"`approveAndTransferIfAllowed` only works in Topology B."* and *"⇒ bind the RuleEngine to an operation rule, not the token."* `RuleConditionalTransferLightBase` has had a separate `bindRuleEngine` since v0.5.0; `doc/technical/contracts/RuleConditionalTransferLight.md` says *"Works in **both** topologies, provided the bindings are set correctly"* and *"Always bind the **token** with `bindToken` — putting the RuleEngine there … `approveAndTransferIfAllowed` will revert."* The guide's advice is the exact mis-wiring the contract page warns against. (`RuleMintAllowance` *does* bind the engine — its `onlyBoundToken` gate has no separate engine slot — so the sentence is right for one operation rule and wrong for the other; say which.)
2. **Line 29** — *"`RuleIdentityRegistry`, `RuleBlacklist`, `RuleSanctionsList`, `RuleERC2980` do **not** [exempt mint/burn from the spender check] — they screen the minter … for `RuleIdentityRegistry` it means the minter must itself be identity-verified (see `CLAUDE_AUDIT.md` F-1)."* The code exempts mint: `RuleIdentityRegistryBase._detectTransferRestrictionFrom` guards with `from != address(0)` and its comment reads *"This is what makes an unverified MINTER able to mint to a verified recipient, exactly as the specification requires."* `RULE_SEMANTICS.md` §1 marks it "❌ exempt [3]". F-1 was fixed and the guide kept the pre-fix text. The three deny-list rules do screen the minter, so the sentence should drop `RuleIdentityRegistry` and the F-1 clause.
3. **Line 30-ish, I-12 paragraph** — correct here ("both single and batch adds revert"), but `RULE_SEMANTICS.md:89` says the opposite (U-5); the guide should be the one cited from the semantics page, not contradicted by it.

## U-2 · `doc/README.md` §"How rules differ" repeats the minter claim (P1)

> `RuleIdentityRegistry` also screens it, so the minter must itself be identity-verified

Same error as U-1 item 2, in the user-facing reference. This bullet sits directly above the link to `RULE_SEMANTICS.md`, which says the opposite in its first table. Replace with "`RuleIdentityRegistry` exempts the minter by default (ERC-3643: only the receiver must be verified); `checkSpender` screens the spender on `transferFrom` only".

## U-3 · Compatibility: "every other rule uses the 3-argument path only" (P1)

`README.md` §Compatibility:

| Rule | Minimum CMTAT | Why |
|---|---|---|
| `RuleMintAllowance` | **v3.3** | … |
| Every other rule | v3.0.0 | Uses the 3-argument path only. |

and `doc/README.md` §Compatibility: *"The other rules only use the 3-argument path and work across the full CMTAT ≥ v3.0.0 range."*

Neither sentence is true. Every validation rule implements the 4-arg `transferred(spender, …)` and screens the spender on it; what changed between CMTAT lines is *which operations* arrive on that overload (verified in `RuleEngine/lib/CMTATv3.0.0/contracts/modules/0_CMTATBaseCommon.sol:169-185` vs. the current `:129-152`):

| CMTAT | `transferFrom` | `mint` / `burn` | `batchTransfer` |
|---|---|---|---|
| v3.0.0 – v3.2.0 | 4-arg, `spender = _msgSender()` | **3-arg** (`transferred(0, to, v)` / `transferred(from, 0, v)`) | 3-arg |
| v3.3.0-rc0+ | 4-arg | **4-arg**, `spender = operator` | 4-arg, `spender == from` (rc3) |

So on CMTAT v3.0–v3.2 the rules "work" but enforce less than their configuration says, and nothing reverts to tell the operator:

| Rule | On CMTAT v3.0.0 – v3.2.0 |
|---|---|
| `RuleMintAllowance` | inert (already documented) |
| `RuleBlacklist`, `RuleSanctionsList`, `RuleERC2980` | a listed / sanctioned / frozen **minter or burner** is not caught (transfer-time spender screening on `transferFrom` still works) |
| `RuleWhitelist`, `RuleWhitelistWrapper`, `RuleIdentityRegistry` with `checkSpender` | flag never fires on mint/burn — same as their documented ERC-3643 behaviour, which is exactly the analogy to draw |
| `RuleSpenderWhitelist` | unaffected (it exempts mint/burn anyway) |
| cap rules, receiver whitelist, conditional transfer | unaffected |

This is the same "one leg inert" failure mode `RULE_SEMANTICS.md` §6 describes for ERC-3643 tokens, applied to older CMTAT. Replace the two-row table with the one above (or a link to a new `RULE_SEMANTICS.md` §7 "Older CMTAT"), and change "Every other rule | v3.0.0" to "v3.0.0 for the `from`/`to` legs; v3.3.0 for any mint/burn spender screening".

## U-4 · `RULE_SEMANTICS.md` tables omit three shipped rules (P1)

`RULE_SEMANTICS.md` is introduced everywhere as *"the single place those differences are laid out side by side"* and `CLAUDE.md` says *"consult it before assuming any rule behaves like its siblings"*. Its §1 (who is screened), §2 (unset oracle / stateful / authoritative view / codes) and §3 (overload surface) tables have 13 rows each and no row for:

- `RuleMaxBalance` — screens `to` only, burns exempt, mints capped, exemption list, codes 82–83, **not** supported on ERC-3643 (§6 says this; §1–§3 do not list it);
- `RuleMaxTotalSupplyERC3643` — same read path as the stock rule, write path re-phased, ERC-3643 only;
- `RuleChainlinkPoRERC3643` — same.

An integrator comparing `RuleMaxBalance` against `RuleMaxTotalSupply` (both cap rules, both read the token, one per-address) has to open the contract page to learn it is stateless, receiver-only and exempt-list driven. Add the three rows to all three tables; §5/§6 already have the prose.

## U-5 · `RULE_SEMANTICS.md:89` — batch `address(0)` semantics inverted (P2)

> The zero address can **never** enter any list (single adds revert, batch adds skip it)

`AddressSetBatchLib.addBatch` runs the zero guard on every element and reverts the whole batch; `RuleAddressSetInternal.sol:39`, `RuleAddressSet.sol:60`, `RuleERC2980Internal.sol:42`, `CLAUDE.md` I-12, `doc/README.md` §"Zero address in batch operations" and every contract page say so. This is the only sentence in the repo that still describes the pre-I-12 behaviour, and it is in the page that claims to be authoritative.

## U-6 · Four rule inventories, four different sets (P2)

| Where | Missing |
|---|---|
| `README.md` §"The rules" (15 rows) | `RuleConditionalTransferLightMultiToken` |
| `doc/README.md` §"Which rule should I use?" (13 rows) | `RuleMaxBalance`, `RuleMaxTotalSupplyERC3643`, `RuleChainlinkPoRERC3643` |
| `doc/README.md` §"Available validation rules" (10 names) | `RuleMaxBalance`, `RuleMaxTotalSupplyERC3643`, `RuleChainlinkPoRERC3643` |
| `doc/README.md` §"Summary tab" (13 rows + 2 external) | `RuleMaxBalance`, `RuleMaxTotalSupplyERC3643`, `RuleChainlinkPoRERC3643` (mentioned only in footnotes c and f) |
| `RULE_SEMANTICS.md` §1–§3 | the same three (U-4) |

`RuleMaxBalance` in particular has its own code range (82–83), its own contract page and its own audit note (`CLAUDE_ANALYSIS_MAXBALANCE.md`) but is absent from the "which rule should I use" table — the one place someone with the need "cap a single holder" would look. Suggest one generated-looking master table in `doc/README.md` and short lists elsewhere that link to it.

## U-7 · `spender == from` on the CMTAT path (P2)

`RULE_SEMANTICS.md` §1 defines its spender columns for `transferFrom`, mint and burn. Two CMTAT entrypoints produce a fourth shape, `transferred(spender, from, to, v)` with `spender == from`:

- `batchTransfer` (MINTER_ROLE moving its **own** balance) — `CMTAT/contracts/modules/0_CMTATBaseCommon.sol:150-152` threads `_msgSender()` as spender since v3.3.0-rc3 (NM-5);
- `transferFrom(msg.sender, to, v)` — a holder self-delegating, allowed by OpenZeppelin.

`RuleNFTAdapter._isDelegated` routes `spender == from` to the **direct** hook for the ERC-7943 and `ITransferContext` entrypoints and, deliberately, **not** for the CMTAT 4-arg path (the adapter's header comment and §"How each interface signals a direct transfer" explain why). Consequence, stated nowhere: `RuleSpenderWhitelist` blocks a minter's `batchTransfer` unless the minter is on the spender whitelist; `RuleWhitelist` / `RuleWhitelistWrapper` / `RuleIdentityRegistry` with `checkSpender`, and the frozen-spender leg of `RuleERC2980`, screen the minter a second time as spender. Add a "self-spender" note to §1 and one line to `RuleSpenderWhitelist.md` ("direct transfers are always allowed" is true for `transfer`, not for `batchTransfer`).

## U-8 · `approveAndTransferIfAllowed` and its neighbours (P2)

`RuleConditionalTransferLight.md` §`approveAndTransferIfAllowed` describes the allowance and the callback post-condition (NM-17) but not that, in Topology A, **the rule contract is the ERC-20 spender CMTAT reports to every other rule in the engine**: `RuleWhitelist(checkSpender)`, `RuleSpenderWhitelist`, the spender legs of `RuleBlacklist` / `RuleSanctionsList` / `RuleERC2980` / `RuleIdentityRegistry(checkSpender)`, and CMTAT's own frozen-spender check (`ValidationModule._canTransferisFrozenAndRevert`). Each must admit the rule address or the helper reverts. One paragraph under "Points worth knowing".

Also `doc/README.md` §Operational Notes › RuleConditionalTransferLight: *"`transferred()` is restricted to the single token bound via `bindToken`"* — since `bindRuleEngine` it is restricted to the bound token **or** the bound engine (`isTransferExecutor`). The Binding-model table two sections up is correct; this bullet is the leftover.

## U-9 · Oracle **broken** is not the same as oracle **unset** (P2)

`RULE_SEMANTICS.md` §2 "When its oracle/registry is unset": `RuleSanctionsList` "oracle == 0 ⇒ all allowed (fail-open)", `RuleIdentityRegistry` "registry == 0 ⇒ all allowed (fail-open)". The cap rules' cells go further ("a token that later reverts ⇒ mints rejected, code 51/78") because their read path is guarded. The two oracle rules' read path is not: `setSanctionListOracle` / `setIdentityRegistry` accept any non-zero address without a `code.length` probe, and `_detectTransferRestriction` calls `isSanctioned` / `isVerified` bare. A set-but-broken oracle (EOA, wrong contract, registry that starts reverting) therefore makes `canTransfer` / `canTransferFrom` / `detectTransferRestriction*` **revert** — on the rule, on the engine, and on the CMTAT token, which forwards them unguarded — while the library's own contract says *"These are views and must not revert"* (`CLAUDE.md`). On the write path it is fail-**closed** (every transfer reverts), which is the opposite of the "fail-open" the table promises for the neighbouring case.

Either align the setters and read path with `TotalSupplyCapManager` / `ChainlinkPoRFeedManager` (probe at configuration, `try/catch` + dedicated code on read) or add a third failure column to §2: "oracle set but reverting ⇒ views revert, transfers revert". `RuleSanctionsList.md` and `RuleIdentityRegistry.md` need the same sentence.

## U-10 · CMTAT `forcedTransfer` bypasses every rule (P2)

`RULE_SEMANTICS.md` §6 spends a paragraph on T-REX's `forcedTransfer` *notifying* compliance and what that means for a `RuleMaxBalance` ERC-3643 variant. The CMTAT side is the mirror image and is stated only obliquely (`RuleMaxBalance.md:129` "after a `forcedTransfer`"): `ERC20EnforcementModuleInternal._forcedTransfer` moves value with **no** `_checkTransferred`, so no rule sees it. An `ERC20ENFORCER_ROLE` holder can push a recipient above `maxBalance`, move tokens to a non-whitelisted address, bypass a conditional-transfer approval, and none of it debits or blocks anything. Observing rules self-heal (further receipts are blocked); stateful ones do not. One row in §6's "two differences" table ("Forced transfer: CMTAT — not notified; T-REX — notified") and a sentence in `RuleMaxBalance.md`, `RuleWhitelist.md` and `RuleConditionalTransferLight.md`.

## U-11 · State the reentrancy invariant as an authoring rule (P2)

Six of the nine RuleEngine-capable CMTAT variants (Debt, DebtEngine, Permit, UUPS, ERC1363, HolderList) have no reentrancy guard around the `transferred` callback and rely on the engine and its rules never handing control to untrusted code (`CMTAT/doc/modules/controllers/validationRuleEngine.md`). Every rule in this library satisfies that because every `transferred*` hook that calls out (`RuleSanctionsList`, `RuleIdentityRegistry`, `RuleChainlinkPoR`, `RuleMaxBalance`, `RuleMaxTotalSupply`, `RuleWhitelistWrapper`) is `view`, so the outbound calls are `STATICCALL`s. That is a property, not a policy: `CLAUDE.md` §Conventions lists "all `internal` functions virtual", "`require` with custom errors", "hooks are `view`" — but not "write-path hooks must not make non-view external calls". Add it, with the six variant names, so the next operation rule that wants to call a token or an oracle non-`view` is stopped at review.

## U-12 · `RuleConditionalTransferLightMultiToken` passes `addRule` (P2)

`RuleConditionalTransferLightMultiToken.md` §"Deployment topology" is thorough on why the engine wiring fails and the C′ case strands approvals permanently. It does not say that the rule's `supportsInterface` answers true for `RuleInterfaceId.IRULE_INTERFACE_ID` (`RuleConditionalTransferLightMultiToken.sol:40-42`), so `RuleEngine.addRule(rule)` **succeeds** and the only guard against C′ is the reader having read this page. Until the ERC-165 answer is narrowed (`FEEDBACK.md` R-3 — drop `IRule`, keep `RULE_ENGINE_INTERFACE_ID` for direct binding, which CMTAT's `setRuleEngine` does not ERC-165-check anyway), the page and the `doc/README.md` Binding-model row should say "`addRule` will not stop you".

## U-13 · "Authoritative pre-flight view" — on which contract? (P3)

`RULE_SEMANTICS.md` §2 names `canTransfer` / `canTransferFrom` as authoritative for most rules. Called on the **rule** or the **engine**, `detectTransferRestriction(address(0), to, v)` is a mint pre-check and every rule treats it as one. Called on the **CMTAT token**, it is not: CMTAT documents that its token-level `detectTransferRestriction*` do not accept the `address(0)` encoding (they answer `TRANSFER_REJECTED_PAUSED` while paused even though `mint` proceeds while paused); only `canTransfer` / `canTransferFrom` on the token branch on the zero address (`CMTAT/contracts/modules/wrapper/extensions/ValidationModule/ValidationModuleERC1404.sol` header). Add "on the rule/engine; on the token use `canTransfer*`" to the column header or a footnote.

## U-14 · "Security Audit planned in the roadmap" column (P3)

`doc/README.md` §Summary tab has a column mixing ☑, ✘ and "(community project)" with no legend and no date. No rule has had an external audit; the repo has a v0.4.0 manual threat model (`CLAUDE_AUDIT.md`), v0.5.0 code-quality reviews and Nethermind AuditAgent / Slither / Aderyn runs, all linked from §Security. Rename to "Audit status", fill with the actual reviews, or drop it — CMTAT's `doc/README.md` copies this column verbatim (`CMTAT_DOC.md` D-5), so fixing it here fixes two places.

## U-15 · `RuleMaxBalance` self-transfer edge case (P3)

`BalanceCapManager._capExceeded(to, value)` reads `balanceOf(to)` and adds `value`; `from == to` is not special-cased, so a holder exactly at `maxBalance` cannot `transfer(self, v)` although the net balance is unchanged. `RuleMaxBalance.md` already has sections for "Who is screened", "`maxBalance` has no magic value" and "Zero-value transfers to a holder already over the cap"; add a sibling for this one (a wallet "send to self" smoke test is the usual way it surfaces).

## U-16 · Submodule checkout instructions (P3)

`README.md` §Compatibility and `CLAUDE.md` §Toolchain: `git submodule update --init --recursive`. `.gitmodules` includes `lib/chainlink-doc` (the entire `smartcontractkit/documentation` site — reference material for `RuleChainlinkPoR`, not a build input) and `lib/chainlink-ace`, and `--recursive` also clones the nested submodules of `lib/CMTAT`, `lib/RuleEngine` (which itself carries a second CMTAT and an ERC-3643) and `lib/ERC-3643`, none of which the remappings use. Neither Chainlink repo is referenced by `remappings.txt`, `foundry.toml`, `src/`, `test/` or `script/` (verified by grep), so they are reference material only. State the minimal set (`forge-std`, `openzeppelin-contracts`, `openzeppelin-contracts-upgradeable`, `CMTAT`, `RuleEngine`, `ERC-3643`) or move the two Chainlink repos out of `lib/` into `doc/`.

## U-17 · Two compatibility sections (P3)

`README.md` §Compatibility (table + per-rule minimum) and `doc/README.md` §Compatibility (table + the "3-argument path only" prose, U-3) say the same thing in different words and will drift again. Keep the root one (it is the GitHub front page), make the `doc/README.md` one a link.

---

## Suggested order

1. U-1, U-2, U-3 — the guidance statements that are currently wrong (minter screening, topology, "3-arg only").
2. U-4, U-5, U-6 — make `RULE_SEMANTICS.md` actually complete and self-consistent, then derive the other inventories from it.
3. U-7 … U-12 — the CMTAT-side behaviours (self-spender, `forcedTransfer`, reentrancy invariant, broken oracle, `addRule` acceptance) that are true today and written nowhere.
4. U-13 … U-17 — polish.
