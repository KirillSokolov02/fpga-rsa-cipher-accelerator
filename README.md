# RSA Hardware Accelerator — 128-Bit Modular Exponentiation

A **Vivado FPGA project** implementing a **hardware RSA cipher accelerator** capable of performing **128-bit modular exponentiation** (`c = m^e mod n`) for RSA encryption or decryption. Built for **Xilinx Artix-7** using **Vivado 2022.2**.

## Overview

This project implements a **128-bit RSA hardware accelerator** that performs public-key encryption/decryption using the square-and-multiply algorithm for modular exponentiation. RSA (Rivest–Shamir–Adleman) is one of the first and most widely used public-key cryptosystems.

The accelerator computes `c = m^e mod n` where:
- **m** (message/ciphertext) is a 128-bit input
- **e** (exponent) is a 128-bit key component
- **n** (modulus) is a 128-bit key component
- **c** (ciphertext/message) is a 128-bit output

## Cryptographic Background

RSA security relies on the difficulty of factoring the product of two large primes. The key components are:
- **Public key**: (e, n) — used for encryption
- **Private key**: (d, n) — used for decryption, where d·e ≡ 1 (mod φ(n))

**Encryption**: `c = m^e mod n`
**Decryption**: `m = c^d mod n`

## Architecture

The accelerator uses a **square-and-multiply** algorithm (binary method for modular exponentiation), which iterates through the bits of the exponent:

```
Algorithm: Square-and-Multiply (Right-to-Left)
input: indata (m), inExp (e), inMod (n)
output: cypher (m^e mod n)

root ← m
sq ← n  (or precomputed value)
temp ← 1

for i from 0 to 127:
    if bit i of e is 1:
        temp ← (temp * root) mod n
    root ← (root * root) mod n

cypher ← temp
```

### Component Breakdown

#### RSACypher Entity

Top-level RSA cipher module that orchestrates the modular exponentiation:

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| **indata** | 128-bit | Input | Message (m) for encryption or ciphertext (c) for decryption |
| **inExp** | 128-bit | Input | Exponent (e for encryption, d for decryption) |
| **inMod** | 128-bit | Input | RSA modulus (n) — common to both operations |
| **cypher** | 128-bit | Output | Result: ciphertext or decrypted message |
| **clk** | 1-bit | Input | Clock signal |
| **ds** | 1-bit | Input | Data strobe / start trigger |
| **reset** | 1-bit | Input | Synchronous reset |
| **ready** | 1-bit | Output | Computation complete flag |

**Generic Parameter**: `KEYSIZE` — configurable key size (default 128 bits)

**Operation**:
1. On `ds` (data strobe) assertion, the module begins processing
2. The square-and-multiply loop runs for 128 clock cycles (one per exponent bit)
3. Two `modmult32` units perform parallel squaring and multiplication
4. On completion, `ready` is asserted for one cycle
5. Result is available on `cypher` output

#### modmult32 (Modular Multiplier)

Core component performing `(a × b) mod m` for 128-bit operands:

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| **mpand** | MPWID-bit | Input | First operand |
| **mplier** | MPWID-bit | Input | Second operand |
| **modulus** | MPWID-bit | Input | Modulus for reduction |
| **product** | MPWID-bit | Output | Result: (mpand × mplier) mod modulus |
| **clk** | 1-bit | Input | Clock |
| **ds** | 1-bit | Input | Data strobe / start trigger |
| **reset** | 1-bit | Input | Synchronous reset |
| **ready** | 1-bit | Output | Computation complete |

**Generic Parameter**: `MPWID` — operand width (default 128 bits)

**Algorithm**: The modular multiplier uses a **restoring division algorithm** with a 129-bit accumulator:
- The multiplication produces a 256-bit intermediate result
- Sequential subtraction of the modulus (shifted to appropriate position)
- Final result fits in 128 bits
- Internal registers use 129 bits (MPWID+1) to handle carry

**Internal Structure**:
- **mpreg**: First operand register
- **mcreg/mcreg1/mcreg2**: Multiplier registers with extension
- **modreg1/modreg2**: Modulus registers with extension
- **prodreg/prodreg1–4**: Partial product pipeline registers
- **count**: Iteration counter for the algorithm
- **modstate**: FSM state for operation control

### Data Path

```
RSACypher (top-level)
├── modmult32 × 2 (parallel squaring and multiplication)
│   ├── Square unit: root × root mod modulus
│   └── Multiply unit: temp × root mod modulus (conditional)
├── 128-bit registers (root, temp, sqrin, tempin)
├── 128-bit mux (select temp or root for output)
└── 128-bit counter (exponent bit iteration)
```

**Two modular multipliers operate in parallel**:
1. **Square multiplier**: Always computes `root × root mod modulus`
2. **Multiply multiplier**: Computes `temp × root mod modulus` when the current exponent bit is 1

The conditional multiply is handled by muxing the inputs to the multiplier.

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| **KEYSIZE** | 128 | RSA key size in bits (top-level) |
| **MPWID** | 128 | Modular multiplier operand width |

## Interfaces

### RSACypher Port Map

```vhdl
entity RSACypher is
    Generic (KEYSIZE: integer := 128);
    Port ( indata   : in  std_logic_vector(KEYSIZE-1 downto 0);
           inExp    : in  std_logic_vector(KEYSIZE-1 downto 0);
           inMod    : in  std_logic_vector(KEYSIZE-1 downto 0);
           cypher   : out std_logic_vector(KEYSIZE-1 downto 0);
           clk      : in  std_logic;
           ds       : in  std_logic;
           reset    : in  std_logic;
           ready    : out std_logic
         );
end entity;
```

### modmult32 Port Map

```vhdl
entity modmult32 is
    Generic (MPWID: integer := 128);
    Port ( mpand     : in  std_logic_vector(MPWID-1 downto 0);
           mplier    : in  std_logic_vector(MPWID-1 downto 0);
           modulus   : in  std_logic_vector(MPWID-1 downto 0);
           product   : out std_logic_vector(MPWID-1 downto 0);
           clk       : in  std_logic;
           ds        : in  std_logic;
           reset     : in  std_logic;
           ready     : out std_logic
         );
end entity;
```

## Operation Flow

1. **Load Phase**: Apply `indata`, `inExp`, and `inMod` to the inputs
2. **Start**: Assert `ds` for one clock cycle to begin computation
3. **Processing**: The square-and-multiply loop executes over 128 clock cycles:
   - For each exponent bit (LSB first):
     - If bit = 1: multiply `temp` by `root` and take modulo
     - Square `root` and take modulo
4. **Complete**: `ready` is asserted when computation finishes
5. **Read**: Read `cypher` output for the result

**Total latency**: 128 clock cycles (for 128-bit key)

## Security Considerations

**Important**: This is a **baseline implementation** for educational purposes. It does **NOT** include countermeasures against:

- **Timing attacks**: Execution time depends on the exponent bits
- **Power analysis attacks**: Power consumption varies with operations
- **Fault injection attacks**: No error detection/correction

For production use, consider:
- Exponent blinding (randomize with `m' = m × r^e mod n`)
- Constant-time algorithms
- Side-channel resistant implementations
- Key sizes ≥ 2048 bits (this project uses 128 bits for demonstration)

## Building

### Prerequisites

- **Xilinx Vivado 2022.2** (or later, 64-bit)
- **Xilinx Artix-7 FPGA board** (e.g., Digilent Arty A7, Basys 3)
- Sufficient DSP resources for modular multiplication

### Compilation Steps

1. Open **Vivado 2022.2**
2. Select **File → Open Project**
3. Open `rsa.xpr`
4. Select **Flow → Run Synthesis**
5. After synthesis, select **Flow → Run Implementation**
6. Generate bitstream with **Flow → Generate Bitstream**
7. Program the Artix-7 board

### Device Configuration

| Parameter | Value |
|-----------|-------|
| **Family** | Artix-7 |
| **Device** | XC7A35T |
| **Package** | CPG236 |
| **Speed Grade** | -1 |
| **Vivado Version** | 2022.2 |
| **Top-Level Entity** | RSACypher |

### Resource Usage (Estimated)

| Resource | Estimated Usage |
|----------|----------------|
| **LUTs** | ~10-15% of available |
| **FFs** | ~5-10% of available |
| **DSPs** | 0 (all arithmetic in LUTs/registers) |
| **BRAM** | Minimal (only registers used) |

Note: The implementation uses only sequential logic (LUTs and flip-flops) — no dedicated DSP blocks are required. The 128-bit multipliers are implemented using LUT-based arithmetic.

## Algorithm Details

### Square-and-Multiply (Right-to-Left)

```
function modexp(m, e, n):
    result ← 1
    m ← m mod n
    while e > 0:
        if (e AND 1) == 1:
            result ← (result × m) mod n
        e ← e >> 1
        m ← (m × m) mod n
    return result
```

The hardware implementation processes exponent bits from LSB to MSB, with:
- **`root`** register holding `m^(2^i) mod n` at iteration `i`
- **`temp`** register accumulating the result
- Two parallel modular multipliers for squaring and conditional multiplication

### Modular Multiplication

The `modmult32` module implements a **restoring algorithm**:
1. Multiply the two operands → 256-bit partial product
2. Iteratively subtract the modulus (aligned by shift)
3. If result is negative, add back the modulus (restore)
4. Final value in the low 128 bits is the result

## VHDL Conventions

- **IEEE 1164** — Standard logic types
- **IEEE.STD_LOGIC_ARITH / STD_LOGIC_UNSIGNED** — Arithmetic operations (note: non-standard, use `numeric_std` in production)
- **Generic parameters** — Configurable key sizes
- **Behavioral style** — Descriptive signal naming

## References

- Rivest, R. L., Shamir, A., & Adleman, L. (1978). "A Method for Obtaining Digital Signatures and Public-Key Cryptosystems"
- Xilinx PG035: Artix-7 Data Sheet
- Xilinx UG687: Vivado Design Suite User Guide
- Kocher, P. (1996). "Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems"

> *Note: This README was generated by AI to describe the project. The source code itself was written by the user.*
