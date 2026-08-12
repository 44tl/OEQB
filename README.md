# Obfuscation, Encryption, and Quantum Breaking

---

## 1. Introduction

Every day, billions of people rely on systems that hide information. Your phone encrypts messages before sending them. Software companies obfuscate their compiled binaries to stop competitors from copying their logic. Websites hash passwords before storing them. And nation-states encrypt military communications with algorithms that would take classical computers longer than the age of the universe to break.

But the landscape is shifting fast. AI models are getting eerily good at reading mangled code and piecing together what obfuscation tried to hide. And on the horizon, quantum computers promise to break some of the most foundational encryption systems ever created.

This document covers three interconnected topics. First, obfuscation methods: what they are, how they work, and how long it takes to reverse them with current tools. Second, encryption methods: how they compare, how long brute-force attacks would take, and how AI changes the equation. Third, quantum computers: how Shor's algorithm dismantles RSA and ECC, how Grover's algorithm weakens AES, and what the actual code looks like.

---

## 2. Obfuscation Methods

Obfuscation is not encryption. Encryption uses a key and a mathematical algorithm to make data unreadable without that key. Obfuscation simply makes something harder to read or understand. There is no key. There is no mathematical proof of security. It is a cat-and-mouse game between the developer who wants to hide logic and the reverse engineer who wants to recover it.

### 2.1 Data Obfuscation

These techniques transform data so it is not immediately recognizable. They are the weakest form of obfuscation because the transformation is usually deterministic and reversible without any key.

#### Base64 Encoding

Base64 converts binary data into a text representation using 64 printable ASCII characters. It was designed for safely transmitting binary data through text-based channels like email, not for security. Anyone who sees a Base64 string can recognize the character set and decode it instantly.

Break time: **instantaneous**. A single function call reverses it.

```python
import base64

original = b"Hello, this is a secret message"
encoded = base64.b64encode(original)
# b'SGVsbG8sIHRoaXMgaXMgYSBzZWNyZXQgbWVzc2FnZQ=='

decoded = base64.b64decode(encoded)
# b'Hello, this is a secret message'
```

#### ROT13

ROT13 replaces each letter with the letter 13 positions after it in the alphabet. Since the alphabet has 26 letters, applying ROT13 twice returns the original text. It is the same operation for encoding and decoding.

Break time: **instantaneous**.

```python
import codecs

original = "The password is swordfish"
encoded = codecs.encode(original, 'rot_13')
# "Gur cnffjbeq vf fjbeqsvfu"

decoded = codecs.encode(encoded, 'rot_13')
# "The password is swordfish"
```

#### XOR Obfuscation

XOR (exclusive OR) is a bitwise operation. If you XOR data with a key, you get ciphertext. If you XOR the ciphertext with the same key, you get the original data back. XOR obfuscation is common in malware because it is fast and simple. Single-byte XOR is trivially broken. Multi-byte XOR with a repeating key is harder but still vulnerable to frequency analysis and known-plaintext attacks.

Break time:
- Single-byte XOR key: **seconds** (frequency analysis or brute force over 256 possibilities)
- 4-byte XOR key: **minutes** (known-plaintext attack reveals the key)
- 16-byte XOR key: **minutes to hours** if you have known plaintext, longer if not
- 256-byte XOR key with no known plaintext: **hours to days** for a determined analyst

```python
def xor_obfuscate(data: bytes, key: bytes) -> bytes:
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])

message = b"Attack at dawn"
key = b"SECRET"

obfuscated = xor_obfuscate(message, key)
# b'\x04\x15\x02\x1e\x1d\x59\x06\x03\x04\x1c\x5a\x47\x5e\x03\x1b'

recovered = xor_obfuscate(obfuscated, key)
# b'Attack at dawn'
```

#### Comparison of Data Obfuscation Methods

```
+------------------+---------------------------+--------------------+------------------+
| Method           | Key Required?             | Break Time         | Security Level   |
+------------------+---------------------------+--------------------+------------------+
| Base64           | No                        | Instantaneous      | None            |
| ROT13            | No                        | Instantaneous      | None            |
| Hex Encoding     | No                        | Instantaneous      | None            |
| URL Encoding     | No                        | Instantaneous      | None            |
| Single-byte XOR  | 1 byte (implicit)         | Seconds            | Negligible       |
| Multi-byte XOR   | N bytes (implicit)        | Minutes to hours   | Very Low        |
| Repeated-key XOR | Repeating key             | Hours to days      | Low             |
+------------------+---------------------------+--------------------+------------------+
```

### 2.2 Code Obfuscation

Code obfuscation transforms the structure of a program to make reverse engineering harder while keeping the program functionally identical. This is the kind of obfuscation used to protect commercial software, malware payloads, and intellectual property.

#### 2.2.1 Identifier Renaming

Variable names, function names, and class names are replaced with meaningless strings. `calculate_revenue` becomes `a1` or `_0x3f2a`. This is the weakest form of code obfuscation. A determined reverse engineer can rename things back based on context, and automated tools often recover the semantics through control flow analysis.

Break time: **minutes with automated tools**. Decompilers like Procyon, CFR, and Fernflower for Java, or dnSpy for .NET, handle this routinely.

#### 2.2.2 String Encryption

String literals in the code (URLs, API keys, error messages, file paths) are encrypted or encoded and only decrypted at runtime. This prevents reverse engineers from searching the binary for recognizable strings to understand what the program does.

Break time: **minutes to hours**. Dynamic analysis (running the program and hooking the decryption calls) reveals all strings immediately. Static analysis requires finding the decryption routine and applying it manually, which takes longer but is still straightforward for experienced analysts.

#### 2.2.3 Control Flow Flattening

This is one of the most powerful obfuscation techniques. The normal structure of if/else statements, loops, and function calls is destroyed and replaced with a single large switch statement inside a while loop. A state variable controls which block of code executes next. The original logic is still there, but the control flow graph becomes a flat, tangled mess.

A simple function like this:

```
Original control flow:         Flattened control flow:

    A                           +--->[Block A]--+
    |                           |               |
    v                           |    switch     |
    B                           |   (state var) |
    |                           |               |
    v                           +--->[Block B]--+
    C                           |               |
    |                           |    switch     |
    v                           |   (state var) |
    D                           |               |
                                +--->[Block C]--+
                                      |
                                +--->[Block D]--+
                                      |
                                 (back to switch)
```

Break time: **hours to days** for a human analyst manually, **minutes to an hour** for automated deobfuscation tools like SATURN, Unicorn, or machine-learning-based deflattening tools. Research from 2024 and 2025 shows that tools using symbolic execution and pattern matching can recover the original control flow in a significant percentage of cases.

#### 2.2.4 Virtual Machine Protection (VM Obfuscation)

The program's bytecode is translated into a custom, proprietary instruction set. At runtime, a custom virtual machine interpreter executes these custom bytecode instructions. The original logic is now encoded in an architecture that no standard disassembler understands.

This is currently one of the strongest obfuscation methods available. Tools like VMProtect, Themida, and Tigress use this approach. It makes static analysis extremely difficult because the analyst must first reverse-engineer the entire custom VM architecture before they can understand a single operation.

Break time: **days to weeks** for a human expert, **varies widely** for automated tools. Some research prototypes can partially devirtualize simple VMs, but commercial VM protectors are still considered hard targets.

#### 2.2.5 Code Integrity Checks and Anti-Debugging

These are not obfuscation per se, but they are layered on top of it to prevent dynamic analysis. Techniques include:
- Checking whether a debugger is attached to the process
- Calculating checksums of code sections at runtime and aborting if they have been modified
- Detecting virtual machine environments
- Using timing checks to detect single-stepping in a debugger
- Packing the executable so the real code is only decrypted in memory at runtime

Break time: **adds hours to days** to the reverse engineering process. Each layer must be identified and bypassed before analysis can proceed. Tools like x64dbg, Cheat Engine, and Frida are commonly used to bypass these protections.

#### Comparison of Code Obfuscation Methods

```
+----------------------------+-------------------+-----------------------+-------------------+
| Technique                  | Manual Break     | Automated Break       | Resilience Level |
|                            | Time             | Time                  |                   |
+----------------------------+-------------------+-----------------------+-------------------+
| Identifier renaming        | Minutes           | Seconds               | Very Low         |
| String encryption          | Hours             | Minutes (dynamic)     | Low              |
| Dead code insertion        | Hours             | Minutes               | Low              |
| Control flow flattening    | Days              | Minutes to hours      | Medium           |
| Instruction substitution   | Days              | Hours                 | Medium           |
| Opaque predicates          | Days              | Hours                 | Medium-High      |
| Virtual machine protection | Weeks             | Partial/Unreliable    | High             |
| Multi-layer packing +      | Weeks to months   | Varies                | Very High        |
| anti-debug + VM            |                   |                       |                   |
+----------------------------+-------------------+-----------------------+-------------------+
```

---

## 3. Encryption Methods and Break Times

Unlike obfuscation, encryption is designed with mathematical proofs of security. The question is not whether the algorithm can be reversed in principle, but how long it would take with available computing power.

### 3.1 Symmetric Encryption (AES)

Advanced Encryption Standard (AES) is the most widely used symmetric cipher. The same key is used to encrypt and decrypt. AES comes in three key sizes: 128-bit, 192-bit, and 256-bit. The only known way to break AES (when used correctly) is brute force: trying every possible key until you find the right one.

The number of possible keys for each variant:

```
AES-128:  2^128  = 340,282,366,920,938,463,463,374,607,431,768,211,456
AES-192:  2^192  = 6,277,101,735,386,680,763,835,789,423,207,666,416,102,355,444,464,034,512,896
AES-256:  2^256  = 115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,936
```

To put these numbers in perspective, assume you could check one trillion (10^12) keys per second, which is far beyond the capability of any single classical computer but theoretically possible with a massive supercomputing cluster:

```
+----------+-----------------------------+----------------------------+
| Cipher   | Time at 10^12 keys/sec      | Comparison                  |
+----------+-----------------------------+----------------------------+
| AES-128  | ~10.8 billion years         | ~78% of the age of the      |
|          |                             | universe                    |
| AES-192  | ~2.0 x 10^27 years          | 1.5 x 10^17 times the age   |
|          |                             | of the universe             |
| AES-256  | ~3.7 x 10^45 years          | 2.7 x 10^35 times the age   |
|          |                             | of the universe             |
+----------+-----------------------------+----------------------------+
```

Even if every atom on Earth were a computer checking keys at a billion per second, AES-256 would still take longer than the age of the universe to brute force. AI does not change this math. AI cannot magically skip checking keys. The key space is simply too large.

### 3.2 Asymmetric Encryption (RSA)

RSA security relies on the difficulty of factoring the product of two large prime numbers. If you multiply two primes together, the result is easy to compute. If I give you the product and ask you to find the original primes, there is no known efficient classical algorithm to do it.

The most efficient classical algorithm for factoring large integers is the General Number Field Sieve (GNFS). Its running time is sub-exponential, which means it is much faster than brute force but still astronomically slow for large key sizes.

```
+----------+-----------------------------+-----------------------------------+
| Key Size | GNFS Estimated Break Time   | Notes                             |
+----------+-----------------------------+-----------------------------------+
| RSA-512  | Months on a cluster         | Factored in 1999 (155 digits)    |
| RSA-768  | ~2,000 core-years           | Factored in 2009 (232 digits)    |
| RSA-1024 | Billions of dollars, years   | Not yet factored, but feasible    |
|          | of effort (estimated)       | for well-funded nation-states    |
| RSA-2048 | Trillions of dollars,       | Completely infeasible with       |
|          | millions of years           | classical computers              |
| RSA-4096 | Beyond any conceivable      | Heat death of the universe       |
|          | classical effort            | territory                        |
+----------+-----------------------------+-----------------------------------+
```

### 3.3 Elliptic Curve Cryptography (ECC)

ECC provides security equivalent to RSA with much smaller key sizes. An ECC-256 key offers roughly the same security as RSA-3072. ECC's security relies on the Elliptic Curve Discrete Logarithm Problem (ECDLP), which has no known efficient classical solution.

```
+------------------+-------------------+-------------------+
| ECC Key Size     | RSA Equivalent    | Classical Break  |
+------------------+-------------------+-------------------+
| ECC-160          | RSA-1024          | Feasible with    |
|                  |                   | effort           |
| ECC-256          | RSA-3072          | Infeasible       |
| ECC-384          | RSA-7680          | Far beyond       |
|                  |                   | classical reach  |
| ECC-521          | RSA-15360         | Computationally  |
|                  |                   | impossible       |
+------------------+-------------------+-------------------+
```

### 3.4 Hashing (Not Encryption, But Related)

Hashing is one-way. You cannot "decrypt" a hash. The only approach is to guess the input, hash it, and compare. For strong hashes like bcrypt, scrypt, or Argon2, this is deliberately made slow.

```
+------------------+-------------------------+----------------------------------+
| Hash             | Brute Force Speed       | Notes                            |
+------------------+-------------------------+----------------------------------+
| MD5              | ~30 billion/sec         | Broken, collision attacks exist  |
| SHA-256          | ~5 billion/sec          | Still secure for hashing         |
| bcrypt           | ~10,000-100,000/sec     | Deliberately slow (cost factor)  |
| Argon2           | ~1,000-10,000/sec       | Memory-hard, best current choice |
+------------------+-------------------------+----------------------------------+
```

---

## 4. AI and the Deobfuscation Game

This is where things get interesting. Traditional deobfuscation required a human reverse engineer sitting at a disassembler for hours or days. AI is changing that.

### 4.1 LLMs for Code Deobfuscation

A 2024 study by Patsakis and colleagues evaluated GPT-4, Claude, and other state-of-the-art LLMs on their ability to deobfuscate real-world malware samples. The results were notable. While the models were not perfect, some of them could efficiently deobfuscate payloads that would take a human analyst significant time to understand. The models performed best on string deobfuscation (recovering encoded strings) and moderately well on control flow recovery.

A 2025 paper by Hajipour and colleagues trained an LLM specifically on code deobfuscation using 30,000 training samples. Their model achieved substantially better results than general-purpose LLMs, suggesting that specialized training makes a real difference. A 2026 paper by Choi went further, proposing an architecture called Efficient Deobfuscation via LLMs that uses a two-stage approach: first, the model classifies the obfuscation type, then it applies a specialized deobfuscation strategy.

A separate 2025 study provided the first comprehensive evaluation of commercial LLMs on assembly-level code deobfuscation, introducing a four-dimensional framework for measuring deobfuscation quality. The findings showed that while LLMs can handle simple obfuscation (Base64, ROT13, single-byte XOR) near-perfectly, they struggle with multi-layer obfuscation and virtual machine protection.

### 4.2 What AI Can and Cannot Do

```
+---------------------------+---------------------+------------------------------------+
| Task                      | AI Capability       | Notes                              |
+---------------------------+---------------------+------------------------------------+
| Base64 decode             | Near-perfect        | Trivial for any LLM                |
| ROT13 decode              | Near-perfect        | Trivial for any LLM                |
| Single-byte XOR           | Near-perfect        | Easily identified and reversed     |
| Multi-byte XOR            | Good                | With known plaintext, very capable |
| String deobfuscation      | Good to very good   | 2024-2025 research shows strong    |
|                           |                     | results on real malware            |
| Identifier renaming       | Moderate            | Can infer purpose from context     |
| Control flow recovery     | Moderate            | Improving rapidly with specialized |
|                           |                     | training                           |
| Opaque predicate          | Moderate            | Can identify common patterns       |
| detection                 |                     |                                    |
| VM-based obfuscation      | Weak                | Still a significant challenge      |
| Breaking AES-256          | Not applicable      | AI cannot break mathematically     |
|                           |                     | secure encryption                  |
| Breaking RSA-2048         | Not applicable      | Same reason                        |
| Finding zero-days in      | Emerging            | Research stage, not reliable       |
| crypto implementations    |                     |                                    |
+---------------------------+---------------------+------------------------------------+
```

The key insight is that AI is extremely effective at undoing obfuscation (which was never mathematically secure to begin with) but fundamentally cannot break encryption whose security rests on mathematical hardness. No amount of pattern recognition changes the fact that AES-256 has 2^256 possible keys.

### 4.3 AI Tools in Practice

Several practical tools now integrate AI into the reverse engineering workflow:

- **Ghidra with AI plugins**: The NSA's open-source reverse engineering tool now has community-built LLM plugins that can rename variables, explain functions, and identify obfuscation patterns.
- **IDA Pro with AI assist**: Commercial plugin ecosystems increasingly include ML-based decompilation improvements and heuristic analysis.
- **Binary Ninja with ML plugins**: Offers cloud-based AI analysis that can identify obfuscation types and suggest deobfuscation strategies.
- **Custom LLM pipelines**: Security teams at large organizations are building internal pipelines that feed disassembled code to fine-tuned models for automated initial triage.

---

## 5. Quantum Computers and Cryptography

This is the part that has cryptographers, governments, and tech companies genuinely concerned. Quantum computers do not just make breaking encryption faster. They make a fundamentally different class of attacks possible.

### 5.1 Why Quantum Is Different

Classical computers process information in bits. Each bit is either 0 or 1. Quantum computers use qubits, which can exist in a superposition of both 0 and 1 simultaneously. When you have multiple qubits, they can be entangled, meaning the state of one qubit is correlated with the state of others. This allows quantum computers to process a vast number of possibilities simultaneously in a way that classical computers simply cannot.

The crucial detail is that quantum computers do not just "try all keys at once." That is a common oversimplification. What they actually do is exploit quantum interference to amplify the probability of the correct answer while canceling out wrong answers. The algorithms that do this, Shor's and Grover's, are fundamentally different from brute force.

### 5.2 Shor's Algorithm: Breaking RSA and ECC

Shor's algorithm, developed by Peter Shor in 1994, is the single biggest threat to modern public-key cryptography. It can factor large integers and compute discrete logarithms in polynomial time on a quantum computer. Since RSA and ECC both rely on problems that Shor's algorithm solves efficiently, both systems collapse entirely if a sufficiently powerful quantum computer exists.

#### How It Works (Simplified)

The algorithm factors an integer N by reducing the problem to finding the period of a function. The steps are:

1. Pick a random number a less than N
2. Use a quantum computer to find the period r of the function f(x) = a^x mod N
3. If r is even, compute gcd(a^(r/2) - 1, N), which gives a factor of N with high probability
4. If r is odd, go back to step 1 and pick a different a

The quantum part is step 2: finding the period. Classically, finding the period requires evaluating the function many times. Quantumly, Shor's algorithm uses the Quantum Phase Estimation subroutine, which evaluates the function in superposition and uses the Quantum Fourier Transform to extract the period with high probability.

#### What It Means for Specific Algorithms

```
+-------------------+------------------+----------------------------------------+
| Algorithm         | Status Under     | Impact                                 |
|                   | Shor's Algorithm |                                        |
+-------------------+------------------+----------------------------------------+
| RSA               | Completely       | Factoring the modulus breaks RSA      |
|                   | broken           | entirely. All key sizes.               |
+-------------------+------------------+----------------------------------------+
| Diffie-Hellman    | Completely       | Computing discrete logs breaks DH     |
| (finite field)    | broken           | key exchange entirely.                 |
+-------------------+------------------+----------------------------------------+
| ECDH / ECDSA      | Completely       | Computing elliptic curve discrete     |
|                   | broken           | logs breaks ECC entirely.             |
+-------------------+------------------+----------------------------------------+
| AES (symmetric)   | Not affected     | Shor's algorithm does not apply to    |
|                   | by Shor's        | symmetric encryption.                  |
+-------------------+------------------+----------------------------------------+
| SHA-256 (hashing) | Not affected     | Shor's algorithm does not apply.      |
|                   | by Shor's        |                                        |
+-------------------+------------------+----------------------------------------+
```

#### How Many Qubits Does It Actually Take?

This is the critical question. As of 2026, quantum computers have not broken RSA-2048. The reason is not that Shor's algorithm is theoretical. It has been demonstrated in the lab. The reason is that current quantum computers do not have enough stable, error-corrected qubits.

```
+----------+-------------------------+--------------------------+------------------+
| Target   | Logical Qubits Needed   | Physical Qubits Needed  | Current Status  |
|          | (ideal)                 | (with error correction) |                 |
+----------+-------------------------+--------------------------+------------------+
| RSA-15   | ~10                     | ~7 (done in 2001 by IBM)| Factored        |
| RSA-21   | ~12                     | ~10                      | Factored        |
| RSA-2048 | ~4,096                  | ~20 million (2025 est.)  | Not yet        |
|          |                         | ~100,000 (2026 improved  |                 |
|          |                         | estimate)                |                 |
| RSA-4096 | ~8,192                  | Hundreds of millions     | Not yet        |
+----------+-------------------------+--------------------------+------------------+
```

A 2025 paper estimated that breaking RSA-2048 could be done with roughly 20 million physical qubits in about 8 hours. A 2026 follow-up paper improved the methodology and reduced the estimate to approximately 100,000 physical qubits, a hundredfold reduction. Both are far beyond the current state of the art. IBM's largest quantum processor as of 2026 has on the order of a thousand qubits, and those are physical qubits without full error correction.

### 5.3 Grover's Algorithm: Weakening AES

Grover's algorithm provides a quadratic speedup for unstructured search problems. In the context of encryption, this means it can search the key space of a symmetric cipher roughly sqrt(N) times faster than classical brute force.

#### The Math

For AES-128:
- Classical brute force: 2^128 operations
- Grover's algorithm: 2^64 operations (quadratic speedup)

For AES-256:
- Classical brute force: 2^256 operations
- Grover's algorithm: 2^128 operations

#### Why This Is Manageable

A quadratic speedup sounds dramatic, but it is far less devastating than Shor's exponential speedup. The practical response is simple: **double the key size**.

```
+------------------+-------------------------+-------------------------+
| Cipher           | Classical Security      | Post-Quantum Security   |
|                  | (Classical Brute Force) | (With Grover's)         |
+------------------+-------------------------+-------------------------+
| AES-128          | 128-bit security        | 64-bit security         |
|                  | (infeasible)            | (feasible with effort)  |
| AES-192          | 192-bit security        | 96-bit security         |
|                  | (infeasible)            | (still infeasible)      |
| AES-256          | 256-bit security        | 128-bit security        |
|                  | (infeasible)            | (infeasible)            |
+------------------+-------------------------+-------------------------+
```

AES-256, even with Grover's algorithm, still provides 128 bits of security. That is equivalent to the classical security of AES-128, which is considered unbreakable. So the fix is straightforward: migrate to AES-256 if you have not already.

A 2026 paper explicitly argued that quantum computers are not a threat to 128-bit symmetric encryption in practice, because the overhead of quantum error correction and the constant factors in Grover's algorithm make the attack impractical with any quantum computer that could be built in the foreseeable future.

### 5.4 Example Code: Shor's Algorithm Factoring 15

This is a working example using Qiskit, IBM's open-source quantum computing framework. It factors the number 15 (3 x 5), which is the canonical demo because it is small enough to run on current quantum hardware (and simulators). Do not expect this to break RSA-2048. Factoring 15 requires only a handful of qubits. Factoring a 617-digit RSA-2048 modulus would require millions.

```python
"""
Shor's Algorithm Demo - Factoring 15 using Qiskit
This is a simplified educational implementation.
It will NOT break real-world RSA keys.
Requires: pip install qiskit qiskit-aer
"""

from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import math
from math import gcd
import random

def shors_algorithm_factoring(N: int):
    """
    Factor N using Shor's algorithm (simplified).
    This demo works for small numbers like 15.
    """
    print(f"Attempting to factor {N}...")
    
    # Try multiple random 'a' values if needed
    for attempt in range(10):
        a = random.randint(2, N - 1)
        
        # Make sure a and N are coprime
        if gcd(a, N) != 1:
            factor = gcd(a, N)
            print(f"  Found factor trivially: gcd({a}, {N}) = {factor}")
            return factor, N // factor
        
        print(f"  Attempt {attempt + 1}: trying a = {a}")
        
        # --- QUANTUM PART: Find the period r of f(x) = a^x mod N ---
        # For N=15, we need n = ceil(log2(N)) = 4 qubits for the
        # work register and 2n = 8 qubits for the counting register
        
        n = math.ceil(math.log2(N))
        counting_qubits = 2 * n
        
        qc = QuantumCircuit(counting_qubits + n, counting_qubits)
        
        # Step 1: Apply Hadamard gates to counting register
        # This puts all counting qubits in superposition
        for q in range(counting_qubits):
            qc.h(q)
        
        # Step 2: Apply the modular exponentiation unitary
        # For N=15, a=2, the function f(x) = 2^x mod 15 has period 4:
        #   2^0 mod 15 = 1
        #   2^1 mod 15 = 2
        #   2^2 mod 15 = 4
        #   2^3 mod 15 = 8
        #   2^4 mod 15 = 1  (period repeats)
        # This is the hard part for large N. For N=15, we hardcode it.
        
        # Simplified modular exponentiation for a=2, N=15
        # The pattern repeats every 4, so we encode that
        if a == 2 and N == 15:
            # Encode f(x) = 2^x mod 15 into the work register
            # x=0 -> 1 (0001), x=1 -> 2 (0010), x=2 -> 4 (0100),
            # x=3 -> 8 (1000)
            qc.ccx(0, 1, 4)   # partial entanglement
            qc.ccx(0, 2, 5)
            qc.ccx(0, 3, 6)
            qc.ccx(1, 2, 7)
        elif a == 4 and N == 15:
            qc.x(5)          # 4 -> 0100
        elif a == 7 and N == 15:
            qc.x(4)          # 7 -> 0111
            qc.x(5)
            qc.x(6)
        elif a == 8 and N == 15:
            qc.x(7)          # 8 -> 1000
        elif a == 11 and N == 15:
            qc.x(4)          # 11 -> 1011
            qc.x(5)
            qc.x(7)
        elif a == 13 and N == 15:
            qc.x(4)          # 13 -> 1101
            qc.x(6)
            qc.x(7)
        else:
            print(f"  Skipping a={a}, not precomputed for N=15")
            continue
        
        # Step 3: Apply Inverse Quantum Fourier Transform to
        # counting register to extract the period information
        for q in range(counting_qubits // 2):
            qc.swap(q, counting_qubits - 1 - q)
        
        for j in range(counting_qubits):
            for k in range(j):
                angle = -math.pi / (2 ** (j - k))
                qc.cp(angle, k, j)
            qc.h(j)
        
        # Step 4: Measure the counting register
        qc.measure(range(counting_qubits), range(counting_qubits))
        
        # Run on simulator
        simulator = AerSimulator()
        compiled = transpile(qc, simulator)
        result = simulator.run(compiled, shots=1024).result()
        counts = result.get_counts()
        
        # Get the most frequent measurement outcome
        measured = max(counts, key=counts.get)
        phase = int(measured, 2) / (2 ** counting_qubits)
        
        # Use continued fractions to estimate the period r
        # from the measured phase
        r = estimate_period(phase, counting_qubits)
        
        if r is None or r % 2 != 0:
            print(f"  Could not determine even period, trying again...")
            continue
        
        # Step 5: If r is even, try to extract factors
        # gcd(a^(r/2) - 1, N) should give a factor
        candidate = pow(a, r // 2, N)
        factor = gcd(candidate - 1, N)
        
        if 1 < factor < N:
            print(f"  Period r = {r}")
            print(f"  gcd({a}^{r//2} - 1, {N}) = gcd({candidate - 1}, {N}) = {factor}")
            return factor, N // factor
        
        factor = gcd(candidate + 1, N)
        if 1 < factor < N:
            print(f"  Period r = {r}")
            print(f"  gcd({a}^{r//2} + 1, {N}) = gcd({candidate + 1}, {N}) = {factor}")
            return factor, N // factor
        
        print(f"  Period r = {r} but no factor found, trying another a...")
    
    return None, None


def estimate_period(phase, num_bits):
    """
    Estimate the period r from the measured phase using
    continued fraction expansion.
    """
    if phase == 0:
        return None
    
    # Continued fraction expansion to find the denominator
    # that best approximates the phase
    from fractions import Fraction
    frac = Fraction(phase).limit_denominator(2 ** num_bits)
    return frac.denominator


# --- Run the demo ---
if __name__ == "__main__":
    N = 15
    print("=" * 50)
    print("Shor's Algorithm Demo: Factoring 15")
    print("=" * 50)
    print()
    
    factor1, factor2 = shors_algorithm_factoring(N)
    
    if factor1 and factor2:
        print()
        print(f"Result: {N} = {factor1} x {factor2}")
    else:
        print("\nFailed to find factors. Run again for another attempt.")
```

#### Running the Same Thing as a Classical Comparison

To understand why Shor's algorithm matters, here is what the classical approach looks like for factoring the same number. For N=15 the difference is invisible. For N = a 617-digit RSA modulus, the difference is the age of the universe versus hours.

```python
"""
Classical factoring for comparison.
Trivial for 15. Impossible for RSA-2048.
"""

def classical_factor(N):
    """Try dividing by every number up to sqrt(N)."""
    for i in range(2, int(N**0.5) + 1):
        if N % i == 0:
            return i, N // i
    return None, None

# Factoring 15: instant on any computer
print(classical_factor(15))  # (3, 5)

# Factoring a 2048-bit RSA modulus (617 decimal digits):
# Even with the General Number Field Sieve (the best classical
# algorithm), this would require more operations than there
# are atoms in the observable universe.
```

### 5.5 Q-Day: When Will Quantum Computers Break Real Encryption?

"Q-Day" is the term used in government and industry to describe the day a quantum computer becomes capable of breaking widely used public-key encryption in practice. Nobody knows exactly when this will happen, and estimates vary widely.

```
+-------------------+-------------------------------------------+------------------+
| Source / Estimate | Predicted Timeline                       | Confidence       |
+-------------------+-------------------------------------------+------------------+
| NSA (2022)        | "Within the next few decades"            | Conservative     |
| IBM (2025)        | 2030s for practical quantum advantage     | Moderate         |
|                 | for specific problems                     |                  |
| Q-Day Clock       | ~2030 +/- 2 years for RSA-2048            | Aggressive       |
| project (2026)    |                                           |                  |
| Academic consensus| 2035-2050 for cryptographically relevant  | Moderate         |
| (2025-2026)       | quantum computers                          |                  |
| Skeptics          | 2050+ or never at useful scale             | Questioning      |
+-------------------+-------------------------------------------+------------------+
```

The uncertainty comes from the gap between logical qubits (the error-free qubits that algorithms are designed for) and physical qubits (the noisy, error-prone qubits that current hardware actually provides). Current quantum error correction requires roughly 1,000 physical qubits to produce one reliable logical qubit. This overhead is the main bottleneck.

A 2026 paper from cryptography engineers argued that if a quantum computer becomes stable enough to break RSA-2048, it means the hardest problem in quantum computing (fault-tolerant scaling) has been largely solved, which would be a far bigger achievement than the factoring itself. The reasoning is that the engineering challenges involved in building a machine with 100,000+ error-corrected qubits are so immense that the timeline is inherently uncertain.

### 5.6 The Real Threat: Harvest Now, Decrypt Later

The most pressing quantum threat is not about breaking encryption today. It is about encrypted data being captured today and decrypted later when quantum computers become available.

A nation-state or well-funded adversary could be recording encrypted internet traffic right now, storing it, and waiting. In ten or twenty years, when a quantum computer capable of running Shor's algorithm exists, they decrypt everything they have been saving. This means data that is sensitive for decades (military secrets, intelligence communications, personal health records, financial data) is already at risk, even though quantum computers cannot break it yet.

This is why the push to migrate to post-quantum cryptography is urgent, even though current quantum computers cannot break anything meaningful.

---

## 6. Post-Quantum Cryptography: The Response

In August 2024, NIST finalized its first three post-quantum cryptography standards. These are algorithms designed to be secure against both classical and quantum computers.

### 6.1 NIST PQC Standards (Finalized 2024)

```
+------------------------+------------------+---------------------------------------+
| Standard               | NIST FIPS        | Type                                  |
+------------------------+------------------+---------------------------------------+
| ML-KEM (Kyber)         | FIPS 203         | Key encapsulation mechanism           |
|                        |                  | (replaces RSA/ECDH key exchange)     |
+------------------------+------------------+---------------------------------------+
| ML-DSA (Dilithium)     | FIPS 204         | Digital signature algorithm           |
|                        |                  | (replaces RSA/ECDSA signatures)      |
+------------------------+------------------+---------------------------------------+
| SLH-DSA (SPHINCS+)     | FIPS 205         | Hash-based digital signature          |
|                        |                  | (backup, conservative option)        |
+------------------------+------------------+---------------------------------------+
```

### 6.2 How These Algorithms Resist Quantum Attack

**ML-KEM (Kyber)** is a lattice-based cryptographic scheme. Its security is based on the Learning With Errors (LWE) problem, which involves finding a short vector in a high-dimensional lattice. Shor's algorithm does not apply to lattice problems. The best known quantum attack on LWE is still exponential in the security parameter, meaning the problem remains hard even for quantum computers.

**ML-DSA (Dilithium)** is also lattice-based, relying on the Module Learning With Errors problem. Same story: no polynomial-time quantum algorithm is known.

**SLH-DSA (SPHINCS+)** is hash-based. It relies only on the security of hash functions. Quantum computers can use Grover's algorithm to speed up collisions, but the scheme accounts for this by using large hash outputs. It is the most conservative option but also the one with the largest signatures.

### 6.3 Migration Status

As of 2026, major tech companies are in various stages of PQC migration. Google has been testing Kyber in Chrome since 2023. Apple, Cloudflare, and Signal have integrated PQC algorithms into their protocols. The U.S. government has mandated that all federal systems must migrate to PQC by 2035 (per NSM-10). The challenge is that the migration is enormously complex. Every certificate, every key exchange, every signature in every system needs to be updated without breaking existing functionality.

---

## 7. Putting It All Together: The Break-Time Master Table

This table summarizes everything covered in this document. Break times assume a well-funded, technically sophisticated adversary.

```
+-----------------------------+-------------------------+---------------------------------------+
| Protection Method          | Classical Break Time    | Quantum Break Time                   |
+-----------------------------+-------------------------+---------------------------------------+
| Base64 encoding            | Instantaneous           | Instantaneous (not relevant)          |
| ROT13                      | Instantaneous           | Instantaneous (not relevant)          |
| Single-byte XOR            | Seconds                 | Seconds (not relevant)                |
| Multi-byte XOR (16 bytes)  | Minutes to hours        | Minutes to hours (not relevant)       |
| Identifier renaming        | Minutes (automated)     | Minutes (not relevant)                |
| String encryption          | Minutes to hours        | Minutes to hours (not relevant)       |
| Control flow flattening    | Days (manual)           | Days (not relevant)                   |
| VM-based obfuscation       | Weeks (manual)          | Weeks (not relevant)                  |
| AES-128                    | 10+ billion years       | ~100+ million years (Grover)          |
| AES-256                    | 3.7 x 10^45 years       | 10+ billion years (Grover)           |
| RSA-1024                   | Feasible (expensive)    | Minutes (Shor, with sufficient qubits)|
| RSA-2048                   | Trillions of years      | Hours (Shor, with ~100K+ qubits)      |
| RSA-4096                   | Beyond classical reach  | Hours-days (Shor, with more qubits)   |
| ECC-256                    | Infeasible              | Hours (Shor, with sufficient qubits)  |
| ML-KEM / Kyber (PQC)       | Infeasible              | Infeasible (by design)                |
| ML-DSA / Dilithium (PQC)   | Infeasible              | Infeasible (by design)                |
| SHA-256                    | Infeasible              | Weakened but still feasible at        |
|                            |                         | 128-bit equivalent (Grover)           |
| bcrypt / Argon2            | Infeasible              | Reduced but still infeasible          |
+-----------------------------+-------------------------+---------------------------------------+
```

---

## 8. Summary of Key Takeaways

**On obfuscation:** Obfuscation is a speed bump, not a wall. It slows down reverse engineers but does not stop them. AI is rapidly making it faster to undo obfuscation, but the strongest methods (VM protection, multi-layer packing) still require significant human expertise. If your threat model includes determined reverse engineers, obfuscation alone is not enough.

**On encryption:** Mathematically sound encryption (AES-256, RSA-2048, ECC-256) is currently unbreakable with classical computers. AI cannot change this because the security rests on the sheer size of the key space, not on the obscurity of the algorithm. The weakest link in any encrypted system is almost always the implementation, not the math.

**On quantum computers:** Shor's algorithm is a real and existential threat to RSA and ECC. It is not theoretical in principle, only in practice due to hardware limitations. Grover's algorithm weakens AES but the fix is simple: use AES-256. Post-quantum algorithms (Kyber, Dilithium, SPHINCS+) already exist and have been standardized. The real danger is the "harvest now, decrypt later" threat, which makes migration urgent today even though quantum computers cannot break anything yet.

---

## 9. References and Further Reading

- Shor, P. (1994). Algorithms for quantum computation: discrete logarithms and factoring. *Proceedings of the 35th Annual Symposium on Foundations of Computer Science*.
- Grover, L. (1996). A fast quantum mechanical algorithm for database search. *Proceedings of the 28th Annual ACM Symposium on Theory of Computing*.
- Patsakis, C. et al. (2024). Assessing LLMs in malicious code deobfuscation of real-world malware. *IEEE Access*.
- Hajipour, H. et al. (2025). Exploring the Potential of LLMs for Code Deobfuscation.
- Choi, B. (2026). Toward Efficient Deobfuscation via Large Language Models.
- NIST (2024). FIPS 203: ML-KEM (Module-Lattice-Based Key-Encapsulation Mechanism).
- NIST (2024). FIPS 204: ML-DSA (Module-Lattice-Based Digital Signature Algorithm).
- NIST (2024). FIPS 205: SLH-DSA (Stateless Hash-Based Digital Signature Standard).
- Craig, G. et al. (2025). The Enormous Energy Cost of Breaking RSA-2048 with Quantum Computers.
- Robinson, D.G. (2026). On reducing the cost of breaking RSA-2048.
- Oedipus framework: Metadata recovery from obfuscated programs using machine learning.
- IBM Quantum Learning. Shor's Algorithm. https://learning.quantum.ibm.com/
- Qiskit documentation. https://qiskit.org/documentation/
