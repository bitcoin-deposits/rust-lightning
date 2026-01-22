# Bitcoin Deposits Protocol - rust-lightning Changes

This document summarizes the changes made to rust-lightning to support the Bitcoin Deposits protocol. These changes implement a generic "commitment extra outputs" mechanism that allows external code to add outputs to Lightning commitment transactions.

## Overview

The Bitcoin Deposits protocol requires the ability to add special outputs to commitment transactions that serve as on-chain reserves backing off-chain deposits. Rather than implementing deposits-specific logic directly in rust-lightning, we've added a generic hook-based mechanism that can be used by any protocol extension.

**Total changes:** ~917 lines added across 5 files

## Commits

### 1. Add generic commitment extra outputs mechanism
`7c0cdf2c2`

Core infrastructure for adding extra outputs to commitment transactions.

**chan_utils.rs changes:**
- Added `CommitmentExtraOutput` struct representing an output to be added
- Added `CommitmentTransaction::new_with_extra_outputs()` constructor
- Added `build_outputs_and_htlcs_with_extra()` for building transactions with extra outputs

**channel.rs changes:**
- Added channel state fields for tracking extra outputs:
  - `holder_extra_outputs` / `counterparty_extra_outputs` (confirmed outputs)
  - `pending_inbound_extra_outputs` (proposals awaiting validation)
  - `pending_outbound_extra_outputs` (proposals awaiting acceptance)
  - `holding_cell_extra_outputs` (queued when channel is busy)
- Added `FundedChannel` methods:
  - `propose_extra_outputs()` - initiate a proposal
  - `receive_extra_outputs_proposal()` - handle incoming proposal
  - `accept_extra_outputs_proposal()` / `reject_extra_outputs_proposal()`
  - `extra_outputs_accepted()` - handle counterparty acceptance
  - `clear_extra_outputs()` - remove all extra outputs
- Integrated extra outputs into commitment transaction building

**tx_builder.rs changes:**
- Added `TxBuilder::build_commitment_transaction_with_extra_outputs()`

### 2. Add ChannelManager API and ExtraOutputsProposed event
`142e4a799`

Public API for managing extra outputs from external code.

**channelmanager.rs changes:**
- `propose_extra_outputs()` - propose extra outputs for a channel
- `receive_extra_outputs_proposal()` - handle counterparty proposals
- `accept_extra_outputs_proposal()` - accept a pending proposal
- `reject_extra_outputs_proposal()` - reject a pending proposal
- `extra_outputs_accepted()` - handle counterparty acceptance
- `get_channel_extra_outputs()` - query current extra outputs
- `clear_channel_extra_outputs()` - clear all extra outputs

**events/mod.rs changes:**
- Added `Event::ExtraOutputsProposed` - emitted when counterparty proposes extra outputs, allowing external code to validate and respond

### 3. Account for extra outputs in channel capacity calculations
`40344e25b`

Ensures channel balance calculations correctly reflect extra outputs.

**channel.rs changes:**
- Modified balance calculations to subtract extra output values:
  - Holder extra outputs reduce local balance
  - Counterparty extra outputs reduce remote balance
- Uses "effective" outputs (pending if exists, otherwise confirmed) to reflect actual committed state

## Usage by deposits-ldk

The `deposits-ldk` crate uses these hooks to implement Bitcoin Deposits reserves:

1. When reserves need to be added, `deposits-ldk` calls `propose_extra_outputs()` with the reserves output (a Taproot output committing to the ledger state)

2. When a counterparty proposes extra outputs, `deposits-ldk` receives the `ExtraOutputsProposed` event, validates the proposal against its ledger state, and calls `accept_extra_outputs_proposal()` or `reject_extra_outputs_proposal()`

3. The extra outputs are included in subsequent commitment transactions, providing on-chain backing for off-chain deposits

## Design Principles

1. **Generic mechanism** - The extra outputs API is protocol-agnostic and can be used by any extension

2. **Minimal invasiveness** - Changes are additive and don't modify existing Lightning protocol behavior

3. **External validation** - Protocol-specific validation happens outside rust-lightning via events

4. **Proper capacity accounting** - Extra outputs are correctly subtracted from available channel balances

## Files Modified

| File | Lines Added |
|------|-------------|
| `lightning/src/ln/channelmanager.rs` | +262 |
| `lightning/src/ln/channel.rs` | +226 |
| `lightning/src/ln/chan_utils.rs` | +215 |
| `lightning/src/sign/tx_builder.rs` | +168 |
| `lightning/src/events/mod.rs` | +58 |
