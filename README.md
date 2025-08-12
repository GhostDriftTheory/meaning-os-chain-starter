# Meaning OS × Blockchain — A-grade Starter

[![CI](https://github.com/<OWNER>/<REPO>/actions/workflows/test.yml/badge.svg)](https://github.com/<OWNER>/<REPO>/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**What you get**
- SHPRegistry（Commit→Reveal→OCW→Finalize→JumpLog）
- MinorityGuard712（EIP-712マルチシグでΓ-floor／OCW延長）
- 最小CLI（commit / reveal / sign-gamma / post-gamma / finalize）
- Foundryテスト（不変量＆ファズ）

## Quick Start
```bash
# Hardhat（任意）でコンパイル
npm i -D hardhat @nomicfoundation/hardhat-toolbox && npx hardhat compile

# guardian付きデプロイ例
node scripts/deploy_with_guardian_712.js   # Hardhat環境なら

# CLI
npm i
# 1) 準備
node cli/cli.js prepare --id case1 --s "S*" --g "g^" --dphi -1 --ocw 3600
# 2) commit / 3) reveal
node cli/cli.js commit --id case1
node cli/cli.js reveal --id case1
# 4) 署名者2名でEIP-712署名（鍵を切替えて実行）
node cli/cli.js sign-gamma --id case1 --mass 7 --nonce 1
# 5) Γ投稿（署名2つを渡す）
node cli/cli.js post-gamma --id case1 --mass 7 --nonce 1 --sigs 0xSigA,0xSigB
# 6) 締切チェック→期限後に finalize
node cli/cli.js deadline --id case1
node cli/cli.js finalize --id case1

## Security Model（要点）
- φ/g の評価は **オフチェーン**。オンチェーンは **手続と証跡** を保証（Commit–Reveal、不可逆JumpLog、OCW、Γ-floor）。
- `finalize` は **「無チャレンジ ∧ Δφ < 0」** のときだけ accepted。履歴はイベントで改変不可能。
- Γ-floor は **EIP-712 署名の閾値承認（θ）** で OCW を延長。**`nonce`** でリプレイ防止。

## Roadmap
- SNARK/STARK による `proof` のオンチェーン検証（現状はハッシュ/URIアンカー）。
- 最小Web UI（1画面：commit / reveal / Γ署名収集 / postGamma / finalize）。
- 署名の重み付きマルチシグ、または DAO ガバナンス接続（θや延長秒のオンチェーン投票）。

