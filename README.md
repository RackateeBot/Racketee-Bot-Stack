# Racketee-Bot-Stack
Rackateebot is a Rust arbitrage bot for Solana. It streams live DEX pool data, scans for cross-DEX price gaps and multi-hop loops, then checks slippage, risk limits and fresh reserves before acting. Trades go out as atomic Jito private bundles with profit-scaled, randomized tips, optionally raced against SWQoS senders. Includes metrics and dry-run.
