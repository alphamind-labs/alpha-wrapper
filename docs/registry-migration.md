# Registry migration and coverage inventory

This cleanup follows [TAO20 PR #86](https://github.com/alphamind-labs/tao20-contract/pull/86),
merged at `c1f85933b88f0a6d1fe24cbecd6139f6cf33d9e5`. TAO20 owns the production
`AttestedValidatorRegistry`, its signing helpers and its real registry integration
suite. Alpha-wrapper keeps `BasicValidatorRegistry` and `IValidatorRegistry`,
including the interface's 64-validator bound. The dependency remains one-way.

## Solidity

Against alpha-wrapper base `7dd8096bfc493dc6689f92a5ac462cd5436ce7ba`, all **585**
remaining test and invariant functions across **26** files retain their names and
scenarios. The only test-body fixture change is the unconfigured-registry test,
which now deploys an empty mock. Other vault assertions remain unchanged.

`AlphaVaultTestBase` installs a controllable `MockValidatorRegistry`. Its normal
setter enforces valid weighted sets, reads owner records when updating, stores
owner snapshots and increments only the updated subnet's nonce. Reads never
refresh ownership. `setRaw` remains a separate malformed-response path for vault
defensive tests. Fifteen additional mock unit/fuzz tests check this fixture contract.

Real `BasicValidatorRegistry` unit, gas and vault integration tests stay here,
including rotation, exits, two-step ownership and parking release. All three
invariant campaigns retain their handlers and assertions and use the mock through
the common fixture.

| Retained Solidity file | Existing test/invariant functions |
| --- | ---: |
| `test/AlphaAccountingInvariant.t.sol` | 2 |
| `test/AlphaVault.gas.t.sol` | 26 |
| `test/AlphaVault.t.sol` | 166 |
| `test/AlphaVaultLens.t.sol` | 21 |
| `test/AlphaVaultPublicProperties.t.sol` | 4 |
| `test/AlphaVaultRounding.t.sol` | 6 |
| `test/BackingInvariant.t.sol` | 4 |
| `test/BackingRecord.t.sol` | 30 |
| `test/BackingRecovery.t.sol` | 54 |
| `test/BackingResolution.t.sol` | 18 |
| `test/BasicValidatorRegistry.gas.t.sol` | 5 |
| `test/BasicValidatorRegistry.t.sol` | 21 |
| `test/BasicValidatorRegistryVault.t.sol` | 6 |
| `test/ClaimableTao.t.sol` | 23 |
| `test/ClaimableTaoInvariant.t.sol` | 4 |
| `test/ClaimableTaoRoundingRegression.t.sol` | 1 |
| `test/CloneBase.t.sol` | 3 |
| `test/DynamicValidatorSet.t.sol` | 12 |
| `test/LockedAlphaDeposit.t.sol` | 17 |
| `test/MinStakeFloor.t.sol` | 28 |
| `test/MockStaking.t.sol` | 9 |
| `test/ReclaimMailboxAlphaAsTao.t.sol` | 12 |
| `test/RecoveryDeadline.t.sol` | 21 |
| `test/RecoveryDust.t.sol` | 11 |
| `test/RollerConsolidation.t.sol` | 16 |
| `test/UnwrapForTao.t.sol` | 65 |

The removed `ValidatorRegistry.t.sol` (79 cases) and `ValidatorRegistry.gas.t.sol`
(eight cases) were checked against TAO20's renamed suites before removal. Signature
validation, quorum, signer administration, replay and batch behavior cannot be
meaningfully tested against a mock. Their implementation and tests live together
in [TAO20's attested tests](https://github.com/alphamind-labs/tao20-contract/tree/c1f85933b88f0a6d1fe24cbecd6139f6cf33d9e5/test/attested).

Gas snapshots now include mock-backed vault calls. Differences from the old
fixture are not production gas optimizations; real precompile measurements come
from e2e receipts. Snapshot regeneration runs the unit/fuzz suite once with the CI
profile, seed `0x1` and four threads, excludes invariant contracts, and removes
sampled fuzz entries from `.gas-snapshot` to match this repository's deterministic
snapshot policy. CI runs fuzz tests and invariants separately.

## E2E

Every retained e2e case uses the real Basic registry, vault and Subtensor
precompiles. No mock is deployed. The 13 retained scenario files contain 15 cases;
173 scenario assertions remain. The unused gas-baseline assertion for the removed
weighted-rebalance leg was dropped; the shared transaction helper still checks
that every receipt contains gas usage. The ownerless-hotkey case
was renamed from `test_holder_exits_after_attesters_replace_the_ownerless_name` to
`test_holder_exits_after_owner_replaces_the_ownerless_name`.

| Retained Basic scenario file | Cases |
| --- | ---: |
| `test_claimable_tao.py` | 1 |
| `test_convicted_alpha.py` | 1 |
| `test_dust_dos.py` | 3 |
| `test_full_flow.py` | 1 |
| `test_locked_deposit.py` | 1 |
| `test_min_stake_floor.py` | 1 |
| `test_min_stake_liveness.py` | 1 |
| `test_parked_recovery.py` | 1 |
| `test_parked_stake.py` | 1 |
| `test_parking_isolation.py` | 1 |
| `test_subnet_dissolved.py` | 1 |
| `test_subnet_generation.py` | 1 |
| `test_transfers_off.py` | 1 |

The five multi-validator scenarios and their original assertions are present in
[TAO20's merged e2e suite](https://github.com/alphamind-labs/tao20-contract/tree/c1f85933b88f0a6d1fe24cbecd6139f6cf33d9e5/e2e/tests/attested_vault):
concurrent swaps, shared recovery deadlines, recovery dust, hostile dust on a
recorded slot, and a refused sale slot alongside live backing. See the
[e2e compatibility table](../e2e/README.md#registry-and-scenario-ownership) for
why Basic cannot reproduce each setup. The weighted third leg of the
minimum-stake-floor scenario remains there too; its Basic deposit and dust
consolidation legs stay here. Generic Solidity equivalents remain upstream.

## Final downstream step

After this cleanup merges, TAO20 must advance its pinned alpha-wrapper submodule
and validate its local attested registry against the cleaned dependency. It must
not import alpha-wrapper test helpers or the removed concrete registry.
