# Hello FHEVM — Beginner-friendly README

**Goal:** A step-by-step, beginner-friendly guide that helps a developer build and run a complete dApp (smart contract + frontend) using Zama’s FHE tools (fhEVM, fhevm Hardhat template, and the relayer / fhevmjs SDK). By the end, you’ll be able to: compile an `FHECounter` contract, encrypt inputs in the frontend, call the contract with encrypted inputs, and run everything locally or on Sepolia. This README is written so you can copy-paste it into the terminal in your project folder.

---

## TL;DR (quick commands)

```bash
# clone this repo (replace with your repo)
git clone https://github.com/YOUR-USER/YOUR-REPO.git
cd YOUR-REPO

# backend/contracts (hardhat)
cd hardhat
npm install
npx hardhat compile
# start a local FHE-ready node and deploy locally
npm run hardhat-node   # opens local node (mock encryption)
npm run deploy:localhost

# frontend (react)
cd ../frontend
npm install
npm run dev:mock   # run frontend connected to the local mock node
```

> When your dApp is stable, follow the **Sepolia** deploy steps below to deploy with real encryption.

---

## What you'll build (high level)

A tiny, reproducible dApp:

* **Smart contract**: `FHECounter.sol` (an encrypted counter that stores `euint32` and exposes `increment()` and `decrement()` which accept encrypted inputs).
* **Frontend**: Minimal React page that:

  * Connects to a wallet (MetaMask or compatible provider)
  * Initializes the FHE SDK in the browser (WASM modules)
  * Encrypts a number off-chain, produces the required proof, and calls `increment(externalEuint32, proof)` on-chain
  * Reads encrypted state and (optionally) decrypts it client-side

This mirrors the official Zama `fhevm-hardhat-template` + `fhevm-react-template` flow, but simplified for beginners.

---

## Prerequisites

* Node.js LTS (v18 or v20 recommended)
* npm (or yarn/pnpm)
* Git
* A browser with MetaMask installed (for Sepolia or local dev testing)
* (Optional) An Infura or Alchemy API key for Sepolia deployments
* Basic Solidity knowledge (writing & compiling simple contracts)

---

## Project layout (suggested)

```
hello-fhevm/
├── hardhat/            # based on fhevm-hardhat-template
│   ├── contracts/
│   │   └── FHECounter.sol
│   ├── scripts/
│   ├── test/
│   └── package.json
├── frontend/           # based on fhevm-react-template
│   ├── packages/site/
│   ├── pages/
│   └── package.json
└── README.md           # this file
```

> Tip: You can use the official templates as a starting point and copy the minimal files from them into your repo.

---

## Smart contract: `FHECounter.sol` (complete example)

Create `hardhat/contracts/FHECounter.sol` and paste the code below. This example is the minimal FHE-enabled counter adapted for clarity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { FHE, euint32, externalEuint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/// @title A simple FHE counter contract
contract FHECounter is SepoliaConfig {
  euint32 private _count;

  /// @notice Returns the current (encrypted) count
  function getCount() external view returns (euint32) {
    return _count;
  }

  /// @notice Increments the counter by an encrypted value
  function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
    euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
    _count = FHE.add(_count, evalue);

    // allow decryption permissions for the result
    FHE.allowThis(_count);
    FHE.allow(_count, msg.sender);
  }

  /// @notice Decrements the counter by an encrypted value
  function decrement(externalEuint32 inputEuint32, bytes calldata inputProof) external {
    euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
    _count = FHE.sub(_count, evalue);

    FHE.allowThis(_count);
    FHE.allow(_count, msg.sender);
  }
}
```

**Why this is beginner-friendly:** the contract hides the crypto details — it accepts `externalEuint32` and `inputProof`. You generate those off-chain with the SDK and pass them as normal call arguments.

---

## Hardhat (local dev & tests)

We recommend starting from the official fhEVM Hardhat template. It already includes the necessary plugin and helper `fhevm` in the Hardhat Runtime Environment (HRE).

### Setup (hardhat folder)

```bash
cd hardhat
npm install
# set environment variables (MNEMONIC is optional for local dev)
# You can use: npx hardhat vars set MNEMONIC "test test ..."

npx hardhat compile
```

### Run a local FHE-ready node (mock encryption — fast)

```bash
# in hardhat/  (the template exposes scripts)
npm run hardhat-node
# in a separate terminal, deploy to the local node
npm run deploy:localhost
```

The mock mode is fast and ideal for iterative development. For real encryption testing you will later deploy to Sepolia (see below).

### Example test that encrypts an input (Hardhat script)

A minimal TypeScript/Javascript test uses the `fhevm` helper from the Hardhat environment:

```js
// test/fheCounter.test.js (pseudo-code)
const { ethers } = require('hardhat');

it('encrypts and increments', async () => {
  const [alice] = await ethers.getSigners();

  // deploy FHECounter
  const FHECounter = await ethers.getContractFactory('FHECounter');
  const counter = await FHECounter.deploy();
  await counter.deployed();

  // create encrypted input via the Hardhat FHE helper
  const input = fhevm.createEncryptedInput(counter.address, alice.address);
  input.add32(5); // add a single uint32 value
  const encryptedInputs = await input.encrypt();

  // encryptedInputs.handles[0] is the externalEuint32 value
  // encryptedInputs.proof is the inputProof bytes
  await counter.increment(encryptedInputs.handles[0], encryptedInputs.proof);
});
```

> The Hardhat helper handles key generation and proof creation for local tests in mock mode.

---

## Frontend (React) — encrypting on the client and calling the contract

This section explains how to encrypt a value in the browser using the FHE SDK and submit the transaction to the contract.

### Install SDK (frontend)

In your frontend project:

```bash
npm install @zama-fhe/relayer-sdk fhevmjs
# or: npm i @zama-fhe/relayer-sdk
```

### Browser initialization (frontend snippet)

```js
// src/lib/fhe.js
import { initFhevm, createInstance } from 'fhevmjs';

export async function initFHE(provider) {
  // Loads WASM TFHE modules (must await)
  await initFhevm();

  // create an instance connected to the user's wallet provider (e.g., MetaMask)
  const instance = await createInstance({
    network: provider, // window.ethereum
    gatewayUrl: process.env.NEXT_PUBLIC_FHEVM_GATEWAY_URL || 'https://gateway.zama.ai',
  });

  return instance;
}
```

### Create encrypted input & call contract (frontend)

```js
// Example React handler that encrypts a number and calls increment()
import { ethers } from 'ethers';
import { initFHE } from '../lib/fhe';

async function onIncrement(value, contractAddress) {
  const provider = window.ethereum;
  const signer = (new ethers.providers.Web3Provider(provider)).getSigner();

  // init the FHE SDK in browser
  const instance = await initFHE(provider);

  // create an encrypted input bound to the contract and signer
  const input = await instance.createEncryptedInput(contractAddress, await signer.getAddress());
  input.add32(Number(value));
  const encryptedInputs = await input.encrypt();

  // encryptedInputs.handles[0] and encryptedInputs.proof match the contract signature
  const externalHandle = encryptedInputs.handles[0];
  const inputProof = encryptedInputs.proof;

  // call the contract (use ethers.Contract connected to signer)
  const contract = new ethers.Contract(contractAddress, FHECounterABI, signer);
  const tx = await contract.increment(externalHandle, inputProof);
  await tx.wait();
}
```

**Notes:**

* `createEncryptedInput(...)` returns a builder where you `add32()`, `add64()` etc. and then call `.encrypt()` which produces the `handles` array and `proof` bytes required by the contract.
* Loading the WASM files can be slower in some build setups (Vite / Next). The templates already shard the WASM and loading is handled by the SDK; if you make a custom setup follow the template’s approach.

---

## Deploy to Sepolia (real encryption)

**Important:** Sepolia is the network where the fhEVM coprocessor was deployed for testing. Deploying there uses real encryption and is slower than mock mode.

### Steps

1. Add environment variables in `hardhat`:

   * `MNEMONIC` (or private key)
   * `INFURA_API_KEY` (or Alchemy key)
2. In `hardhat/` run:

```bash
npm run compile
npm run deploy:sepolia
```

3. The deploy script prints the contract address. Copy this to your frontend `.env` (e.g., `NEXT_PUBLIC_CONTRACT_ADDRESS`).
4. In the frontend, run the app pointing to Sepolia (and make sure your provider is connected to Sepolia in MetaMask).

> You will need Sepolia ETH for test transactions.

---

## Deploy frontend to Vercel (optional)

1. Push your code to GitHub.
2. Go to Vercel and create a new project connected to your repo.
3. Set these environment variables in Vercel Settings (Project → Environment Variables):

   * `NEXT_PUBLIC_FHEVM_GATEWAY_URL` (e.g. `https://gateway.zama.ai`)
   * `NEXT_PUBLIC_CONTRACT_ADDRESS`
   * `INFURA_API_KEY` (if your frontend needs provider-based access; usually frontend uses user wallet so it's optional)
4. Build command: `npm run build` (or the template default). Start command: `npm start` or `next start`.

Because WASM assets are required for fhevmjs, ensure your build copies the WASM files to the `public/` directory (the templates already take care of this).

---

## How the full encryption → compute → decrypt flow works (simple)

1. **Client**: Loads fhevmjs, creates an encrypted input (the SDK loads TFHE WASM and uses public keys). The client produces:

   * `handles` (external handles referencing ciphertexts)
   * `proof` (ZK proof binding the ciphertext to the caller & contract)
2. **Client → Contract**: Client calls `contract.increment(externalHandle, proof)` (on Sepolia this includes real ciphertext and proof).
3. **Contract**: Inside `increment()` the contract calls `FHE.fromExternal(handle, proof)` to validate and convert the handle into an on-chain encrypted type (`euint32`). The contract computes on `euint32` values using FHE operations like `FHE.add()` / `FHE.sub()`.
4. **Grant permission**: Contract grants `FHE.allow()` to the appropriate actors so they can request decryption of the new state.
5. **Client or off-chain worker**: Uses the gateway/relayer to re-encrypt or decrypt using the granted permission (the SDK exposes reencrypt/decrypt helpers).

This means **plaintext never touches the chain** — computations happen on ciphertexts.

---

## Troubleshooting & Tips

* **Editor opens during `git merge` or `pull`**: If Git opens an editor asking for a commit message, save & exit (Vim: `:wq`, Nano: `Ctrl+O`, `Enter`, `Ctrl+X`). Or use `--no-edit` on merge commands.
* **WASM loading issues in the browser**: Use the official templates, they already handle WASM bundling. If you build your own, ensure WASM files are served under `public/` and the SDK is pointed to them.
* **MetaMask + Hardhat nonce mismatch**: If you see nonce errors when restarting Hardhat, clear MetaMask activity tab or restart the extension.
* **Version mismatches**: If you use older/experimental testnets, you may need specific versions of `fhevm` and `fhevmjs`. Check the community if you hit runtime `unreachable` errors.

---

## What to include in your Git repo for a submission

* `hardhat/` folder (contracts, deploy scripts, tests)
* `frontend/` folder (React app with the FHE client snippets)
* A `README.md` (this file)
* A short demo video or screenshots (optional but helpful)
* (Optional) A link to a deployed frontend on Vercel and the Sepolia contract address

---

## Where I got the references (official docs & templates)

This guide follows the official Zama fhEVM templates and docs (Hardhat + React templates) and the `fhevmjs` SDK. If you want to dig deeper, look at the official:

* `fhevm-hardhat-template` (Hardhat starter with FHE integration)
* `fhevm-react-template` (minimal React frontend to interact with FHE contracts)
* Zama Protocol docs: FHEVM guides (how to turn a contract into an FHE contract and how encrypted inputs work)

---

## Next steps I can do for you (pick any)

* Add a polished `FHECounter` frontend page (React component) ready to drop into `frontend/`.
* Create `hardhat` deploy scripts & minimal tests and add them to the repo.
* Add a one-click Vercel + Sepolia deployment guide (with required environment variables and secrets).

---

Good luck — paste this README into your repo's `README.md` and tell me if you want the full code files (deploy scripts, the React page, or a ready-to-deploy repo). Happy to generate the exact files next.
