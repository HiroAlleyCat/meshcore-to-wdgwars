# Deriving a WDGWars `node_id` from a MeshCore node

Reference for anyone writing or maintaining a MeshCore feeder for
[WDGWars](https://wdgwars.pl). Copy it, argue with it, link it. Confirmed against
the server by LOCOSP on 2026-08-10 and updated for the mesh-slot contract of
2026-08-12.

If you maintain a feeder and take one thing from this page: **the id is the first
8 bytes of the node's public key, lowercase hex, 16 characters.**

## The rule

```
short on-air ID   0CE8
full public key   0CE8FE4FB8E5EA2C0A7B1A974DFBD604BA8A63F68DA6A26F146C4BD1CEA1FFE6
node_id           0ce8fe4fb8e5ea2c
```

`node_id = public_key[:16].lower()`

The server gate is `[a-f0-9]{8,16}`, so 16 hex has always been accepted. Nothing
had to change server-side for this form to work.

## Why 8 bytes and not fewer

MeshCore names a node on the air by the leading bytes of its Ed25519 public key,
so the 2 to 6 hex characters a capture prints is a truncation of the same number
rather than a different identifier. Taking more digits of it is not an invention.

LOCOSP measured the collision behaviour on the live corpus of 3,723 nodes:

| Prefix | Result |
|---|---|
| 1 byte | every single node collides with another |
| 2 bytes | 396 nodes collapse into 179 groups. One prefix, `fddd`, covers six distinct nodes |
| 3 bytes | 122 collapse into 57 |
| 4 bytes | first clean one, zero collisions |
| 8 bytes | does not run out at any scale this project will see |

The column is `varchar(16)`, so 8 bytes is also exactly what fits. A full 32-byte
key would not.

Collisions are not cosmetic. **The importer updates a node's position on an id
match**, so two distinct repeaters sharing an id overwrite each other's
coordinates. A wrong identity is worse than a rejected record.

## Why the short ID is not enough

A repeater only puts its full ID on the air in an advert frame, which on default
repeater config can be once in roughly 68 hours. Capture apps realistically log
the short prefix and nothing else, which is why waiting for capture tools to print
longer IDs was never going to work.

That constraint was established by **@formtapez**. The derivation itself came from
**@nicolasrata** in [Heimdall issue #1](https://github.com/Yggdrasil-AI-labs/meshcore-to-wdgwars/issues/1),
who proved it with a proxy that rewrote the ID between MeshMapper and Heimdall.

## Where the key actually is

Not every capture carries one.

| Source | Carries the key? |
|---|---|
| MeshCore offline ping-log JSON (`pings[]`, from meshcore-ha or a MeshCore offline capture) | yes, on key-bearing pings such as `DISC` and `TRACE` |
| MeshMapper "Logs then Copy CSV" export | **no**, so those nodes cannot clear the floor |

Gate on the presence of `public_key` rather than on a ping type label. `TRACE` is
key-bearing too, and type spellings vary between tools.

If a capture has no key for a node anywhere, leave the short ID and report the
predicted rejection. Do not pad it, do not invent digits, do not guess. See the
position-overwrite consequence above.

## Two guard rails worth copying

**Only use a key that actually belongs to that sighting.** Use it when the key
starts with the short ID the capture heard. If it does not, something is wrong and
the sighting should keep its short ID.

**Resolve a short prefix against keys elsewhere in the same capture only when
exactly one node matches it.** This is genuinely useful (3 of 16 nodes in one real
sample) and genuinely dangerous: if a capture logged the key for only one of two
same-prefix nodes, an unguarded lookup attaches the sighting to the wrong node.
Ambiguous stays short. That resolution step was **D3mo's** idea, implemented in
DedDrop.

## Optional fields worth sending

**`public_key`**, the full 64 hex. Optional server-side. When present it is
checked at 64 hex with `node_id` verified as its prefix, rejecting as
`bad_public_key` or `key_prefix_mismatch` otherwise, and its absence never
rejects. Omit the field rather than sending null.

It is not proof the node exists, since anyone can mint a keypair and derive a
matching id. It catches mistakes and lazy fabrication. The reason to send it
anyway: holding full keys lets the server re-derive the canonical id form later
and merge id namespaces deterministically, without asking every feeder to change
again.

**`network`**, `meshcore` or `meshtastic`, as of 2026-08-12. Authoritative when
sent. Omitted, the server infers it from role-name casing, since MeshCore roles
are Title Case and Meshtastic roles are SCREAMING_SNAKE. Any other value is
rejected as `bad_network`. Send it explicitly.

**`path_hops`** where the capture has it. A sighting always counts however it
arrived, but a hopped one will not move a node's position ahead of a sighting that
arrived at least as directly. Absent means the server assumes direct.

## If you are switching a feeder from 4 bytes to 8

You do not need to coordinate with anyone. The importer recognises the same node
across id lengths in both directions, so a feeder changing form does not arrive as
a fresh set of nodes. Before that fix a coordinated switch would have doubled
players' node counts and handed out the 50-node badge for free. Move whenever
suits you.

Prefix merging is **MeshCore only**. A Meshtastic id is a device number rather
than a key prefix, so the same string can legitimately mean two different devices
across the two networks, and the server treats them as two nodes.

## Credits

- **@nicolasrata**, the derivation and the first real-world baseline capture
- **@formtapez**, the advert-frame constraint that made it necessary
- **D3mo** (`d3mocide`), the same-capture prefix resolution, in DedDrop
- **LOCOSP**, the collision measurements, the canonical form, and the
  both-directions importer fix
