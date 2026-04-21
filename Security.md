# Security Policy

## Supported contracts

The following deployed contracts on Arbitrum One (chain ID 42161) are the current live protocol surface:

| Contract | Address |
|---|---|
| TGBT token | `0x31228eE520e895DA19f728DE5459b1b317d9b8D8` |
| TemporalGradientCore | `0xF6556DDC7CdD3635A05428BD85BCf33A09F752e6` |
| MiningModule | `0xb2b3d9bC63993b725Aea36aC90601c22292F3171` |
| BatchMiningModule | `0xAf07E37D104E9be17639FE7a51B36972D4738651` |
| RandomnessModule | `0x583863CFC5EFc0106886BA485e1b67F0966584f9` |
| TokenomicsModuleV2 | `0x7B871bdeDdED0064C34e22902181A9a983C9E2ab` |
| RateLimitModule | `0x61dEEEf2B2956db3AD291c639939669cD5399c1B` |
| StaleBlockOracle | `0xdc4eDF632187d05da50393Af87D19A08f6986517` |

## Reporting a vulnerability

Please report security vulnerabilities privately to:

**emiliano.arlington@gmail.com**

Do **not** open a public GitHub issue for security-sensitive findings.

Please include:

- the affected contract address,
- a minimal reproduction or proof-of-concept,
- the suspected impact (e.g. unauthorized mint, unauthorized state change, reward draining, denial-of-service).

The project will acknowledge receipt within a reasonable timeframe and coordinate on disclosure. There is no bug bounty program at this time.

## Audit status

TGBT has **not** undergone a third-party security audit. Source code is public and verified on Arbiscan. The project does not describe itself as "audited" in any user-facing material.
