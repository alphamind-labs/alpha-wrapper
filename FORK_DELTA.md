# Fork delta

This fork carries the full, weighted validator logic on top of the public
upstream (bittensor-church/alpha-wrapper), whose production validator source
is the deployment-pinned `FixedValidator`. Everything else - the vault, the
lens, the interfaces - is shared and merges cleanly from upstream.

## Fork-owned paths

Upstream must never (re)create these paths; the fork owns them outright:

- `src/WeightedValidatorRegistry.sol` - the full registry (contract name is
  still `ValidatorRegistry`: per-subnet hotkey sets and BPS weights, EIP-712
  threshold attestations, AccessControl admin). Imports `MAX_VALIDATORS`
  from `IValidatorRegistry.sol` instead of defining it.
- `test/WeightedValidatorRegistry.t.sol`, `test/WeightedValidatorRegistry.gas.t.sol`
- `test/helpers/WeightedAttestationHelper.sol`
- `docs/attester-guide.md`
- `scripts/get_validator_updates.py`
- `snapshots/ValidatorRegistry.json`
- `e2e/` - the whole localnet suite (upstream dropped it; its scenarios need
  weighted sets and mid-test rotations)
- `.github/workflows/e2e.yml`

## Fork-side edits of shared files

Kept as small as possible; each is a known merge-conflict surface:

- `.github/workflows/test.yml`: the Python setup + e2e chainless unit-test
  steps are restored here.
- `.gas-snapshot`: carries the WeightedValidatorRegistry entries on top of
  upstream's. On merge conflicts take upstream's version, then regenerate
  cold: `forge clean && FOUNDRY_PROFILE=ci forge snapshot --no-match-contract
  Invariant --threads 4`.

## Known doc drift

`docs/overview.md` and `docs/security-model.md` describe upstream's
`FixedValidator` wiring. This fork's deployments wire
`WeightedValidatorRegistry` instead - see `docs/attester-guide.md`. The
shared docs are left untouched to keep merges clean.

## Merge routine (upstream -> fork)

1. `git merge upstream/main`
2. Resolve `.gas-snapshot` / `test.yml` if touched (see above).
3. `forge clean && FOUNDRY_PROFILE=ci forge snapshot --no-match-contract Invariant --threads 4`
4. Run the e2e suite against the localnet before releasing.
