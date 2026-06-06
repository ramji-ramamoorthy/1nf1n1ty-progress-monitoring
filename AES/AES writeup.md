# CryptoHack - How AES Works Writeup

**Author:** Ramji R  
**Platform:** Cryptohack    
**Category:** AES                                           
**Date:** June 2026

---

## 1. Keyed Permutations (5 pts)

### Challenge

AES performs a "keyed permutation", mapping every possible input block to a unique output block. The question asks for the mathematical term for a one-to-one correspondence.

### Answer

**Bijection**

A bijection is a function that is both:

* **Injective (one-to-one):** every input maps to a unique output
* **Surjective (onto):** every possible output is mapped to by some input

In AES-128, the keyed permutation is bijective over all 2^128 possible 128-bit blocks. This guarantees that every plaintext maps to exactly one ciphertext, making decryption possible.

**Flag:** `crypto{bijection}`

---

## 2. Resisting Bruteforce (10 pts)

### Challenge

Given the context of AES-128's 128-bit keyspace and known attacks, the question asks for the name of the best single-key attack against AES.

### Answer

**The Biclique Attack**

Key facts:

* Best known single-key attack on AES-128
* Reduces security from 128 bits to **126.1 bits**, only marginally better than brute force
* Published in 2011 by Bogdanov et al.; not significantly improved since
* Still completely infeasible in practice
* Uses biclique structures from graph theory to slightly speed up exhaustive key search

For context on why brute force alone is already infeasible: the entire Bitcoin mining network turned against a single AES-128 key would take over 100 times the age of the universe to crack it.

On the quantum side, Grover's algorithm halves the effective security of symmetric ciphers (128-bit to 64-bit effective), which is why AES-256 is recommended for post-quantum contexts.

**Flag:** `crypto{biclique}`

---

## 3. Structure of AES / Round Keys (15 + 20 pts)

### Challenge

Implement `matrix2bytes` to convert a 4×4 AES state matrix back into a 16-byte plaintext.

### Solution

```python
def bytes2matrix(text):
    return [list(text[i:i+4]) for i in range(0, len(text), 4)]

def matrix2bytes(matrix):
    return bytes(sum(matrix, []))

matrix = [
    [99, 114, 121, 112],
    [116, 111, 123, 105],
    [110, 109, 97, 116],
    [114, 105, 120, 125],
]

print(matrix2bytes(matrix))
```

### Explanation

`sum(matrix, [])` flattens the list of lists into a single list of integers. `bytes()` converts those integers into a byte string, recovering the plaintext.

**Flag:** `crypto{inmatrix}`

---

## 4. AddRoundKey (20 pts)

### Challenge

Implement `add_round_key`, XOR the state matrix with the round key matrix element-wise.

### Solution

```python
def add_round_key(s, k):
    return [[s[i][j] ^ k[i][j] for j in range(4)] for i in range(4)]

print(matrix2bytes(add_round_key(state, round_key)))
```

### Explanation

AddRoundKey is the only step in AES where the key material is mixed into the state. XOR is self-inverse, so `AddRoundKey` and its inverse are identical. Applying it twice with the same key restores the original state. This is what makes AES a keyed permutation rather than just a permutation.

**Flag:** `crypto{r0undk3y}`

---

## 5. Confusion through Substitution - SubBytes (25 pts)

### Challenge

Implement `sub_bytes` using a given S-box lookup table, then pass the state through the inverse S-box to recover the flag.

### Solution

```python
def sub_bytes(s, sbox=s_box):
    return [[sbox[b] for b in row] for row in s]

print(matrix2bytes(sub_bytes(state, sbox=inv_s_box)))
```

### Explanation

SubBytes provides **confusion**, making the relationship between the key and ciphertext as complex as possible. Each byte is used as an index into the S-box, which is a precomputed lookup table derived from:

1. Taking the modular inverse in GF(2^8) (Galois Field)
2. Applying an affine transformation tuned for maximum non-linearity

This makes AES resistant to linear approximation attacks. The inverse S-box simply reverses this substitution.

**Flag:** `crypto{l1n34rly}`

---

## 6. Diffusion through Permutation - ShiftRows + MixColumns (30 pts)

### Challenge

Implement `inv_shift_rows` to reverse the ShiftRows operation, then apply `inv_mix_columns` and `inv_shift_rows` on the given state to recover the flag.

### Solution

```python
def inv_shift_rows(s):
    s[0][1], s[1][1], s[2][1], s[3][1] = s[3][1], s[0][1], s[1][1], s[2][1]
    s[0][2], s[1][2], s[2][2], s[3][2] = s[2][2], s[3][2], s[0][2], s[1][2]
    s[0][3], s[1][3], s[2][3], s[3][3] = s[1][3], s[2][3], s[3][3], s[0][3]

inv_mix_columns(state)
inv_shift_rows(state)
print(matrix2bytes(state))
```

### Explanation

**ShiftRows** provides **diffusion**, ensuring that changes in one byte propagate across the entire state.

The forward `shift_rows` cyclically shifts columns (in row-major notation):

* Column index 1: left shift by 1 -> `[0,1,2,3] -> [1,2,3,0]`
* Column index 2: left shift by 2 -> `[0,1,2,3] -> [2,3,0,1]`
* Column index 3: left shift by 3 -> `[0,1,2,3] -> [3,0,1,2]`

The inverse reverses each shift direction:

* Column index 1: right shift by 1 -> `[0,1,2,3] -> [3,0,1,2]`
* Column index 2: shift by 2 is self-inverse -> `[0,1,2,3] -> [2,3,0,1]`
* Column index 3: right shift by 3 -> `[0,1,2,3] -> [1,2,3,0]`

**MixColumns** performs matrix multiplication in Rijndael's Galois field (GF(2^8)), ensuring every byte in a column affects all other bytes in that column. Together, ShiftRows and MixColumns guarantee that after just two rounds, every byte affects every other byte, achieving the **Avalanche Effect** where a 1-bit change in plaintext statistically flips about 50% of ciphertext bits.

**Flag:** `crypto{d1ffUs3R}`

---

## 7. Bringing It All Together - Full AES-128 Decrypt (50 pts)

### Challenge

Assemble all components into a working `decrypt(key, ciphertext)` function and decrypt the given ciphertext.

### Solution

```python
def decrypt(key, ciphertext):
    round_keys = expand_key(key)

    state = bytes2matrix(ciphertext)

    # Initial AddRoundKey with last round key
    state = add_round_key(state, round_keys[N_ROUNDS])

    # 9 main rounds in reverse order
    for i in range(N_ROUNDS - 1, 0, -1):
        inv_shift_rows(state)
        state = sub_bytes(state, sbox=inv_s_box)
        state = add_round_key(state, round_keys[i])
        inv_mix_columns(state)

    # Final round: no InvMixColumns
    inv_shift_rows(state)
    state = sub_bytes(state, sbox=inv_s_box)
    state = add_round_key(state, round_keys[0])

    return matrix2bytes(state)

print(decrypt(key, ciphertext))
```

### Explanation

AES decryption is the exact reverse of encryption, using inverse operations and round keys in reverse order:

| Encryption Order   | Decryption Order     |
| ------------------ | -------------------- |
| AddRoundKey (RK0)  | AddRoundKey (RK10)   |
| SubBytes           | InvShiftRows         |
| ShiftRows          | InvSubBytes          |
| MixColumns         | AddRoundKey (RKi)    |
| AddRoundKey (RKi)  | InvMixColumns        |
| ...                | ...                  |
| SubBytes (final)   | InvShiftRows (final) |
| ShiftRows (final)  | InvSubBytes (final)  |
| AddRoundKey (RK10) | AddRoundKey (RK0)    |

The final round skips `InvMixColumns`, mirroring how encryption's final round skips `MixColumns`. This asymmetry ensures the encrypt and decrypt structures remain consistent.

**Flag:** `crypto{MYAES128}`

---

## Summary

| Challenge                      | Key Concept            | Flag                |
| ------------------------------ | ---------------------- | ------------------- |
| Keyed Permutations             | Bijection              | `crypto{bijection}` |
| Resisting Bruteforce           | Biclique Attack        | `crypto{biclique}`  |
| Structure of AES               | matrix2bytes           | `crypto{inmatrix}`  |
| Round Keys                     | AddRoundKey (XOR)      | `crypto{r0undk3y}`  |
| Confusion through Substitution | SubBytes / S-box       | `crypto{l1n34rly}`  |
| Diffusion through Permutation  | ShiftRows + MixColumns | `crypto{d1ffUs3R}`  |
| Bringing It All Together       | Full AES-128 Decrypt   | `crypto{MYAES128}`  |
