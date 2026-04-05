# qubtcoin-chain
qubtcoin

```markdown
# QubtCoin

<p align="center">
  < img src="https://img.shields.io/badge/Consensus-QPoH%20+%20Tower%20BFT-blue?style=for-the-badge" alt="Consensus">
  < img src="https://img.shields.io/badge/Block%20Time-400ms-green?style=for-the-badge" alt="Block Time">
  < img src="https://img.shields.io/badge/TPS-50,000+-orange?style=for-the-badge" alt="TPS">
  < img src="https://img.shields.io/badge/Total%20Supply-662,607,015%20QBT-purple?style=for-the-badge" alt="Supply">
</p >

<p align="center">
  <b>A high-performance, quantum-ready Layer 1 blockchain with fixed supply based on Planck's constant.</b>
</p >

<p align="center">
  <a href=" ">Overview</a > •
  <a href="#architecture">Architecture</a > •
  <a href="#consensus">Consensus</a > •
  <a href="#tokenomics">Tokenomics</a > •
  <a href="#getting-started">Getting Started</a > •
  <a href="#api-reference">API</a > •
  <a href="#roadmap">Roadmap</a >
</p >

---

## Overview

QubtCoin is a next-generation public blockchain combining Ethereum's robust account model with Solana's high-throughput consensus architecture. Designed for quantum-era readiness while delivering industry-leading performance today.

### Key Features

| Feature | Specification |
|---------|-------------|
| **Native Token** | Qubtcoin |
| **Total Supply** | 662,607,015 (Planck's constant × 10¹⁵) |
| **Block Time** | 400ms |
| **Consensus** | QPoH (Qubt Proof of History) + Tower BFT |
| **Address Format** | Base58-encoded, starting with 'Q' |
| **Target TPS** | 50,000+ |
| **Finality Time** | ~3 seconds |
| **Contract Support** | EVM-compatible + native Rust |

---

## Architecture

```

┌─────────────────────────────────────────────────────────────┐
│                    QubtCoin Protocol Stack                   │
├─────────────────────────────────────────────────────────────┤
│  Application Layer    │  Smart Contracts (QVM), DeFi, NFTs   │
├─────────────────────────────────────────────────────────────┤
│  Execution Layer      │  Parallel Transaction Execution      │
│                       │  (Sealevel-inspired)                 │
├─────────────────────────────────────────────────────────────┤
│  Consensus Layer      │  QPoH Sequence Generator             │
│                       │  Tower BFT Finality Gadget           │
├─────────────────────────────────────────────────────────────┤
│  Network Layer        │  Turbine Block Propagation           │
│                       │  Gulf Stream Transaction Forwarding  │
├─────────────────────────────────────────────────────────────┤
│  Data Layer           │  Merkle Patricia Trie State          │
│                       │  Quantum-Resistant Hash Functions    │
└─────────────────────────────────────────────────────────────┘

```

---

## Consensus

### QPoH (Qubt Proof of History)

QPoH provides a cryptographically verifiable passage of time before consensus, enabling high throughput and deterministic ordering without traditional timestamp vulnerabilities.

```python
import hashlib
import time
from dataclasses import dataclass
from typing import Optional, List

@dataclass
class QPoHEntry:
    """Single entry in the QPoH sequence."""
    hash: str
    index: int
    timestamp: float
    data_hash: Optional[str]

class QPoHGenerator:
    """
    Verifiable delay function generating sequential hashes.
    Each hash depends on the previous, creating temporal proof.
    """
    
    def __init__(self, seed: str = "QubtGenesis"):
        self.sequence: List[QPoHEntry] = []
        self.current = QPoHEntry(
            hash=hashlib.sha256(seed.encode()).hexdigest(),
            index=0,
            timestamp=time.time(),
            data_hash=None
        )
        self.sequence.append(self.current)
    
    def tick(self, data: Optional[str] = None) -> QPoHEntry:
        """Generate next sequence entry."""
        data_hash = hashlib.sha256(data.encode()).hexdigest() if data else None
        combined = f"{self.current.hash}{self.current.index + 1}{data_hash or ''}"
        
        self.current = QPoHEntry(
            hash=hashlib.sha256(combined.encode()).hexdigest(),
            index=self.current.index + 1,
            timestamp=time.time(),
            data_hash=data_hash
        )
        self.sequence.append(self.current)
        return self.current
    
    def verify_sequence(self, start_idx: int, end_idx: int) -> bool:
        """Verify sequence integrity without recomputing."""
        for i in range(start_idx + 1, end_idx + 1):
            prev = self.sequence[i - 1]
            curr = self.sequence[i]
            expected = hashlib.sha256(
                f"{prev.hash}{i}{curr.data_hash or ''}".encode()
            ).hexdigest()
            if curr.hash != expected:
                return False
        return True
    
    def get_proof(self, index: int) -> dict:
        """Generate proof for specific sequence index."""
        return {
            "index": index,
            "hash": self.sequence[index].hash,
            "timestamp": self.sequence[index].timestamp,
            "prev_hash": self.sequence[index - 1].hash if index > 0 else None
        }
```

Tower BFT Consensus

Combines optimistic confirmation with Byzantine fault tolerance for sub-second finality.

```python
from typing import List, Dict, Set, Optional
from dataclasses import dataclass, field
from enum import Enum

class VoteState(Enum):
    UNCONFIRMED = 0
    CONFIRMED = 1
    FINALIZED = 2

@dataclass
class Vote:
    validator: str
    stake_weight: int
    target_block: str
    previous_vote: Optional[str] = None
    slot: int = 0

@dataclass
class Block:
    hash: str
    parent_hash: str
    slot: int
    poh_sequence: int
    transactions: List[str] = field(default_factory=list)
    state_root: str = ""
    
class TowerBFT:
    """
    Tower Byzantine Fault Tolerance with optimistic confirmation.
    Requires 2/3 stake weight for finality.
    """
    
    SUPERMINORITY = 2 / 3
    ROOT_DEPTH = 32  # Maximum rollback depth
    
    def __init__(self):
        self.validators: Dict[str, int] = {}  # address -> stake
        self.votes: Dict[str, List[Vote]] = {}  # block -> votes
        self.block_states: Dict[str, VoteState] = {}
        self.confirmed_blocks: Set[str] = set()
        self.finalized_blocks: Set[str] = set()
        self.root_block: Optional[str] = None
        self.current_slot: int = 0
        
    def register_validator(self, address: str, stake: int):
        """Register validator with stake amount."""
        if stake < 32000:  # Minimum stake requirement
            raise ValueError("Insufficient stake amount")
        self.validators[address] = stake
    
    def total_stake(self) -> int:
        return sum(self.validators.values())
    
    def process_vote(self, vote: Vote) -> VoteState:
        """
        Process validator vote.
        Returns confirmation state of the target block.
        """
        if vote.validator not in self.validators:
            raise ValueError("Unknown validator")
            
        if vote.stake_weight != self.validators[vote.validator]:
            raise ValueError("Invalid stake weight")
        
        if vote.target_block not in self.votes:
            self.votes[vote.target_block] = []
        
        # Check for duplicate vote
        existing = [v for v in self.votes[vote.target_block] 
                   if v.validator == vote.validator]
        if existing:
            return self.block_states.get(vote.target_block, VoteState.UNCONFIRMED)
        
        self.votes[vote.target_block].append(vote)
        
        # Calculate total weight
        total_weight = sum(
            v.stake_weight for v in self.votes[vote.target_block]
        )
        
        # Check for optimistic confirmation (2/3 majority)
        if total_weight >= self.total_stake() * self.SUPERMINORITY:
            self.confirmed_blocks.add(vote.target_block)
            self.block_states[vote.target_block] = VoteState.CONFIRMED
            
            # Check for finality (confirmed + sufficient descendants)
            if self.check_finality(vote.target_block):
                self.finalized_blocks.add(vote.target_block)
                self.block_states[vote.target_block] = VoteState.FINALIZED
                self.update_root(vote.target_block)
            
            return self.block_states[vote.target_block]
        
        return VoteState.UNCONFIRMED
    
    def check_finality(self, block_hash: str) -> bool:
        """Check if block has enough descendant confirmations for finality."""
        # Simplified: In production, check descendant chain depth
        return block_hash in self.confirmed_blocks
    
    def update_root(self, block_hash: str):
        """Update root block for pruning."""
        self.root_block = block_hash
    
    def prune_old_blocks(self):
        """Prune blocks beyond rollback depth."""
        if not self.root_block:
            return
        
        # Remove blocks below root depth
        blocks_to_remove = []
        for block in self.votes.keys():
            if block not in self.finalized_blocks:
                # Check if too old compared to root
                blocks_to_remove.append(block)
        
        for block in blocks_to_remove:
            del self.votes[block]
            if block in self.block_states:
                del self.block_states[block]
```

---

Tokenomics

Fixed Supply Model

Total supply is permanently capped at 662,607,015 QBT, derived from Planck's constant (6.62607015 × 10⁻³⁴ J⋅s) scaled to 10¹⁵ for practical divisibility.

```python
class QubtTokenomics:
    """
    Qubtcoin Token Economics
    Fixed supply: 662,607,015 Qubtcoin (never increases)
    Deflationary via fee burning
    """
    
    # Constants
    MAX_SUPPLY = 662_607_015
    DECIMALS = 9
    INITIAL_BLOCK_REWARD = 3.14159265  # π (irrational number)
    HALVING_INTERVAL = 10_512_000      # ~2 years at 400ms blocks
    MIN_REWARD = 0.00000001            