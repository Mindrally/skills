---
name: solidity
description: "Best practices for secure, gas-efficient Solidity smart contract development and Web3 frontend integration. Use when writing or reviewing Solidity contracts, hardening against reentrancy and access-control bugs, optimizing gas usage, setting up Hardhat/Foundry testing and static analysis, or wiring a React frontend to wallets, providers, and on-chain transactions."
---

# Solidity

This skill covers best practices for Solidity smart contract development, including security patterns, gas optimization, testing/tooling, and integrating contracts with a Web3 React frontend.

## Core Principles

- Cut the fluff. Code or detailed explanations only
- Maintain brevity while prioritizing accuracy and depth
- Answer first, explain later when needed

## Code Structure & Security

- Use explicit visibility modifiers and NatSpec documentation
- Apply function modifiers to reduce redundancy
- Follow naming conventions:
  - CamelCase for contracts
  - PascalCase for interfaces (prefix with "I")
- Implement Interface Segregation Principle
- Use proxy patterns for upgradeable contracts
- Emit comprehensive events for state changes
- Follow Checks-Effects-Interactions pattern against reentrancy

## Security Best Practices

- Use OpenZeppelin's AccessControl for permissions
- Require Solidity 0.8.0+ for overflow/underflow protection
- Use Pausable pattern for circuit breakers
- Implement ReentrancyGuard for additional protection
- Use SafeERC20 for token interactions
- Employ pull-over-push payment patterns
- Implement timelocks and multisig controls for sensitive operations

## Gas Optimization

- Optimize gas consumption (deployment and runtime)
- Use immutable variables for constructor-set values
- Use custom errors instead of revert strings
- Pack storage variables efficiently
- Use appropriate data types

## Tools & Analysis

- Integrate Slither and Mythril for static analysis
- Leverage Hardhat's testing and development environment
- Implement robust CI/CD pipelines
- Use pre-commit linting tools

## Advanced Patterns

- Chainlink VRF for randomness
- Strategic assembly use with extensive documentation
- State machine patterns for complex logic
- ERC20Snapshot, ERC20Permit, and ERC20Votes for specialized tokens

## Testing & Quality

- Comprehensive unit, integration, and end-to-end testing
- Property-based testing approaches
- High coverage targets
- Regular security audits

## Web3 Frontend Integration

- Keep frontend code that talks to contracts type-safe and explicit — generate TypeScript types from ABIs (e.g. via TypeChain or wagmi/viem codegen) rather than hand-writing contract call signatures
- Treat wallet connection, chain/network state, and transaction status as explicit, typed state (connected/connecting/wrong-network/error), not implicit booleans
- Surface transaction lifecycle clearly in the UI: submitted, pending confirmation, confirmed, reverted — never assume a submitted transaction succeeded
- Validate and simulate transactions (e.g. `eth_call`/gas estimation) before prompting the user to sign, to catch reverts early
- Never trust client-side reads of contract state for authorization decisions — always re-verify on-chain in the contract itself
- Handle provider/RPC failures and user rejection of signature requests as expected, recoverable error states
