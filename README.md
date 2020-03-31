# Multi-party computation setup ceremony for the Semaphore zero-knowledge gadget

Before the [Semaphore zero-knowledge
gadget](https://github.com/appliedzkp/semaphore) can be production-ready, the
authors and the Ethereum community must perform a multi-party computation setup to
produce a proving key and verifying key. At least one ceremony participant must
securely discard the toxic waste produced during the process in order for the
final result to be trustworthy and secure.

There are two phases to the ceremony: phase 1 and phase 2.

Phase 1 is almost complete. We will use a challenge file (specified below) from
the Perpetual Powers of Tau ceremony, which applies to any Groth16 zk-SNARK
circuit with up to `2 ^ 28` constraints. What remains is to generate a public, unbiased
random number which is to be applied to said challenge file.

We will collaborate with [Supranational](https://www.supranational.net/), a
member of the [VDF Alliance](https://www.vdfalliance.org/), to run a verifiable
delay function (VDF) on an Ethereum (ETH1) mainnet block hash to produce this
number.

The goal is to produce a random number in such a way that an adversary cannot
reasonably affect or bias this value in a way that
favours them. We use an ETH1 mainnet block hash whose block height will be
announced at least a day in advance as its value is sufficiently difficult to
predict by an adversary.

Furthermore, we use a VDF for this reason described by [Bunz et al, page 2](http://www.jbonneau.com/doc/BGB17-IEEESB-proof_of_delay_ethereum.pdf):

> Intuitively, the idea is that miners (or any oher party) cannot determine the beacon result
from a given block before some non-negligible mount of time has elapsed, at which point it is too late to attack as the blockchain has already moved on.

In phase 2, we will use the output of phase 1 as the starting point of a
circuit-specific multi-party computation setup. Like in phase 1, participants in
this phase will take turns to apply a secret random number to the previous
participant's output.

After the final phase 2 participant submits their contribution, we will pick
another block hash at least a day in advance, apply a VDF on it, and use this
final output to produce a proving key and verifying key for the Semaphore
circuit defined in [this repository](https://github.com/appliedzkp/semaphore)
at commit `a652d654ed992a0ace51b5345d4618e8f9be21ea` at
`circuits/circom/semaphore.circom`.

`circuit.json` has SHA256 hash
`3c9e8106555fbe26f5606a5fd3213d436070395910afb82228df388c5296f2c2` and is located at this URL:

```
https://www.dropbox.com/s/3gzxjibqgb6ke13/circuit.json?dl=1
```

Note, however, that `circuit.json` is not deterministically generated, so if
you recompile the circuit, the hash will differ. Its `constraints` attribute,
however, is deterministic, and has the SHA256 hash
`95fa431bdcdc99660e34943dda47134323978feaf41d22791b02a58c9af7896b`. Anyone can
verify the hash of `constraints` using the
[hash_circuit_and_constraints.py](./hash_circuit_and_constraints.py) Python 3
script.

## Ceremony progress

TODO
<!--| Participant ID | Identity | GPG key | Attestation |-->
<!--|-|-|-|-|-->
<!--| 0001 | Koh Wei Jie | [Keybase](https://keybase.io/contactkohweijie) | [0001_weijie_response](./0001_weijie_response/README.md) |-->

## Ceremony process

### 1. The challenge file

We will use a challenge file (`challenge_0025`) from the Perpetual Powers of Tau
ceremony as the starting point. The Blake2b hash of the challenge file we will use,
produced using the `b2sum -b` command, is:

```
ab45d9d9de4a950da97ae2b1d20fb7c61b6d10986cb02d418a8be0c64e2c99d6c000a914c03d9ab271ee07c0990a41fb569714202c380711d722350cc19a1152
```

Its SHA256 hash is:

```
357d502815eed3bc031f19ef48baa358a321624eaa10aba8c6d09ff21290afc2
```

Its URL is:

```
https://ppot.blob.core.windows.net/public/challenge_0025
```

We will stick to the above challenge file even though new ones are available.

### 2. The block hash

We will use the hash of block
[9730000](https://etherscan.io/block/countdown/9730000) on the Ethereum mainnet,
which will be mined around Tue Mar 24 2020 05:14:35 GMT+0800. We chose this particular
block height as:

1. It is at between 1 and 3 days in the future relative to the date that we will announce it.
2. It is rounded to a multiple of 1000.

We then interpret the block hash as a big-endian number, which is used as an
input to the VDF as a decimal number. The block hash can be obtained with a
synced Geth node from the v1.9.12 release (commit hash
`b6f1c8dcc058a936955eb8e5766e2962218924bc`) using `eth.getBlock(9730000).hash`.

### 3. The VDF

We will generate a random value using the [RSA-based verifiable delay
function](https://eprint.iacr.org/2018/623.pdf), with the [RSA-2048
modulus](https://web.archive.org/web/20130507115513/http://www.rsa.com/rsalabs/node.asp?id=2093).

The RSA-2048 modulus value is:

```
25195908475657893494027183240048398571429282126204032027777137836043662020707595556264018525880784406918290641249515082189298559149176184502808489120072844992687392807287776735971418347270261896375014971824691165077613379859095700097330459748808428401797429100642458691817195118746121515172654632282216869987549182422433637259085141865462043576798423387184774447920739934236584823824281198163815010674810451660377306056201619676256133844143603833904414952634432190114657544454178424020924616515723350778707749817125772467962926386356373289912154831438167899885040445364023527381951378636564391212010397122822120720357
```

The VDF uses repeated squaring in the corresponding RSA quotient group, where elements and their negatives are considered the same.

The following is an example of a slow implementation of the VDF in Python:

```python
#!/usr/bin/python3

block_hash = <BLOCK HASH>

modulus = 25195908475657893494027183240048398571429282126204032027777137836043662020707595556264018525880784406918290641249515082189298559149176184502808489120072844992687392807287776735971418347270261896375014971824691165077613379859095700097330459748808428401797429100642458691817195118746121515172654632282216869987549182422433637259085141865462043576798423387184774447920739934236584823824281198163815010674810451660377306056201619676256133844143603833904414952634432190114657544454178424020924616515723350778707749817125772467962926386356373289912154831438167899885040445364023527381951378636564391212010397122822120720357

iterations = 16

result = block_hash
for i in range(iterations):
    result = result ** 2 % modulus

if result > modulus // 2:
    result = modulus - result
    
print(result)
```

We assume the following:

1. An Ethereum block hash is considered final after roughly 6 minutes.
2. The RSA-2048 modulus is not factorizable.
3. The best adversary (e.g. with an RSA ASIC) can perform a squaring in the RSA group no faster than `0.1` nanoseconds.

We will run the VDF for 3600000000000 iterations. This number is derived from
the amount of iterations the best adversary would have to run in order to get
to `6` minutes - `(6 * 60 * 10 ^ 9 ns / (0.1 ns/iteration))`. Since the best attacker
can do squarings in the RSA group no faster than `0.1` nanoseconds, then the
attacker could not have affected the chosen block hash and therefore the random
number is unbiased.

\[Note: The VDF Alliance's RSA-based VDF FPGA implementation runs at about `88.9245`
ns/iteration, therefore the duration it would take it to run is about 5335
minutes.\]

The block hash is:

```
0xb8ba422c143fc4091be420a7702cdd814b6d7de7bba7f19ec4f546b97691194f
```

The above block hash as a big endian decimal integer is:

```
83554654396998814025015691931508621990409003355162694699046114859281714059599
```

We will convert the above block hash from hexadecimal to a decimal using this Python3 snippet:

```python3
print(int('0xb8ba422c143fc4091be420a7702cdd814b6d7de7bba7f19ec4f546b97691194f', 16))
```

The decimal will be fed into the VDF using this snippet:

```
mpz_set_str(
    x_in,
    "83554654396998814025015691931508621990409003355162694699046114859281714059599",
    10
);
```

The output of the VDF is:

```
10977993103982121932239571465640301635762027965778194798910975354072884358350494172672647299765252840966576799178009376323650485178763413504349033403215382586241254899159195313346305695023748302468610424285905583247850388608587325650709157002017034725430299473131263305265832122359834767808351785119640714358834116601625436810622324602730551082077115422457111213950798926963774735333296895627587092632562291756198158559197379553015590673052743673005493881216459740346530146895497101358484528882909331921168487313102020250775017033237065548899535128240671163142253679430742485059665815423506876245572059454779721146064
```

It is the integer `y` resulting from the repeated squarings such that if
`y > N / 2`, we take `N - y` where `N` is the RSA-2048 modulus.

Supranational has generated a proof of the VDF, and anyone can use
[verify_proof.py](./verify_proof.py) to verify it. The proof is located at
[proof.json](./proof.json).

The proof follows the protocol described in
[this paper by Wesolowski 2018](https://eprint.iacr.org/2018/623.pdf).

### 4. SHA256-hashing the final output once

We will only apply one SHA256 hash to the VDF output, interpreted as a
big-endian integer, so that we can get a 32-byte value which the
`beacon_constrained` program requires (see below).

\[Note: In contrast to the previous run, we will *not* apply iterated SHA256
hashes to the output of the VDF. \]

To convert the VDF output, we will use the following Python 3 code, which will
print the hash to the console as a hexadecimal value:

```python3
import hashlib

vdf_output = hex(10977993103982121932239571465640301635762027965778194798910975354072884358350494172672647299765252840966576799178009376323650485178763413504349033403215382586241254899159195313346305695023748302468610424285905583247850388608587325650709157002017034725430299473131263305265832122359834767808351785119640714358834116601625436810622324602730551082077115422457111213950798926963774735333296895627587092632562291756198158559197379553015590673052743673005493881216459740346530146895497101358484528882909331921168487313102020250775017033237065548899535128240671163142253679430742485059665815423506876245572059454779721146064)

m = hashlib.sha256()
m.update(bytes.fromhex(vdf_output[2:]))
sha256_input = m.digest()

print(sha256_input.hex())
```

The result is:

```
65ffc7bbb5bfa63765f0f5f869801498dfc1c182812fd6bdd6b7097b7ce7a059
```

### 5. Applying the public random value to the chosen challenge file of phase 1

Using the `ppot_fix` branch of
[phase2-bn254](https://github.com/kobigurk/phase2-bn254) (commit hash
`52a9479810f583c58156db292c0a3762ee790af7`), we will modify the source code (as
the value is hardcoded):

`powersoftau/src/bin/beacon_constrained.rs`, line 44:

```
let mut cur_hash: [u8; 32] = hex!("65ffc7bbb5bfa63765f0f5f869801498dfc1c182812fd6bdd6b7097b7ce7a059");
```

Next, we will rebuild the binaries, and use the
`beacon_constrained` program with the `challenge_0025` file to produce a `response`.

This `response` file has the Blake2b hash:

```
4acc62c551a677cf82b8590f04799aff8de37b7eef47b353abcb7c4a7896cd69554a35c5185ca7231dd0e1d4a982b226a3b465acd682ca6a7caaa8a98bde5cf2
```

Its URL is:

```
https://ppot.blob.core.windows.net/public/response_semaphore_phase2
```

Its IPFS content hash is:

````
QmPZjPrWN42jQ7xVSibVwQXHEL7YSguFW3eweNYVsevkBh
```

Also using `ppot_fix`, we will run the `prepare_phase2` program to generate
radix files up to `phase1radix2m16`, which supports running a phase2 ceremony
for circuits up to `2 ^ 16` constraints

### 6. Running phase 2

Next, we will initialise the phase 2 ceremony.

- Using the `master` branch of phase2-bn254 (commit hash
  `5c1350358785474df5c47e0431720ddfdfd04ed6`), we will run the `phase2` `new`
  binary to create the first challenge file:

```bash
cargo run --release --bin new circuit.json circom0.params
```

Each participant must run the following for the file `circom<n>.params`:

```bash
cargo run --release --bin contribute circom<n>.params circom<n+1>.params <random entropy>
```

After each participant, the coordinator must run:

```bash
cargo run --release --bin verify_contribution circuit.json circom<n>.params circom<n+1>.params
```

The coodinator must then send `circom<n+1>.params` to the next participant.

After at least 30 contributions, we will stop the ceremony at our own discretion and end up with a final
`final.params` file.

### 7. Applying another public random value to the final contribution to phase 2

We will run the above steps 2 - 4 again with a new, pre-announced block hash (at least a day in
advance, apply a VDF on it, hash it once with SHA256 to derive `<beacon hash>`),
and then apply `<beacon hash>` to the final `.params` file with 0 hash
iterations:

```bash
cargo run --release --bin beacon final.params <beacon hash> 0 final_with_beacon.params
```

### 8. Generate the proving and verifying keys

At the end of the ceremony, we will generate the proving and verifying keys:

```bash
cargo run --release --bin export_keys final_with_beacon.params verification_key.json pk.json

cargo run --release --bin copy_json proving_key.json pk.json transformed_pk.json

mv transformed_pk.json /path/to/semaphore/circuits/build/proving_key.json
```

Note that `proving_key.json` was produced by `snarkjs` before the ceremony, and
only serves as a reference for the copy_json binary.

## Instructions for each participant

First, set up a Linux machine and install Rust and Cargo following instructions [here](https://www.rust-lang.org).

Download and compile the required source code:

```bash
git clone https://github.com/kobigurk/phase2-bn254.git --branch ppot_fix && \
cd phase2-bn254/phase2 && \
cargo build --release
```

Next, get the `circuit<n>.params` file from the coordinator.

In the `phase2` directory, run:

```bash
cargo run --release --bin contribute circom<n>.params circom<n+1>.params <random entropy>
```

Send the `circom<n+1>.params` file to the coordinator.

Please also write an
attestation to your contribution and sign it with a GPG key, Keybase account,
or Ethereum account using the example
[here](https://github.com/weijiekoh/perpetualpowersoftau/blob/master/README.md#your-attestation).

## Logistics

To be confirmed: each participant needs to download a 22M `.params` file and
upload a 22M `.params` file. The computation time should take no more than a
minute on a modern laptop.

For convenience, we will use Dropbox to share the `.params` files. The
coordinator will back them up to Azure blob storage and IPFS.

## Test run (ignore this section)

1. Cloned `phase2-bn254` and switched to the `ppot_fix` branch

2. Ran `b2sum -b challenge` on challenge 0023

```
14e4fe7759391f2c6988e7902abd6343cc27d25980883aec39a88d1e415610b225e0aa5ac1ec9f5a40b699894767ac75983e5ca5441ebbc6ca66a61d049c9112 *../roman/new_challenge
```

3. Ran `beacon_constrained` with `challenge` 0023 in the current working directory.

```
ubuntu@ppotcoordinatorsea:/mnt/disk0/beacon_test$ ../ppot_fix/phase2-bn254/powersoftau/target/release/beacon_constrained
Will contribute a random beacon to accumulator for 2^28 powers of tau
In total will generate up to 536870911 powers                        
0: 0000000000000000000a558a61ddc8ee4e488d647a747fe4dcc362fe2026c620  
1: 0659d0f689b355b876b46cb515fcbadf69506e66b069a85a737d6fcc9958ecce
...
1023: af25c73da6f0fa1de98b7d677026a57513056403260fd80cfe3f4b65c71bf3a0
Final result of beacon: bb8eef164289bf02058b42c4ca2ac72013deb099bc79067e5779c9f1728b1271
Done creating a beacon RNG
Calculating previous contribution hash...
Contributing on top of the hash:
        dfcaaa51 9ada2419 256d73b4 1d944815
        4ff1b687 5874d44c f4e18be8 e933966e
        08db79db 40878856 78138e60 d3c82a9a
        7d50735c 449af38a 76d9ac9c 81af7e2a
Computing and writing your contribution, this could take a while...
Done processing 2097151 powers of tau
...
Done processing 536870910 powers of tau
Finihsing writing your contribution to `./response`...
Done!

Your contribution has been written to `./response`

The BLAKE2b hash of `./response` is:
        52daebb5 4b113656 f7367e29 7d430394
        da749ab9 b6e195da 1d3499ea b63aeacc
        d4e0a252 9ab06ef0 be1c4eb6 d01b3442
        feb12c9d c0523a8c a2331d66 ffb561e8
Thank you for your participation, much appreciated! :)
```

4. Ran `prepare_phase2`.
