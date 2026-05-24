---
name: claw-deepseek-fix
version: 1.0.0
description: Patch for Claw Code fixing DeepSeek V4 API — omits empty tool_calls, preserves reasoning_content.
author: nerudek
license: MIT
compatible-with: claw-code, deepseek-api
---

## Install

git clone https://github.com/nerudek/claw-deepseek-fix
cd claw-deepseek-fix/rust && cargo build -p rusty-claude-cli --release

## Usage

export OPENAI_BASE_URL=https://api.deepseek.com/v1
export OPENAI_API_KEY=your_key
./rust/target/release/claw

