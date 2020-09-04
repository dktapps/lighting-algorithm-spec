# Minecraft block lighting
## Why do we care about light on the server?
On the server side, light is used for a variety of vanilla logic, including (but not limited to):
- Grass growth (and death)
- Crop growth
- Sapling growth
- Ice melting
- Bamboo growth
- Mob spawning (for example, undead mobs won't usually spawn in bright light)

## So why are we calculating light for every single chunk?
No idea. In theory, we don't need to calculate lighting for anything other than chunks currently in ticking areas (usually a few chunks' radius around the player where crops are expected to grow and so on).

## Algorithm
We propose a 3-pass algorithm for relighting chunks, starting from zero light.

### Special cases
If an arbitrary chunk is replaced, its light must be invalidated, and also the light of the surrounding **8 chunks**. This is because a chunk's lighting may have affected any of these adjacent chunks (due to propagation).

### Pass 0: Discover light nodes
#### Sky light
For sky light, this involves calculating the "height map" of the chunk. For each X/Z column in the chunk, the heightmap contains the Y coordinate of the **highest block that doesn't affect light, NOT the highest block**.

Blocks pointed to by the heightmap should be treated as light sources for sky light propagation.
In addition, account for the difference between the current column and the highest immediately adjacent heightmap, to account for cases where sideways propagation is needed.

| Block | Ignored by heightmap | Reason |
|:------|:--------------------:|:-------|
| Glass | Yes | Fully transparent |
| Air | Yes | Fully transparent |
| Leaves | No | Diffuses sky light (doesn't reduce light propagation, but triggers heightmap) |
| Cobweb | No | Diffuses sky light (doesn't reduce light propagation, but triggers heightmap) |
| Water | No | Light is reduced by 3 levels when propagating through (instead of usual 1) |
| Stone | No | Fully opaque |

#### Block light
For block light, you just need to scan the whole chunk and look for blocks with a non-zero light emission. These blocks are your source nodes.

**Tip:** You can optimise these kinds of scans using the block palette. If a subchunk's block palette doesn't contain any light-emitting blockstates, you can avoid scanning the entire 4096 blocks of that subchunk.

### Pass 1: Propagate light inside the bounds of a single chunk column
This pass **must not** influence any other chunks except the source chunk. It's possible, probably practical, to do pass 0 and 1 directly in series.

#### Requirements
- Pass 0 must have been completed.

### Pass 2: Propagate light from adjacent Pass1 chunks into the current chunk
This pass involves propagating light across chunk borders from adjacent chunks into the current one.

This pass is only necessary if any adjacent chunk has a bordering light level of at least 2.

#### Example
- A torch placed on or near a chunk border

#### Requirements
- Pass 1 must have been completed for the current chunk.
- All of the 4 directly adjacent chunks must be available.
- Pass 1 must have been completed for each of the directly adjacent 4 chunks.

### Pass 3: Propagate light from adjacent Pass2 into the current chunk
This pass propagates light originating from the current chunk back into the current chunk.
This is necessary to fill light in U-shaped structures (for example tunnels) where light originating from one side of the U needs to make it to the other side.

#### Example
- A torch placed in a U-shaped tunnel that sits across a chunk border
