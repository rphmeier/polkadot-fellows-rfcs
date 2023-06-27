Feature Name: blockspace-regions
Start Date: June 27, 2023
RFC PR: https://github.com/polkadot-fellows/rfcs/pull/0001

# Summary

Blockspace regions are a dynamic, multi-purpose mechanism for determining the assignment of parachains to a relay chain's Execution Cores.

Each blockspace region is a data structure which indicates future rights to create blocks for an assigned parachain and keeps records of how many blocks have already been created. Regions can be split, transferred, and reassigned as a way of trading rights to blockspace in secondary markets. Regions are initially created by core relay-chain logic, which is beyond scope for this RFC. Parachains can own an arbitrary number of regions and are limited in block production only by the number and frequency of regions which they own. Each region belongs to a specific execution core, and cannot move between cores. Regions are referenceable by a unique 256-bit identifier within the relay-chain.

# Motivation

Polkadot must go beyond the one-core-per-chain paradigm in order to maximize allocative efficiency of the primary resource it creates: secure blockspace. Regions as a primitive are designed to accommodate a wide variety of mechanisms for allocating blockspace to processes running on Polkadot. Regions are flexible, in that they can be split and transferred between consumers to enable secondary blockspace markets. Regions work across a wide variety of different durations and frequencies, ranging from long-term guaranteed blockspace to even single-block regions.

Demand for applications is highly volatile. It is quite common for applications to suddenly experience a large influx of traffic and interest which lasts for a short time. Accommodating the burstiness of demand is one of the primary motivations for the region primitive. Applications will be able to acquire an arbitrary number of blockspace regions as needed to meet demand, bounded only by the limits of their ability to create blocks and the regions available in the market, either via primary or secondary markets. Likewise, during periods of low demand, applications should be able to sell off some proportion of their rights to future blockspace in order to recoup costs. Regions are a core primitive for adaptive demand.

Regions also reduce barriers to further innovation in the core relay-chain blockspace offerings. Once the runtime and node software have been adapted to run on blockspace regions, all future mechanisms for the relay-chain to sell blockspace may be implemented as simple algorithms which create and assign blockspace regions, without requiring any further modifications to node logic. This will give Polkadot's governance a larger toolkit to regulate the supply and granularity of blockspace entering the economy.

# Guide-level Explanation

A region is a data structure outlined below:

```rust
struct Region {
    core: CoreIndex,
    // Relay-chain block number at which this region becomes active.
    start: u32,
    // Duration of the region in relay-chain blocks, i.e. it ends at start + duration
    duration: u32,
    // The maximum number of blocks which can be made in the region.
    // `None` signifies that the maximum should be limited only by the duration and frequency.
    maximum: Option<u32>,
    // The count of blocks which have already been produced under the region.
    count: u32,
    // This value determines the frequency at which the parachain produces blocks.
    // 
    // It is the numerator of a fraction where the denominator is `FREQUENCY_DENOMINATOR`, where
    // the fraction indicates the rate at which this region provides blockspace as compared to the
    // relay-chain's block-rate. That is, a numerator of `FREQUENCY_DENOMINATOR`, i.e. 1/1, means that
    // the region allows blocks to be made every relay chain block. A numerator of `FREQUENCY_DENOMINATOR/2` gives
    // one block every 2 relay-chain blocks. And so on.
    //
    // This value may not be greater than `FREQUENCY_DENOMINATOR`.
    frequency_numerator: u32,
    // The owner of the region. The owner has the right to split, transfer, or reassign the region.
    // By virtue of the account system, this owner can be a multisig, parachain sovereign account, etc.
    owner: AccountId,
    // The assignee of the region is the parachain the region gives the right to create blocks to.
    assignee: ParaId,
    // The number of children this region has. This is initially set to 0 and is incremented every time the
    // region is either split or decomposed. This provides a predictable and unique input to the calculation
    // of child region-IDs.
    child_count: u32,
}
```

Regions each have a 256-bit identifier, which are unique within the relay-chain they are created. When regions are split or decomposed, the identifier of the newly created region is computed as `blake2_256(RegionId, child_count)`, where `child_count` is incremented afterwards. Because child region identifiers are computed deterministically, it is simple to create atomic sequences of transactions that both create and modify new regions. 

```rust
type RegionId = [u8; 32];
```

Regions are either created directly by relay-chain blockspace allocation logic or by users splitting their
owned regions. User-created regions require a fixed `REGION_STORAGE_DEPOSIT` to create. The `FREQUENCY_DENOMINATOR = 5040` is used to bound the computation done in determining whether regions are in surplus. It is sufficiently large as to allow blocks to come fairly infrequently, at a minimum of 8.4 hours. 5040 is deliberately chosen as a [highly composite](https://en.wikipedia.org/wiki/Highly_composite_number) number, with 60 divisors, including all numbers between 1 and 10. This makes it possible to find exact fractions for common desired frequencies such as 1/3, 1/4, 1/5, etc.

Regions will be managed within a "Regions" Pallet which exposes the following `Call`s:

```rust
// Gated on allowed origins, e.g. a `RegionCreator` origin, set by governance.
// Freshly created blockspace regions don't require storage deposits, as this is intended to be
// gated to logic which creates small, bounded numbers of regions.
fn create_region(Origin, Region) -> RegionId;

// Reassign a region to a new parachain. This is gated to `Signed` origins, where the account is the
// owner of the region.
fn reassign_region(Origin, RegionId, ParaId);

// Transfer a region to a new owner. This is gated to `Signed` origins, where the account is the owner
// of the region. The new owner takes over the storage deposit for the region, if any.
fn transfer_region(Origin, RegionId, AccountId);

// Split a region into two.
//
// This is gated to `Signed` origins, where the account is the owner of the region. For splitting a region, the owner must have `REGION_STORAGE_DEPOSIT` available
// to reserve as a deposit for the newly-created region.
//
// Splits are only valid if 
//   - the input region is non-expired.
//   - the `BlockNumber` is in the future and before the end block of the region.
//   - neither region would have a `Some(0)` maximum.
//
// Any remaining blocks in the region (i.e. maximum - count) are divided pro-rata across the
// regions in accordance to their size.
//
// The input region will have its duration adjusted to `at - start`, and the created region will
// have a duration ending at the original end time. The created region's start block will be set
// `at + frequency`.
//
// The input region's `child_count` field will be incremented. This fails if the number of children is at the maximum already.
fn split(Origin, input: RegionId, at: BlockNumber);

// Split a region on the basis of frequency.
// 
// This is gated to `Signed` origins, where the account is the owner of the region. For splitting a region, the owner must have `REGION_STORAGE_DEPOSIT`
// available to reserve as a deposit for the newly-created region.
//
// It must be true that `1 <= new_frequency < frequency_numerator`.
//
// This creates a new region with `frequency_numerator - new_frequency` and the same start, duration, owner, and assignee.
// The `input` region's `frequency_numerator` will be set to `new_frequency`.
// 
// The `count` and `maximum` of the regions are distributed pro-rata across the regions, as closely as possible to the
// ratio of the frequencies.
//
// The input region's `child_count` field will be incremented after this call. This fails if the number of children is at the maximum already.
fn decompose(Origin, input: RegionId, new_frequency: u32);

// Garbage collect an expired region. An expired region is one which either has `start + duration < now` or has 
// `count == maximum`. This can be called by any `Signed` origin. Most of the storage deposit is returned to the
// `owner` of the region, and a portion is returned to the caller of this function. No-op if the region is not
// expired.
fn collect(Origin, RegionId);
```

# Reference-level Explanation

TODO: what happens when a region changes assignee while a block is pending availability?
TODO: what happens when a region is split while a block is pending availability?

# Drawbacks

# Prior Art

# Unresolved Questions

# Future Possibilities
