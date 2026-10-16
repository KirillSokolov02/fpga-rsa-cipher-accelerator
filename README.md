# 128-Bit RSA Hardware Accelerator — Modular Exponentiation on Artix-7

Verilog implementation of a hardware RSA cipher accelerator performing 128-bit modular exponentiation using square-and-multiply algorithm. Built for Xilinx Artix-7.

## Overview

This project implements a complete RSA encryption/decryption engine in hardware. The accelerator performs `c = m^e mod n` for 128-bit operands using a LUT-based modular multiplication unit, demonstrating cryptographic hardware acceleration without DSP slices.

## Project Structure

| File | Description |
|------|-------------|
| `modmult.vhd` | LUT-based modular multiplication: computes (a * b) mod n using square-and-multiply |
| `rsacypher.vhd` | Top-level RSA engine: orchestrates modular multiplication for exponentiation |

## Hardware

- **Device:** Xilinx Artix-7 XC7A35T (xc7a35tcpg236-1)
- **Tool:** Vivado 2022.2
- **Language:** VHDL-2008

## Architecture

**Square-and-Multiply Algorithm:**

The accelerator iterates through bits of the exponent, performing:
- Modular squaring of the base at each bit position
- Conditional modular multiplication when the exponent bit is 1

Both operations use the same `modmult` unit, which implements Montgomery-style modular multiplication using LUT logic (zero DSP usage). The design features pipelined multiply operations with ready/valid handshaking.

## Building

1. Open `rsa.xpr` in Vivado
2. Run Synthesis and Implementation
3. Generate bitstream
