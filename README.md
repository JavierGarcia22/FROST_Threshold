# FROST Threshold Signatures Implementation

This repository contains a practical implementation of FROST (Flexible Round-Optimized Schnorr Threshold) signatures for authentication. The implementation demonstrates a T-out-of-N threshold signature scheme using embedded development boards and a coordinating computer.

## Overview

The system implements threshold signatures where:
- **3 participants** hold secret shares of a signing key
- **2 participants** are required to generate a valid signature (threshold)
- A computer acts as **coordinator**, either as a **trusted dealer** or as one of the parties in a **distributed key generation (DKG)**
- Communication uses **UART** (Nucleo-L476RG) and **USB HID** (OFFPAD)

The implementation follows the FROST protocol with two main phases:
1. **Key Generation**: secret shares are created and distributed to the boards, either by a trusted dealer (the computer) or through DKG, where shares are exchanged directly between participants and encrypted with ElGamal so the computer never sees a plaintext share
2. **Signing**: boards collaborate to sign messages using threshold cryptography, either one at a time or with all boards broadcasting simultaneously

## Directory Structure

```
├── presign_pc, receive_usb_*/                # Trusted-dealer key generation and distribution (PC + boards)
├── presign_pc_dkg_ElGamal/,                  # Distributed Key Generation with ElGamal-encrypted shares
│   presign_nucleo_dkg_ElGamal/,              
│   presign_offpad_dkg_ElGamal/               
├── presign_pc_dkg_ElGamal_blame/,            # Distributed Key Generation with ElGamal-encrypted shares and blame protocol
│   presign_nucleo_dkg_ElGamal_blame/,              
│   presign_offpad_dkg_ElGamal_blame/               
├── signing_pc*/,                             # FROST signing (sequential PC, broadcast PC, boards)
│   signing_nucleo/, 
│   signing_offpad/   
├── *_tests/                                   # Standalone/Zephyr tests mirroring the directories above
├── external/modules/crypto/secp256k1-frost/   # secp256k1 + FROST/ElGamal library used by all boards
└── west-manifest/west.yml                     # Zephyr/West manifest pinning the RTOS version and modules
```

## Hardware Requirements

- **Computer**: Windows machine with GCC compiler
- **STM32 Nucleo-L476RG**: Development board with UART communication
- **OFFPAD**: Development board with USB HID communication
- **USB cables**: For board connections and power

## Dependencies

- **secp256k1** (with the FROST and ElGamal/ECDH extensions vendored in `external/modules/crypto/secp256k1-frost`): Elliptic curve cryptography library
- **Zephyr RTOS**: For embedded board firmware
- **West**: Zephyr build tool
- **GCC**: For PC compilation

## Build Instructions

### 1. Computer Firmware

Navigate to the relevant directory and build the executable:

#### Computer (Trusted-Dealer Key Generation)
```bash
cd presign_pc/src
gcc -g main.c -lsecp256k1 -lsetupapi -lhid -o keygen.exe
```

#### Computer (DKG Coordinator)
```bash
cd presign_pc_dkg_ElGamal/src
gcc -g main.c -lsecp256k1 -lsetupapi -lhid -o dkg_keygen.exe
```
Use `presign_pc_dkg_ElGamal_blame/src` instead if you want blame-protocol support.

#### Computer (Signing)
```bash
cd signing_pc/src
gcc -g main.c -lsecp256k1 -lsetupapi -lhid -o main.exe
```
Or, for the broadcast coordinator:
```bash
cd signing_pc_broadcast
gcc -g main.c -lsecp256k1 -lsetupapi -lhid -o frost_coordinator.exe
```

**Required libraries:**
- `secp256k1`: Cryptographic operations
- `setupapi`: Windows device enumeration
- `hid`: USB HID communication

### 2. Board Firmware

For each board directory, build and flash the firmware using West:

#### Nucleo Board (Key Reception, Trusted Dealer)
```bash
cd receive_usb_nucleo
west build
west flash
```

#### OFFPAD Board (Key Reception, Trusted Dealer)
```bash
cd receive_usb_offpad
west build
west flash
```

#### Nucleo Board (DKG Participant)
```bash
cd presign_nucleo_dkg_ElGamal
west build
west flash
```
Use `presign_nucleo_dkg_ElGamal_blame` instead if you want blame-protocol support.

#### OFFPAD Board (DKG Participant)
```bash
cd presign_offpad_dkg_ElGamal
west build
west flash
```
Use `presign_offpad_dkg_ElGamal_blame` instead if you want blame-protocol support.

#### Nucleo Board (Signing)
```bash
cd signing_nucleo
west build
west flash
```

#### OFFPAD Board (Signing)
```bash
cd signing_offpad
west build
west flash
```

## Running the Implementation

### Phase 1: Key Generation and Distribution

Choose one of the two key setup flows below.

#### Option A: Trusted Dealer

1. **Flash key reception firmware** to all boards:
   - Flash `receive_usb_nucleo` to Nucleo-L476RG
   - Flash `receive_usb_offpad` to OFFPAD

2. **Run key generation** on computer:
   ```bash
   cd presign_pc/src
   ./keygen.exe
   ```

3. **Select communication method** for each board:
   - Choose UART for Nucleo (specify COM port, e.g., COM4)
   - Choose USB HID for OFFPAD

4. **Key distribution**: The computer will generate secret shares and distribute them to each board using their respective communication protocols.

#### Option B: Distributed Key Generation (DKG) with ElGamal Encryption

1. **Flash DKG firmware** to all boards:
   - Flash `presign_nucleo_dkg_ElGamal` (or the `_blame` variant) to Nucleo-L476RG
   - Flash `presign_offpad_dkg_ElGamal` (or the `_blame` variant) to OFFPAD

2. **Run the DKG coordinator** on computer:
   ```bash
   cd presign_pc_dkg_ElGamal/src
   ./dkg_keygen.exe
   ```

3. **DKG round**: Each participant generates an ElGamal keypair and exchanges public keys, then computes DKG commitments and shares. Shares are encrypted under each recipient's ElGamal public key before being sent, decrypted only by the intended recipient, and used to finalize each board's own secret share. The computer relays messages but never sees a plaintext share.

4. **(Blame variant only)**: If a board reports an invalid share, the coordinator can request blame proofs to identify the misbehaving participant instead of the run simply failing.

### Phase 2: Signing

1. **Flash signing firmware** to boards:
   - Flash `signing_nucleo` to Nucleo-L476RG
   - Flash `signing_offpad` to OFFPAD

2. **Run signing coordinator** on computer:
   ```bash
   cd signing_pc/src
   ./main.exe
   ```
   Or, for the broadcast coordinator, which contacts all boards at once instead of one at a time:
   ```bash
   cd signing_pc_broadcast
   ./frost_coordinator.exe
   ```

3. **Threshold signing**: The coordinator will:
   - Send ready signal to participants
   - Collect nonce commitments (Round 1)
   - Send message hash and commitments (Round 2)
   - Aggregate signature shares into final signature

## Important UART Configuration

**Critical**: For UART communication, you must update the UART configuration in the board code to match your specific board setup.

In the Nucleo firmware files, modify the UART device node:
```c
#define UART_DEVICE_NODE DT_NODELABEL(usart1)  // Change as needed
```

Check your board's pin configuration and update accordingly.

## Communication Protocols

### UART (Nucleo-L476RG)
- **Baud Rate**: 115200
- **Data Bits**: 8
- **Stop Bits**: 1
- **Parity**: None
- **Flow Control**: None

### USB HID (OFFPAD)
- **Vendor ID**: 0x2FE3
- **Product ID**: 0x0100
- **Report Size**: 64 bytes
- **Communication**: Bidirectional, with larger DKG/signing messages chunked and reassembled

## Protocol Flow

### Key Generation (Trusted Dealer)
1. Computer generates master secret key
2. Creates secret shares using Shamir's secret sharing
3. Distributes shares to boards via UART/USB HID
4. Boards verify and store shares in flash memory

### Key Generation (DKG with ElGamal-Encrypted Shares)
1. Computer distributes a shared DKG context to all participants
2. Each participant generates an ElGamal keypair and exchanges public keys through the computer
3. Each participant generates DKG commitments and shares, encrypting each share under the recipient's ElGamal public key
4. Each participant decrypts the shares addressed to it, validates them, and finalizes its own secret share
5. (Blame variant) If validation fails, participants can trigger a blame exchange to identify who sent an invalid share

### Signing Process
1. **Round 1**:
   - Computer sends ready signal, to one board at a time or all boards at once
   - Boards generate nonces and commitments
   - Computer collects commitments

2. **Round 2**:
   - Computer sends message hash and all commitments
   - Boards compute signature shares
   - Computer aggregates shares into final signature

## Security Features

- **Secret shares never leave boards** after distribution or DKG finalization
- **DKG shares are ElGamal-encrypted in transit**, so the computer relays but never reads a plaintext share
- **Optional blame protocol** to identify a misbehaving participant during DKG instead of silently aborting
- **Commitment verification** before signature computation
- **Threshold security**: Requires 2 out of 3 participants

## Troubleshooting

### Common Issues

1. **UART Connection Failed**:
   - Verify COM port number
   - Check UART_DEVICE_NODE configuration
   - Ensure proper baud rate (115200)

2. **USB HID Device Not Found**:
   - Check USB cable connection
   - Verify VID/PID in device manager
   - Try different USB port

3. **Build Failures**:
   - Ensure West and Zephyr SDK are properly installed
   - Check secp256k1 library installation, including the FROST/ElGamal extensions
   - Verify GCC and development tools

4. **Flash Storage Issues**:
   - Ensure flash partition is properly configured
   - Check available flash memory
   - Verify flash area permissions

5. **DKG Fails Partway Through**:
   - Confirm every participant flashed a matching DKG firmware variant (all `_blame` or all non-`_blame`)
   - If using the `_blame` variant, check the coordinator's blame-proof output to see which participant was implicated
   - Re-run DKG from scratch rather than resuming, since shares are only valid for one completed DKG context