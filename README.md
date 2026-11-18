# CMTAT analysis

Workspace gathering the [CMTA](https://github.com/CMTA) CMTAT ecosystem repositories as git submodules, grouped by topic.

## Clone

```bash
git clone --recurse-submodules <this-repo-url>
```

If the repository is already cloned without submodules:

```bash
git submodule update --init --recursive
```

`--recursive` also fetches each project's own dependencies (e.g. OpenZeppelin), which are needed to build or test them.

## Submodules

Versions are the commit each submodule is pinned to (`git describe --tags`). `vX.Y.Z+N` means N commits after tag `vX.Y.Z`.

### EVM

| Path | Repository | Version |
| --- | --- | --- |
| `EVM/CMTAT` | [CMTA/CMTAT](https://github.com/CMTA/CMTAT) | v3.3.0 |
| `EVM/CMTAT-v3.0.0` | [CMTA/CMTAT](https://github.com/CMTA/CMTAT) | v3.0.0 |
| `EVM/RuleEngine` | [CMTA/RuleEngine](https://github.com/CMTA/RuleEngine) | v3.0.0-rc7 |
| `EVM/Rules` | [CMTA/Rules](https://github.com/CMTA/Rules) | v0.7.0 |
| `EVM/CMTAT-equivalency-assessment` | [CMTA/CMTAT-equivalency-assessment](https://github.com/CMTA/CMTAT-equivalency-assessment) | v0.3.0+7 |

### CMTAT-EXTEND

| Path | Repository | Version |
| --- | --- | --- |
| `CMTAT-EXTEND/CMTAT-ACE` | [CMTA/CMTAT-ACE](https://github.com/CMTA/CMTAT-ACE) | v0.3.0 |
| `CMTAT-EXTEND/CMTAT-FIX` | [CMTA/CMTAT-FIX](https://github.com/CMTA/CMTAT-FIX) | v0.2.0+2 |

### EVM utility

| Path | Repository | Version |
| --- | --- | --- |
| `EVM/utility/CMTAT-Factory` | [CMTA/CMTAT-Factory](https://github.com/CMTA/CMTAT-Factory) | v0.5.0 |
| `EVM/utility/DebtEngine` | [CMTA/DebtEngine](https://github.com/CMTA/DebtEngine) | v0.3.0 |
| `EVM/utility/DocumentEngine` | [CMTA/DocumentEngine](https://github.com/CMTA/DocumentEngine) | v0.4.0 |
| `EVM/utility/IncomeVault` | [CMTA/IncomeVault](https://github.com/CMTA/IncomeVault) | v2.0.0-rc0 |
| `EVM/utility/SnapshotEngine` | [CMTA/SnapshotEngine](https://github.com/CMTA/SnapshotEngine) | v0.5.0 |

### Confidentiality

| Path | Repository | Version |
| --- | --- | --- |
| `confidentiality/CMTAT-Confidential` | [CMTA/CMTAT-Confidential](https://github.com/CMTA/CMTAT-Confidential) | v1.0.0 |
| `confidentiality/private-CMTAT-aztec` | [CMTA/private-CMTAT-aztec](https://github.com/CMTA/private-CMTAT-aztec) | v0.5.0+3 |
| `confidentiality/CMTAT-Canton` | [CMTA/CMTAT-Canton](https://github.com/CMTA/CMTAT-Canton) | v0.1.0 |

### Other blockchains

| Path | Repository | Version |
| --- | --- | --- |
| `blockchain/CMTAT-Cardano` | [CMTA/CMTAT-Cardano](https://github.com/CMTA/CMTAT-Cardano) | v0.2.0 |
| `blockchain/CMTAT-Solana` | [CMTA/CMTAT-Solana](https://github.com/CMTA/CMTAT-Solana) | v1.0.0 |
| `blockchain/CMTAT-Tezos-FA2` | [CMTA/CMTAT-Tezos-FA2](https://github.com/CMTA/CMTAT-Tezos-FA2) | v1.0+1 |

## CMTA standard documents

The CMTA standard PDFs are available in [`EVM/CMTAT-equivalency-assessment/doc/`](EVM/CMTAT-equivalency-assessment/doc/):

- `cmtat-framework-functional-specifications-june-2026.pdf`: CMTAT framework functional specifications (June 2026)
- `cmta-standard-for-the-tokenization-of-debt-instruments-using-distributed-ledger-technology-may-2025.pdf`: CMTA standard for the tokenization of debt instruments (May 2025)

## Updating a submodule

```bash
cd EVM/Rules
git fetch --tags
git checkout <tag>
cd -
git add EVM/Rules
git commit -m "Bump Rules to <tag>"
```

Update the version in the table above at the same time. Check the current pins with `git submodule status`.

## Local files

`draft/` (questions, answers, feedback and archived notes) is listed in `.gitignore` and is not committed.
