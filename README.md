# @slnknrr/id
> Only the README.md is here. I publish the code on Codeberg. [view →](https://codeberg.org/slnknrr/id_lib.js-for-pm_npm).

ULID and UUID generators (v4, v7, v8), shipped as callable instances. They offer a combination the standard ID schemes lack: **an outsider can read nothing from an ID beyond what its format exposes, while you can recover its creation time and sequence number from it.** Uniqueness comes from the construction, not from a random source.

- **permutation** (`new ulid({ crypto: false })`, `uuid8`): the pair *(millisecond, counter)* goes through a seeded 53-bit bijection. `decode()` restores both parts.
- **crypto** (`ulid`, `uuid4`, `uuid7`): the standard CSPRNG layouts (ULID spec, RFC 9562). `uuid7` is monotonic by default.

Pure ESM, zero dependencies, TypeScript declarations included, runs in any modern JavaScript runtime.

## Install

```sh
npm i @slnknrr/id
```

## Quick start

```js
import { ulid, uuid, uuid4, uuid7, uuid8 } from '@slnknrr/id';

ulid();  // '01M2T18M9G2P10EWC2NS2JGJK3'            ULID, 80 CSPRNG bits
uuid();  // 'd6d43e00-b04e-491c-94dc-d93f90d0c7a7'  alias of uuid4
uuid7(); // '01a0b44a-5a9d-7582-a14b-20a76434926f'  RFC 9562 v7, monotonic
uuid8(); // '7b626d37-ad57-8ffe-8a9f-e90d9a8961c4'  permutation, decode() restores it

const ids = new ulid({ crypto: false }); // permutation ULID
const id = ids();                        // '01M2T18M9DBN1CF6TE86MJWTDY'

const values = ids.decode(id); // { time: 1789727625517, increment: 1 }
ids.encode(values);            // '01M2T18M9DBN1CF6TE86MJWTDY'

const raw = ids.parse(id);     // Uint8Array(16), e.g. for a BINARY(16) or UUID column
ids.stringify(raw);            // '01M2T18M9DBN1CF6TE86MJWTDY'
```

## Why a permutation instead of a CSPRNG

Most ID schemes force a choice:

- **Informative IDs**, such as Snowflake, carry the time, the machine and a sequence number. The issuer benefits, but anyone else can read them too.
- **Opaque IDs**, such as UUIDv4, reveal nothing. They also tell the issuer nothing, so every question about an ID turns into a database lookup.

The permutation generators (`new ulid({ crypto: false })`, `uuid8`) provide both properties at once:

- **Opaque to observers.** `uuid8` exposes nothing. A permutation ULID exposes only the millisecond, as every ULID does.
- **Readable by the issuer.** `decode()` restores the creation time and the position within that millisecond. It also rejects IDs that this generator did not issue.
- **Unique by construction.** Within one generator, `(time, increment)` never repeats, and a bijection keeps distinct inputs distinct. You never have to estimate collision probabilities.
- **No random source.** It is plain integer arithmetic, deterministic for a fixed seed.

### How it works

`(time, increment)` → seeded Feistel network over 53 bits → standard ULID or UUIDv8 layout, with a keyed check in the spare bits.

- A Feistel network is a bijection for any round function, which is where uniqueness comes from.
- The same seed runs the network backwards for `decode()`, and forwards again for `encode()`.

This is an established construction, not a workaround. Format-preserving encryption schemes such as NIST FF1 are Feistel networks over a fixed domain. The PostgreSQL wiki publishes the same idea as `pseudo_encrypt`, which turns sequence numbers into non-sequential unique values.

The difference is deliberate. Here the round function is a fast non-cryptographic mixer and the key is 32 bits, so this is **obfuscation, not encryption**. See "When not to use it" below.

## Comparison

| Scheme | Format | Sorts by time | Issuer can recover | Observer can read | Uniqueness | Random source |
|---|---|---|---|---|---|---|
| UUIDv4 | UUID | no | — | — | probabilistic, 122 bits | CSPRNG |
| UUIDv7 | UUID | yes | time | time; with a monotonic counter, issue order within a ms | probabilistic, plus a counter within one generator | CSPRNG |
| ULID | ULID | yes | time | time; in monotonic mode, issue order within a ms | probabilistic, plus an increment in monotonic mode | CSPRNG |
| Snowflake | 64-bit integer | yes | time, worker, sequence | time, worker, sequence | by construction; needs a unique worker ID per process | none |
| Sqids, Hashids | short string | no | the source integers | nothing without the alphabet¹ | inherited from the source integers | none |
| **`new ulid({ crypto: false })`** | ULID | yes, by ms | time, increment | time | by construction, per generator | none |
| **`uuid8`** | UUIDv8 | no | time, increment | nothing without the seed¹ | by construction, per generator | none |

¹ This is obfuscation: the alphabet or the 32-bit seed is a key that can be recovered, not a cryptographic secret.

## When the permutation pays off

Each case below replaces something you would otherwise have to build, store or query.

- **Public IDs that do not leak volume.** From sequential or counter-based IDs, anyone can estimate how many objects you create and how fast; this is the German tank problem. The usual fix is a second, random public ID next to the internal key, plus an index to map one to the other. `uuid8` serves as both: one column and one index, and the issuer can still read the time and order from it.
- **Creation time without a lookup.** Logging, auditing, expiry checks, rate limiting and cache policies can take the creation time from the ID itself with `gen.decode(id).time`. With UUIDv4 that costs a database read. UUIDv7 and ULID expose the time, but to everyone. `uuid8` keeps it private.
- **Cheap rejection at the edge.** `decode()` checks a keyed digest in-process, before anything reaches storage. Malformed, corrupted and foreign IDs from scanners, typos and stale links return `null` without a query. The check is 27 bits for ULID and 14 bits for UUIDv8. It filters noise; it does not stop a determined attacker.
- **Exact order, kept private.** In public, permutation ULIDs sort by millisecond. Internally, `(time, increment)` gives the exact issue order within the millisecond. Observers cannot see that order or the size of a burst.
- **Deterministic IDs.** A fixed seed and a controlled clock give the same IDs on every run. This helps with snapshot tests, fixtures, event replay, simulations and execution environments that must be replayable, and there is no random source to mock. `encode()` goes one step further: it builds the ID for a `(time, increment)` you choose, with no clock involved at all.
- **No dependency on Web Crypto.** The permutation generators work in embedded engines and restricted sandboxes that lack `crypto.getRandomValues`.
- **Provable uniqueness.** For a single writer, uniqueness follows from the bijection rather than from a birthday-bound estimate. UUIDv4 collisions are not a practical risk. The difference matters where uniqueness has to be argued rather than estimated, for example in an audit or a formal specification.
- **Legacy consumers.** `uuid8` fixes the variant nibble to `8` for systems that accept only `8xxx`, and it still carries a recoverable time.

## When not to use it

- **Secrets.** Do not use it for tokens, session IDs, password-reset links or invite codes. The seed is only 32 bits: given a few IDs, an exhaustive search recovers it in seconds on a single CPU core. Use `uuid4()` or `ulid()` instead.
- **Several writers without assigned seeds.** Two generators with the same seed produce identical IDs within the same millisecond, so give each process its own seed. Snowflake needs the same coordination for its worker IDs, but here the space is 32 bits instead of 10.
- **Time-range scans on `uuid8`.** It does not sort by time. Keep an indexed timestamp, or use the permutation ULID, which sorts by millisecond.
- **Third parties that must parse the IDs.** They would need your seed. Use plain UUIDv7 or ULID instead.

## Generators

| Export | Format | Source | `decode()` payload | Constructor argument |
|---|---|---|---|---|
| `ulid` | ULID | CSPRNG by default, permutation on request | `{ time, randomness }`, or `{ time, increment }` | `UINT32` \| `{ crypto, permutation }` |
| `uuid8` | UUIDv8, variant `8` | always permutation | `{ time, increment }` | `UINT32` \| `{ permutation }` |
| `uuid7` | UUIDv7, RFC 9562 | CSPRNG, monotonic by default | `{ time, counter, randomness }` | `{ monotony }` |
| `uuid4`, `uuid` | UUIDv4, RFC 9562 | CSPRNG | none: 122 random bits | none |

`uuid` is the same instance as `uuid4` under a second name. Every generator also has `stringify()` and `parse()` for the 16-byte binary form; see "Methods" below.

## Constructor syntax

One rule everywhere: **a bare argument is the permutation seed (`UINT32`, `0 … 2^32 − 1`), and every other option goes inside an object, even when it is the only one.** There is no boolean argument, so `new ulid(1)` and `new ulid(true)` can never be confused for one another: the first is seed 1, the second is a `TypeError`.

```js
new ulid();                                   // crypto (default)
new ulid({ crypto: true });                   // crypto
new ulid({ crypto: false });                  // permutation, fresh seed
new ulid(42);                                 // permutation, seed 42
new ulid({ permutation: 42 });                // a seed implies crypto: false
new ulid({ crypto: false, permutation: 42 }); // explicit form
new ulid({ crypto: true, permutation: 42 });  // TypeError: contradiction
new ulid(true);                               // TypeError: a boolean is not an option

new uuid8();                                  // fresh seed
new uuid8(42);                                // seed 42
new uuid8({ permutation: 42 });               // same
new uuid8({ crypto: false });                 // TypeError: uuid8 is always a permutation

new uuid7();                                  // monotonic (default)
new uuid7({ monotony: true });                // same
new uuid7({ monotony: false });               // 74 random bits, no order inside one ms
new uuid7(1);                                 // TypeError: uuid7 has no numeric option

new uuid4();                                  // no options: the standard fixes the layout
```

Only `ulid` accepts `crypto`. `uuid8` is always a permutation, and `uuid7`/`uuid4` always use a CSPRNG. Only `uuid7` accepts `monotony`. A wrong type or an unknown key throws `TypeError`, and a seed out of range throws `RangeError`, both at construction time.

## Methods

Every generator carries two pairs of inverses, one for the wire format and one for the payload.

| Method | Available on | Converts | Bad input |
|---|---|---|---|
| `stringify(bytes)` | every generator | `Uint8Array(16)` → string | `TypeError` |
| `parse(id)` | every generator | string → `Uint8Array(16)` | `null` |
| `encode(values)` | `ulid`, `uuid7`, `uuid8` | payload → string | `TypeError`, `RangeError` |
| `decode(id)` | `ulid`, `uuid7`, `uuid8` | string → payload | `null` |

`stringify` and `parse` carry the same meaning as in the `uuid` package, so the names transfer.

### The 16-byte form: `stringify` / `parse`

- Both handle the format only. They check the length, the alphabet, the dashes and the ULID time range, but not the version or the variant, so `uuid4.parse()` accepts any UUID.
- The bytes are big-endian. For ULID they follow the binary layout of the spec: 48-bit time, then 80 bits. The 16 bytes fit a `BINARY(16)`, `bytea` or native `UUID` column and match the binary form of other ULID and UUID libraries. A Node.js `Buffer` is accepted as input.
- Neither uses `this`, so both detach: `ids.map(uuid.parse)`.

### The payload: `encode` / `decode`

`decode` returns what an ID actually carries, and `encode` is its inverse: `gen.encode(gen.decode(id)) === id` for every ID that generator issued.

| Generator | Payload |
|---|---|
| `ulid`, CSPRNG | `{ time, randomness }`, where `randomness` is a copy of the 10 random bytes |
| `ulid`, permutation | `{ time, increment }`, or `null` if the digest does not match the seed |
| `uuid7` | `{ time, counter, randomness }`, where `randomness` is the 4-byte tail |
| `uuid8` | `{ time, increment }`, or `null` unless both eights and the digest are in place |
| `uuid4` | none: 122 random bits carry nothing to recover, so the pair is absent |

- `time` is in Unix milliseconds, so `new Date(time)` works.
- On `ulid` and `uuid8` the payload means something only under the seed that issued it, so `encode` and `decode` need their own generator as `this`. A detached or foreign call throws `TypeError` instead of returning garbage. `uuid7` follows the RFC layout alone, so its pair detaches like `stringify` and `parse`.
- `encode` accepts only the fields its generator carries. A `{ time, increment }` handed to a CSPRNG `ulid`, or an extra key anywhere, throws `TypeError`; a field out of range throws `RangeError`.

### Which input throws and which returns `null`

Strings arrive from outside, so `parse` and `decode` return `null` for anything malformed. Bytes, values and options come from your own code, so a wrong shape there is a bug and throws.

## TypeScript

The declarations ship with the package, so there is nothing to install from `@types` and nothing to configure:

```ts
import { ulid, uuid7, uuid8 } from '@slnknrr/id';
import type { PermutationValues, Uuid7Values } from '@slnknrr/id';

ulid.decode(id);                        // { time, randomness } | null
new ulid(42).decode(id);                // { time, increment } | null
new ulid({ crypto: false }).decode(id); // { time, increment } | null
uuid7.decode(id);                       // { time, counter, randomness } | null

new ulid(true);  // error: a boolean is not an option
uuid4.decode;    // error: uuid4 has no payload
```

The ULID constructor is overloaded, so a seed or `{ crypto: false }` yields a `PermutationUlid` and everything else a `CryptoUlid`, and the payload type follows from the generator rather than from a cast. When the options are only known at runtime (`new ulid(fromConfig)`), the result is the `Ulid` union and you narrow it with `'increment' in values`.

Exported types: `Ulid`, `CryptoUlid`, `PermutationUlid`, `Uuid8`, `Uuid7`, `Uuid4`, `UlidOptions`, `Uuid8Options`, `Uuid7Options`, `UlidValues`, `UlidRandomValues`, `PermutationValues`, `Uuid7Values`.

`index.d.ts` is written by hand rather than emitted: a generator is a callable instance that is also its own constructor, and no declaration emitter infers that shape from the source. `npm run types` type-checks the declarations, and `npm publish` runs it together with the test suite.

## Metaprogramming and DX

**An instance is a function.** The constructor returns the generator function itself, with its prototype set to the class prototype. `typeof ulid === 'function'`, `ulid instanceof Function`, and `call` and `bind` work. You can pass a generator anywhere a callback is expected:

```js
Array.from({ length: 3 }, ulid); // three ids
```

**Calling `new` on an instance creates a new generator.** Inside the generator, `new.target` tells a call apart from a construction. `ulid()` returns an ID. `new ulid(options)` returns a fresh generator of the same class, with its own clock and counter. You import only the instances, because each exported instance is also its own constructor.

```js
const a = new ulid({ crypto: false });
const b = new a();             // new generator, default options again (crypto); options are not inherited
b instanceof ulid.constructor; // true
```

**What an instance can do shows in its shape.** Every generator has `stringify()` and `parse()`. The payload pair exists only where there is a payload:

```js
'encode' in ulid;  // true: time + randomness, or time + increment
'encode' in uuid7; // true: time + counter + randomness
'encode' in uuid8; // true: time + increment
'encode' in uuid4; // false: 122 random bits, nothing to recover
```

Each method is one function per format, shared by every instance. `stringify`, `parse` and the `uuid7` payload pair work on the format alone and do not use `this`, so you can detach them. `encode` and `decode` on `ulid` and `uuid8` find their instance through `this`. Their state (the permutation keys, or the mark of a CSPRNG instance) lives in a module-private `WeakMap`, so no outside code can read it.

**Fail fast.** Options that contradict each other, have the wrong type or use an unknown key throw at construction, not on the millionth ID. A typo such as `{ permutaton: 42 }` cannot quietly fall back to the defaults.

**Lazy crypto.** Importing the module never touches `crypto`. Permutation generators run where `crypto` is missing. CSPRNG generators throw `TypeError` there on their first call.

**Tree-shaking.** The exported instances are constructed at module level, and each one is marked `/*#__PURE__*/`, so a bundler drops the generators an application never imports.

## Payloads

### ULID, crypto (default)

Follows the ULID spec: 48-bit Unix time in milliseconds, then 80 CSPRNG bits, written as 26 Crockford base32 characters. `decode()` returns `{ time, randomness }`; `encode()` puts the same 10 bytes back.

### ULID, permutation

```
┌──────────────────┬─────────────────────┬────────────────────────────┐
│ time, ms (48)    │ digest (27)         │ permute(increment) (53)    │
└──────────────────┴─────────────────────┴────────────────────────────┘
```

- **time** is stored as is. The string sorts by millisecond like any ULID, and standard ULID parsers can read the time.
- **increment** counts 1, 2, 3 … within one millisecond and is permuted. This hides the order within the millisecond, and `decode()` restores it. As a result, sorting is limited: strings sort by millisecond, and within one millisecond you sort by the decoded increment.
- **digest** is a seeded 27-bit check of the 53 bits. `decode()` rejects IDs that are corrupted or were issued under another seed.

### UUIDv8

```
permute(time) 53 │ permute(increment) 53 │ digest 14          = 120 bits
                 + version 8 (4 bits) + variant nibble 8 (4 bits) = 128 bits

xxxxxxxx-xxxx-8xxx-8xxx-xxxxxxxxxxxx
```

- The variant nibble is fixed to `8` for legacy systems that accept only `8xxx`. That costs 2 bits, so the digest is 14 bits.
- The time is permuted too, so no order is visible at all. `decode()` restores both the time and the increment.

### UUIDv7

Follows RFC 9562. By default it is monotonic (RFC 9562 §6.2, method 1):

```
time, ms 48 │ version 7 │ counter 42 (rand_a 12 + rand_b 30) │ variant 10 │ CSPRNG 32
```

- In each new millisecond, the counter starts at a random value with its top bit clear. The start cannot be guessed, and there are 2^41 steps of headroom.
- Within the millisecond the counter grows by 1, so within one generator the string order equals the issue order. The random tail is fresh for every ID.
- The clock is clamped in the same way as in the permutation generators. If the counter runs out (2^41 IDs in one millisecond), the time moves one millisecond ahead, which the RFC allows.

```js
uuid7(); // '01a0b44a-5a9d-7582-a14b-20a76434926f'
uuid7(); // '01a0b44a-5a9d-7582-a14b-20a80fe084f7'  same ms: counter + 1, new random tail

uuid7.decode('01a0b44a-5a9d-7582-a14b-20a80fe084f7');
// { time: 1789731166877, counter: 1514534543528, randomness: Uint8Array(4) [ 15, 224, 132, 247 ] }
```

`decode()` splits the 74 bits the RFC leaves to the generator into the 42-bit `counter` and the 4-byte `randomness`, exactly as the monotonic generator fills them, and reports the version and variant as `null` when they are not `7` and `10xx`. With `{ monotony: false }` all 74 bits are random, so `counter` is simply their top 42 — the split is the layout, not a claim about the contents.

The trade-off of the default: within one millisecond, the order of neighbouring IDs can be predicted, while the 32-bit tail stays random. If even the order must not be visible, use `new uuid7({ monotony: false })`: a 48-bit millisecond timestamp plus 74 CSPRNG bits.

### UUIDv4

122 CSPRNG bits, per RFC 9562. The variant of v4 and v7 is `10xx` (`8`, `9`, `a`, `b`). There is no payload pair: the bits carry nothing but randomness, and `stringify()`/`parse()` already give you all of them.

## Guarantees and limits

- **Clock rollback.** Permutation generators and monotonic `uuid7` clamp the time to the last issued value while the counter keeps counting. If NTP moves the clock back, no ID repeats and the order holds.
- **Uniqueness holds within one generator.** Two generators with the same seed produce the same IDs in the same millisecond. Default seeds never repeat within one JS realm (a Weyl sequence over 2^32), and clock jitter spreads them across processes. If IDs from several processes or hosts end up in one store, give each process its own seed.
- **`decode()` needs the seed that issued the ID.** A default seed changes when the process restarts. To decode IDs later, fix the seed: `new ulid(42)`, `new uuid8(42)`.
- **`decode()` returns `null`** for malformed input, corrupted IDs and IDs from another seed. The ULID check is 27 bits. The UUIDv8 check is 14 bits, so about 1 foreign ID in 16 384 still passes.
- **`encode()` trusts you.** It rebuilds an ID from any `(time, increment)` in range, including one a clock would never produce, so uniqueness across encoded IDs is yours to keep.
- **Input is case-insensitive.** `time` is in Unix milliseconds: `new Date(time)`.
- ULID time runs out at 2^48 ms (year 10889).

## Performance

- Output strings are built flat with `String.fromCharCode`, so there is nothing to flatten later when an ID becomes a key or goes into JSON.
- CSPRNG words come from a pooled `getRandomValues`.
- Permutation rounds stay in int32 arithmetic.
- Round keys are derived once per generator.

Rough numbers from one machine (Node 24), per ID, including using the string:

| | ns / id |
|---|---:|
| `ulid()` | 90 |
| `new ulid({ crypto: false })()` | 122 |
| `uuid4()` | 61 |
| `uuid7()` | 108 |
| `new uuid7({ monotony: false })()` | 107 |
| `uuid8()` | 153 |
| `crypto.randomUUID()`, for reference | 281 |

## Requirements

- ES2020 or later, ESM only. On Node.js: 20 or later.
- `crypto.getRandomValues` for `ulid` (crypto), `uuid4` and `uuid7`.
- TypeScript is optional; the bundled declarations are checked against TypeScript 5.4 and later.

## Tests

```sh
npm test
```

The tests use `node:test` and have no dependencies. They check:

- the exact output bits against a reference written separately from the code (the original loop-form Feistel, BigInt packing, and the RFC 9562 v7 fields read straight off the 128 bits), so any change to the data fails a test;
- uniqueness, clock rollback and the order of monotonic `uuid7`;
- `stringify()` and `parse()` against the reference binary layout;
- `encode()` as the inverse of `decode()`, on issued IDs and on values no clock would produce;
- `decode()`, including foreign seeds, corrupted input, both eights of `uuid8` and the version and variant of `uuid7`;
- every constructor form, including the ones that must throw;
- a runtime without `crypto`.

Type declarations are checked separately:

```sh
npm run types
```

## Links

- Source: [https://codeberg.org/slnknrr/id_lib.js-for-pm_npm/src/branch/main](https://codeberg.org/slnknrr/id_lib.js-for-pm_npm/src/branch/main)
- npm: [https://www.npmjs.com/package/@slnknrr/id](https://www.npmjs.com/package/@slnknrr/id)

## Author

Yury Slinkin (Юрий Слинкин)

- Email: slnknrr.dev@gmail.com
- Codeberg: https://codeberg.org/slnknrr

## License

MIT. See [LICENSE.md](https://codeberg.org/slnknrr/id_lib.js-for-pm_npm/src/branch/main/LICENSE.md).
