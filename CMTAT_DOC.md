# CMTAT documentation — improvements for the RuleEngine / Rules related content

Scope: `CMTAT/README.md`, `CMTAT/doc/README.md`, `CMTAT/doc/technical/ruleengine-integration.md`, `CMTAT/doc/modules/controllers/validationRuleEngine.md`, and the places in the other technical pages that describe how CMTAT talks to a RuleEngine or a rule. Everything below was checked against CMTAT `5f07359` (`v3.3.0-rc3`), RuleEngine `bf7a203` (`v3.0.0-rc6`), Rules `a3fbc27` (`v0.6.0`) and, for the historical claims, against the CMTAT `v3.0.0` sources vendored in `RuleEngine/lib/CMTATv3.0.0`.

Companion to `FEEDBACK.md` (cross-project review). Items here are documentation-only; where a code finding from `FEEDBACK.md` needs a doc counterpart on the CMTAT side it is referenced by its ID.

Priority key: **P1** — a reader acting on the current text ends up with a wrong or broken deployment · **P2** — wrong or stale, but the mistake is caught quickly · **P3** — polish.

---

## Summary

| # | File | Priority | Problem | Dev comment |
|---|---|---|---|---|
| D-1 | `doc/README.md` §RuleEngine › Version | P1 | RuleEngine version table is stale (rc4 for v3.3.0, rc0 for "v3.0.+") and does not say *which* CMTAT features a given RuleEngine/Rules pairing actually delivers |  |
| D-2 | `doc/README.md` §RuleEngine › Rules | P1 | No Rules ↔ CMTAT compatibility table; per-rule CMTAT-version requirements (spender path) not stated | No need of per-rule compatibility table, but a warning/short summary could be done |
| D-3 | `doc/README.md` §RuleEngine › How it works | P1 | Narrative still says mint/burn call the 3-arg `transferred(from, to, value)`; since v3.3.0 they use the 4-arg overload |  |
| D-4 | `doc/technical/ruleengine-integration.md`, `doc/technical/erc-3643-implementation.md` | P1 | Source links are absolute local paths (`/home/ryan/...`) — dead on GitHub and in the PDF |  |
| D-5 | `doc/README.md` §RuleEngine › Rules | P2 | Rules table: wrong names, typos, "in development" for shipped rules, missing rules, external rule listed as if in `CMTA/Rules`; audit column should reflect the planned audit uniformly | Audit will only be on rules on CMTA/Rules |
| D-6 | `doc/README.md` §RuleEngine › How it works | P2 | Link to `CMTAT_BASE.sol#L198` at a 2023 commit; the file no longer exists |  |
| D-7 | `doc/README.md` §RuleEngine | P2 | "Rules can be set directly on CMTAT" note lacks the two caveats (ERC-3643 needs the engine; `MultiToken` is direct-only) and the ERC-165 non-check |  |
| D-8 | `doc/README.md`, `doc/technical/ruleengine-integration.md` | P2 | Missing: CMTAT `forcedTransfer` bypasses the engine and does **not** notify it (unlike T-REX) — see `FEEDBACK.md` C-2 |  |
| D-9 | `doc/README.md` §RuleEngine › Warning | P2 | "RuleEngine has to restrict `transferred` to the token" — add that the check must be on `msg.sender` (not an ERC-2771 `_msgSender()`), see `FEEDBACK.md` RE-1 |  |
| D-10 | `doc/modules/controllers/validationRuleEngine.md`, `doc/README.md` | P2 | The "no reentrancy guard on 6 variants" warning should state the rule-side invariant that keeps them safe (`transferred*` hooks must not make non-view external calls) — `FEEDBACK.md` C-3 |  |
| D-11 | `doc/README.md` §RuleEngine › Interface | P3 | `0x20c49ce7` is stated but not that it *excludes* `supportsInterface`, unlike RuleEngine's `IRule` id — `FEEDBACK.md` X-3 |  |
| D-12 | `doc/README.md` ERC-1404 code table | P3 | "leave the code 7-12 free" vs. Rules starting at 21 and RuleEngine saying "keep the CMTAT-reserved ranges free" — state one convention | Yes Rules start at 21, RuleEngine does not use code |
| D-13 | `README.md`, `CLAUDE.md` | P3 | Test count differs (5,626 vs 5,630); "v3.3.0" is described as a release while the newest tag is `v3.3.0-rc3`; companion-contract audit status (planned RuleEngine + Rules audit) absent from §Security | Next non candidate release will be v3.3.0 |
| D-14 | `doc/technical/ruleengine-integration.md` | P3 | Entry-point list is the only complete one; `doc/README.md` and the RuleEngine guide should point to it or copy it |  |

---

## D-1 · RuleEngine version table is stale and under-specified (P1)

`doc/README.md`, "###### Version" (lines ~1668–1682):

| CMTAT version | RuleEngine (current text) |
|---|---|
| CMTAT v3.3.0* | RuleEngine v3.0.0-rc4 (unaudited) |
| CMTAT v3.0.+ | RuleEngine v3.0.0-rc0 (unaudited) |
| … | … |

Problems:

1. **rc4 is two release candidates behind.** RuleEngine is at `v3.0.0-rc6`; rc5 and rc6 both target CMTAT `v3.3.0-rc3` per `RuleEngine/doc/technical/RuleEngine-with-CMTAT.md`. rc4 was cut against CMTAT `v3.3.0-rc1`.
2. **"CMTAT v3.0.+ → rc0" is misleading.** RuleEngine's own compatibility table says every release from rc2 on supports CMTAT ≥ v3.0.0, and its test suite deploys a real CMTAT v3.0.0 (`lib/CMTATv3.0.0`) against the current engine. Pointing v3.0.x/v3.1.x/v3.2.x users at rc0 (August 2025) hides rc1–rc6's fixes (interface-id computation, `maxRules` cap, token-binding rework, H-1 view-path documentation) from them. There is no row for v3.1.0 or v3.2.0 at all.
3. **The table answers "which engine compiles" but not "which behaviour you get".** The thing that actually changed between CMTAT lines is *what the engine receives*, and that decides which rules work:

   | CMTAT | `transfer` | `transferFrom` | `mint` / `burn` / `batchTransfer` / cross-chain | Consequence for Rules |
   |---|---|---|---|---|
   | v3.0.0 – v3.2.0 | 3-arg | 4-arg (`spender = _msgSender()`) | **3-arg** (`transferred(0, to, v)` / `transferred(from, 0, v)`), `batchTransfer` 3-arg | `RuleMintAllowance` is **inert** (never debited, `canTransfer` says OK); minter/burner is never screened as spender by `RuleBlacklist` / `RuleSanctionsList` / `RuleERC2980`; `RuleWhitelist.checkSpender` never fires on mint/burn |
   | v3.3.0-rc0+ | 3-arg | 4-arg | **4-arg** (`spender = operator`), `batchTransfer` 4-arg with `spender == from` (rc3, NM-5) | all rules behave as documented in `Rules/doc/technical/guides/RULE_SEMANTICS.md` |

   Verified against the vendored v3.0.0: `0_CMTATBaseCommon.sol:169-185` passes `address(0)` as spender for mint, burn and minter-transfer.

Suggested replacement table:

| CMTAT | RuleEngine | Rules | Notes |
|---|---|---|---|
| v3.3.0-rc3 (current) | [v3.0.0-rc6](https://github.com/CMTA/RuleEngine/releases/tag/v3.0.0-rc6) — **security audit planned** (latest release) | [v0.6.0](https://github.com/CMTA/Rules) — **security audit planned** (latest release) | Mint/burn/`batchTransfer` reach the engine on the 4-arg overload with the operator as `spender` |
| v3.3.0-rc1 / rc2 | v3.0.0-rc4 / rc5 | v0.4.0 – v0.5.0 | rc3 changed `batchTransfer` to forward the minter as spender (NM-5) |
| v3.0.0 – v3.2.0 | v3.0.0-rc2 … rc6 (CMTAT ≥ v3.0.0 supported; tested in `CMTATIntegrationV3.t.sol`) | v0.6.0 with the caveats in the row above | Mint/burn use the **3-arg** overload: spender-keyed rules (`RuleMintAllowance`) are inert |
| v2.5.0 (unaudited) | ≥ v2.0.3 | — | |
| v2.4.0 (unaudited) | ≥ v2.0.0 (last v2.0.2) | — | |
| v2.3.0 (audited) | v1.0.2 (audited) | — | |
| v2.0 (unaudited) | 1.0 | — | |
| v1.0 | none | — | |

Keep the `*` footnote but move its content into the "Notes" column — a footnote on "v3.3.0" is the only place today that says mint/burn are spender-aware, and it is easy to miss.

**Audit status wording.** A security audit is planned on the latest RuleEngine and Rules releases. The current table labels every v3 RuleEngine row "(unaudited)" with no distinction between "never audited" and "audit scheduled". Suggest three states, used consistently across the RuleEngine, Rules, SnapshotEngine, DebtEngine and DocumentEngine tables: **audited (firm, version, link)** · **audit planned (target version)** · **unaudited**. Once the audit is delivered, the row must name the audited commit/tag — the RuleEngine and Rules `version()` strings do not distinguish release candidates (`FEEDBACK.md` X-1), so a tag or commit hash is the only reliable identifier of what was reviewed. Until then the wording should make clear that a *planned* audit does not change the security posture of what is deployed today.

Dev comment: SnapshotEngine, DebtEngine and DocumentEngine => no audit is planned for these project

## D-2 · No Rules compatibility table; per-rule version requirements missing (P1)

The "###### Rules" section lists rules but never says which CMTAT/RuleEngine a given Rules release was built against, nor which rules need the v3.3.0 spender path. `Rules/README.md` has the answer (v0.6.0 → CMTAT ≥ v3.0.0 validated on v3.3.0-rc3, RuleEngine v3.0.0-rc6, OZ 5.7.0). Add the table from D-1 (or a link to `Rules/README.md#compatibility`) and, in the rules table itself, a "Requires" column:

| Rule | Requires |
|---|---|
| `RuleMintAllowance` | CMTAT ≥ v3.3.0 (mint via 4-arg `transferred`) — inert on v3.0–v3.2 |
| `RuleWhitelist` / `RuleWhitelistWrapper` with `checkSpender` on mint/burn, `RuleBlacklist` / `RuleSanctionsList` / `RuleERC2980` minter screening | CMTAT ≥ v3.3.0 for the mint/burn spender leg; transfer legs work on ≥ v3.0.0 |
| `RuleConditionalTransferLightMultiToken` | direct binding (`setRuleEngine(rule)`), never behind a RuleEngine |
| `RuleMaxTotalSupplyERC3643`, `RuleChainlinkPoRERC3643` | **ERC-3643 tokens only** — do not use with CMTAT (double-count) |
| `RuleMaxBalance` | CMTAT only (not supported on ERC-3643) |

The `RuleMintAllowance` cell links to `https://github.com/CMTA/Rules/blob/main/doc/technical/RuleMintAllowance.md`, which no longer exists — Rules moved the per-contract pages to `doc/technical/contracts/` in v0.6.0. Update to `…/doc/technical/contracts/RuleMintAllowance.md` (the only deep link from CMTAT into the Rules repo).

## D-3 · "How it works" still describes the pre-3.3.0 dispatch (P1)

`doc/README.md` §"##### How it works":

> Before each transfer (standard transfer/mint/burn), the CMTAT calls the ERC-3643 function `transferred` … `function transferred(address from, address to, uint256 value)`

and the numbered flow ("2. The validation module … calls … `transferred` … with the following parameters: `from, to, value`"). Since v3.3.0 only a plain `transfer` takes that overload; `transferFrom`, `mint`, `burn`, `batchMint`, `batchBurn`, `burnFrom`, `burn(uint256)`, `batchTransfer`, `crosschainMint`, `crosschainBurn` and both legs of `burnAndMint` take `transferred(spender, from, to, value)`. The later sub-sections ("TransferFrom - Spender restriction", "BurnFrom / CrosschainBurn - Operator restriction") add pieces of this, but plain `mint` / `burn` / `batchTransfer` are never said to be spender-aware anywhere in this chapter except the version-table footnote.

Replace the prose with one dispatch table (the one in `doc/technical/ruleengine-integration.md` §"Spender / Operator Semantics" is almost there; `RuleEngine/doc/technical/RuleEngine-with-CMTAT.md` §2 has the right shape but is itself incomplete — see `FEEDBACK.md` R-4):

| CMTAT entrypoint | `spender` | `from` | `to` | Overload |
|---|---|---|---|---|
| `transfer` | — | holder | recipient | 3-arg |
| `transferFrom` | `_msgSender()` | holder | recipient | 4-arg |
| `mint`, `batchMint`, `crosschainMint`, mint leg of `burnAndMint` | operator | `address(0)` | recipient | 4-arg |
| `burn`, `batchBurn`, `burnFrom`, `burn(uint256)`, `crosschainBurn`, burn leg of `burnAndMint` | operator | holder | `address(0)` | 4-arg |
| `batchTransfer` (MINTER_ROLE, own tokens) | minter | minter | recipient | 4-arg, **`spender == from`** |
| `forcedTransfer` | — | — | — | **not called** |

Also fix the sentence "the token holder calls the ERC-20 function `transfer` which triggers a call to the `RuleEngine` (ERC-3643 `transferred`)" in "##### RuleEngine CMTA implementation" the same way.

## D-4 · Absolute local file paths in two technical pages (P1)

`doc/technical/ruleengine-integration.md:15-24` and `doc/technical/erc-3643-implementation.md:16-61` link source files as `/home/ryan/Pictures/dev/CMTAT/contracts/...`. These render as dead links on GitHub and in the generated PDF, and leak a local path. Replace with repo-relative links (`../../contracts/interfaces/engine/IRuleEngine.sol`) — `doc/README.md` already uses that form. (`doc/security/tools/aderyn/v3.3.0-aderyn-report.md` contains the same prefix, but as tool output; leave it or strip it in the generator.)

## D-5 · Rules table content (P2)

In `doc/README.md` §"###### Rules":

- "Here are the list of rules in development" — Rules v0.6.0 is released; say "available in [CMTA/Rules](…) v0.6.0".
- Column header "Type [ready-only / read-write]" and cells "Ready-only" / "Ready-Write" → "Read-only" / "Read-write".
- `RuleSanctionList` → `RuleSanctionsList` (contract name).
- `RuleConditionalTransfer` lives in [CMTA/RuleConditionalTransfer](https://github.com/CMTA/RuleConditionalTransfer), not in `CMTA/Rules`; mark it external as the Rules README does.
- Missing rules shipped in v0.6.0: `RuleReceiverWhitelist`, `RuleMaxBalance`, `RuleChainlinkPoR`, `RuleChainlinkPoRERC3643`, `RuleMaxTotalSupplyERC3643`, and the registry contract `IdentityRegistryWhitelist` (not a rule — fills the ERC-3643 identity-registry slot).
- "Security Audit planned in the roadmap" column mixes ☑ / "In development" / ✘ with no legend. Keep the column — an audit of the latest Rules release is planned — but make it truthful and uniform: rename it "Audit status", give it a legend, and set every rule that is part of the planned audit scope to the same value ("planned — Rules v0.6.0" or whatever the audited tag will be) rather than the current per-rule mix of ☑ and "In development". Rules that are **out** of scope (external `RuleConditionalTransfer`, community `RuleSelf`) keep ✘ with a note. Today no rule has had an external audit (Rules has tool reviews and a Claude audit at v0.4.0/v0.5.0); when the report lands, replace "planned" with the firm and the audited tag. The same column exists in `Rules/doc/README.md` §Summary tab (`RULES_DOC.md` U-14) — fix both from one source, or link CMTAT's table to the Rules one so the audit status is maintained in one place.
- `RuleConditionalTransferLightMultiToken` description says "Bound directly to each token rather than shared through a RuleEngine" — good, but make it a warning: adding it to a RuleEngine either reverts every transfer or strands approvals (`Rules/doc/technical/contracts/RuleConditionalTransferLightMultiToken.md`).
- `RuleIdentityRegistry` description: "verifies that the transfer participants are registered" — per ERC-3643 and the implementation only the **receiver** is checked by default; sender/spender are opt-in flags. Say "receiver (sender/spender optional)".

## D-6 · Dead link to `CMTAT_BASE.sol` (P2)

`doc/README.md` "How it works": "…defined in [CMTAT_BASE](https://github.com/CMTA/CMTAT/blob/23a1e59f…/contracts/modules/CMTAT_BASE.sol#L198)". That file was split into `0_CMTATBaseCommon.sol` / `3_CMTATBaseRuleEngine.sol` in v3.0.0. Link to `../contracts/modules/3_CMTATBaseRuleEngine.sol` (the `_checkTransferred` override) instead of a pinned commit.

## D-7 · "Rules can be set directly on CMTAT" note needs its caveats (P2)

`doc/README.md` §"##### RuleEngine CMTA implementation":

> the rules for the CMTAT RuleEngine can now also be set **directly** on CMTAT … the RuleEngine is only needed when several rules must be combined on one token.

True, and correct for CMTAT — but the same sentence is read by people running the CMTA RuleEngine in front of an ERC-3643 token (the chapter opens by saying the engine "is also compatible with ERC-3643 tokens"). Add:

- a bare rule **cannot** back an ERC-3643 token (no `created` / `destroyed`), so ERC-3643 always needs the engine (`Rules/doc/README.md` §Topology B);
- `RuleConditionalTransferLightMultiToken` is the opposite: direct binding **only**;
- `setRuleEngine` does not ERC-165-check the address (design choice 1 in `ruleengine-integration.md`), so a rule that lacks `IRuleEngineERC1404` on an ERC-1404 deployment makes `detectTransferRestriction` / `messageForTransferRestriction` on the token revert while transfers keep working. All `CMTA/Rules` rules implement it; a custom rule may not.

## D-8 · `forcedTransfer` and the engine (P2)

Neither `doc/README.md` §RuleEngine nor `ruleengine-integration.md` states that `forcedTransfer` (and its burn form) **never** calls the engine — the only mention is the enforcement-module NatSpec. T-REX's `forcedTransfer` *does* notify compliance, and the Rules docs discuss that at length (`RULE_SEMANTICS.md` §6), so a reader coming from ERC-3643 assumes the same here. One sentence in "How it works": "`forcedTransfer` bypasses the RuleEngine entirely and does not notify it; stateful rules (conditional-transfer approvals, mint quotas) and observing rules (`RuleMaxBalance`, whitelists) are not consulted for enforcement moves." Cross-reference `FEEDBACK.md` C-2.

## D-9 · The "restrict `transferred` to the token" warning should say *how* (P2)

`doc/README.md` §Requirement › Warning: "The `RuleEngine` has to restrict the access of the function `transferred` to only the `CMTAT token contract`." CMTAT's own bridge gate explains why that check must use raw `msg.sender` (`ERC20CrossChainModule.sol:30-40`: "should never be impersonated using a relayer/forwarder"). Add the same sentence here — "compare `msg.sender`, not an ERC-2771 `_msgSender()`: the token never calls through a forwarder, and a forwarder-resolved sender lets the forwarder impersonate the token" — and to authoring guideline 1 in `ruleengine-integration.md`. The CMTA RuleEngine currently uses `_msgSender()` (`FEEDBACK.md` RE-1); if the CMTAT doc states the requirement precisely, the engine fix follows from it.

## D-10 · State the rule-side invariant behind the unguarded variants (P2)

> Dev comment: a configures rules is considered as trusted but if the rules called another cntract, it should be considered as trusted too

`doc/modules/controllers/validationRuleEngine.md` and `doc/README.md` say the RuleEngine "MUST NOT hand control to untrusted code during `transferred(...)`" on the six ❌ variants. Say what that means for a rule author in one testable sentence: "a rule's `transferred*` hooks must be `view`, or at least must not perform non-view external calls (`CALL`), so every outbound call is a `STATICCALL` and cannot re-enter the token." That is exactly the property every `CMTA/Rules` rule has today, and it is currently unstated on both sides (`FEEDBACK.md` C-3). Also list the six variants by name in `ruleengine-integration.md` §"Deployment Coverage", which today only says which variants *have* a RuleEngine.



## D-11 · `IRuleEngine` interface id: say what it covers (P3)

`doc/README.md` ~line 1436: "The ERC-165 interface id for the `IRuleEngine` interface is `0x20c49ce7`." That value is the XOR of the four functional selectors **without** `supportsInterface`; the RuleEngine repo's `IRule` id (`0x2497d6cb`) *includes* `IERC165`. Add "(computed over `transferred`×2, `canTransfer`, `canTransferFrom`; `IERC165.supportsInterface` is excluded, per the OpenZeppelin convention)" so a rule author reproducing it does not get `0x213b5540`. `contracts/mocks/ERC165Helper/ExampleRuleEngineERC165.sol` is the ground truth — link it.

## D-12 · Restriction-code reservation: one convention (P3)

`doc/README.md` ~line 1617 (in the `IERC1404Extend` listing): "leave the code 7-12 free/unused for further CMTAT additions". RuleEngine's guide says "Keep the CMTAT-reserved ranges free" without a number; `CMTA/Rules` starts at 21. State the reservation once, in prose outside the code block ("codes 0–6 are answered by the token, 7–20 are reserved for CMTAT; rules use ≥ 21") and have the RuleEngine guide link to it.

## D-13 · Version and count consistency (P3)

- `README.md:188` "5,626 automated tests" vs `CLAUDE.md:32` "5,630 tests" — pick one source (the coverage run) and reference it.
- `README.md` "Security" warning and `doc/README.md` version tables speak of "v3.3.0" as a release; the newest tag is `v3.3.0-rc3` and `VersionModule` already returns `"3.3.0"` (`FEEDBACK.md` X-1). Either tag `v3.3.0` or write "v3.3.0 (rc3)" consistently — the RuleEngine README links `releases/tag/v3.3.0`, which 404s today. Dev comment: We dont tag candidate release in the doc
- DocumentEngine / Factory tables use "CMTAT v3.3.0" while the RuleEngine table uses "CMTAT v3.3.0*"; align the footnote style.

`doc/technical/ruleengine-integration.md` §"Spender / Operator Semantics" is the only place that lists *every* CMTAT path that reaches the engine (classic `transferFrom`, mint/burn operator flows, cross-chain, `batchTransfer` minter path). `doc/README.md` §RuleEngine repeats a subset in prose, and `RuleEngine/doc/technical/RuleEngine-with-CMTAT.md` §2 repeats a smaller subset. Make the technical page the single source (add the table from D-3 to it) and have the other two link to it, so the next entry-point change (`batchTransfer` in rc3 was exactly such a change) is edited once.

---

## Suggested order

1. D-1, D-2, D-3 — the version table and the dispatch narrative are what an integrator reads to choose versions and write a rule; today they lead to the wrong engine release and the wrong overload.
2. D-4, D-6 — dead links, five-minute fix.
3. D-7, D-8, D-9, D-10 — the safety caveats that live only in NatSpec or in the sibling repos.
4. D-5, D-11 … D-14 — accuracy and polish.
