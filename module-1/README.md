# 🔧 Module 1: Environment Setup (Just Do It™)

> **⚠️ Disclaimer:** This guide provides approximate installation steps. Your environment may differ, and some commands might not work as-is. **Your job is to connect the dots** and adapt these instructions to your specific setup.

## 🎯 Goal

Install these core dependencies for Solana development:

- **Node.js/TypeScript** (for web3 client development)
- **Rust** (for smart contract development)
- **Solana CLI** (for blockchain interaction)
- **Anchor v0.29.0** (for smart contract framework)

The ultimate goal: Get Anchor 0.29.0 working so you can start building.

---

Let's get your development environment ready. No fluff—only the official installers.

---

## 📦 1. Rust (stable toolchain)

Run the official installer:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env
```

Verify:

```bash
rustc --version
```

---

## 🛠 2. Solana CLI

Run the official script:

```bash
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

Ensure the CLI is on your PATH:

```bash
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```

Verify:

```bash
solana --version  [oai_citation:0‡Solana](https://solana.com/docs/intro/installation?utm_source=chatgpt.com) [oai_citation:1‡DEV Community](https://dev.to/sufferer/the-complete-guide-to-full-stack-solana-development-with-nextjs-anchor-and-phantom-4180?utm_source=chatgpt.com) [oai_citation:2‡Helius](https://www.helius.dev/blog/an-introduction-to-anchor-a-beginners-guide-to-building-solana-programs?utm_source=chatgpt.com) [oai_citation:3‡Solana](https://solana.com/docs/toolkit/getting-started?utm_source=chatgpt.com)
```

---

## 🌐 3. Node.js (and Yarn via Corepack)

Install via nvm:

```bash
curl -o‑ https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install --lts
nvm use --lts
```

Enable Yarn:

corepack enable

Verify versions:

node --version && npm --version && yarn --version

---

## ⚓️ 4. Anchor CLI v0.29.0

Install Anchor Version Manager:

```bash
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
```

Then install and select 0.29.0 explicitly:

```bash
avm install 0.29.0
avm use 0.29.0
```

Verify:

```bash
anchor --version  [oai_citation:4‡anchor-lang.com](https://www.anchor-lang.com/docs/updates/release-notes/0-29-0?utm_source=chatgpt.com) [oai_citation:5‡Medium](https://medium.com/%40cryptowikihere/welcome-to-solana-rust-set-up-guide-installing-solana-cli-and-test-run-29b8896537e1?utm_source=chatgpt.com) [oai_citation:6‡Helius](https://www.helius.dev/blog/an-introduction-to-anchor-a-beginners-guide-to-building-solana-programs?utm_source=chatgpt.com)
```

---

## ✅ 5. Verify All Tools

```bash
rustc --version
solana --version
node --version
npm --version
anchor --version
```

Each should print a valid version number.

---

## 🎯 What’s Next?

If everything is installed, you're ready to jump into [Module 2](../module-2/README.md) — building your first real Solana program.

---

## 📚 References

- Official Solana CLI installation guide  [Solana](https://solana.com/docs/intro/installation?utm_source=chatgpt.com) [Medium](https://medium.com/%40cryptowikihere/welcome-to-solana-rust-set-up-guide-installing-solana-cli-and-test-run-29b8896537e1?utm_source=chatgpt.com) [Stack Overflow](https://stackoverflow.com/questions/77195103/solana-installation?utm_source=chatgpt.com)
- Anchor v0.29.0 release notes & tooling  [Anchor lang](https://www.anchor-lang.com/docs/updates/release-notes/0-29-0?utm_source=chatgpt.com)
- Rust installation using rustup ()
- Anchor AVM installer details  [docs.soo.network](https://docs.soo.network/developers/install-anchor-cli?utm_source=chatgpt.com)
