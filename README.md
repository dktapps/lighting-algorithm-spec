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

### Pass 2: Propagate light across chunk borders
For this pass, we take square groups of 4 chunks, and propagate light across their shared borders.
No chunk is special; all 4 chunks in each group are treated the same way.

Each chunk belongs to 4 groups of 4 chunks:

- one where it is the top-left chunk
- one where it is the top-right chunk
- one where it is the bottom-left chunk
- one where it is the bottom-right chunk

Once a chunk has been involved in a pass 2 calculation in all of these positions, it is then fully light-populated.

#### Why not groups of 9?
Groups of 4, while harder to explain, are significantly more parallel-friendly, easier to code, and in the case of PocketMine-MP, require significantly fewer chunk copies (4 instead of 9), due to the fact that a pass2 using groups of 4 only needs to lock a chunk 4 times.

#### Example
- A torch is placed in chunk (0,0) at x=14,z=14
- A tunnel crosses into chunk (0,1), turns right, crosses into chunk (1,1), then upwards, and crosses into chunk (1,0).
- The light from the torch must be propagated to all three chunks.

#### Requirements for each part of the pass
- All chunks involved must be available.
- All chunks involved must have had pass 1 completed.

## Notes
- Block and sky light are independent and can be calculated in parallel.
- If pass 1 did nothing for any of the 4 adjacent chunks, pass 2 can be skipped for the current chunk, since it guarantees that none of the adjacent chunks have any border light that can be received by the current chunk.
- It's possible to record whether light reached chunk borders during pass 1, to allow adjacent chunks to quickly decide whether they want to do pass 2 with that chunk or not.
- Pass 0 and 1 can be embarassingly parallel, since they only require a single chunk to execute.
