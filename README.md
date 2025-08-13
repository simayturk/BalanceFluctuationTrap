# 🛡️ BalanceFluctuationTrap & LogAlertReceiver

This repository contains two Solidity smart contracts designed for monitoring ETH balance fluctuations on a specific wallet address within the **Ethereum Hoodi** network. These contracts can be integrated into the [Drosera](https://github.com/Drosera-Network) framework for automated balance anomaly detection and alert logging.

---

## 📜 Description

1. **BalanceFluctuationTrap** – A trap contract that monitors the ETH balance of a predefined wallet (`0xABcDEF1234567890abCDef1234567890AbcDeF12`).  
   It triggers whenever the balance changes by **0.3% or more** between two consecutive checks (increase or decrease). The contract implements the `ITrap` interface for compatibility with Drosera monitoring systems.

2. **LogAlertReceiver** – An auxiliary contract that receives anomaly messages and emits an `Alert` event. This allows external monitoring services or scripts to capture and react to trap triggers.

---

## ⚙ Features

- ✅ Native **Ethereum Hoodi** network support  
- ✅ Real-time monitoring of ETH balance for a specific wallet  
- ✅ High-precision detection (0.3% threshold with 0.001% granularity)  
- ✅ Safe handling of zero-balance cases  
- ✅ Compatible with Drosera's `ITrap` interface  
- ✅ Optional anomaly logging via `LogAlertReceiver`  

---

## 📜 Contracts

### BalanceFluctuationTrap.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ITrap} from "drosera-contracts/interfaces/ITrap.sol";

contract BalanceFluctuationTrap is ITrap {
    /// Wallet to monitor
    address public constant TARGET = 0xABcDEF1234567890abCDef1234567890AbcDeF12;

    /// Threshold in 0.001% units (bps/10). 0.3% = 300
    uint256 public constant THRESHOLD_MILLI_PERCENT = 300;

    /// Optional guards to avoid noise
    uint256 public constant MIN_PREV_WEI = 0;      // set e.g. 0.1 ether if needed
    uint256 public constant MIN_ABS_DIFF = 0;      // set e.g. 0.01 ether if needed

    function collect() external view returns (bytes memory) {
        return abi.encode(TARGET, TARGET.balance, block.number);
    }

    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory) {
        if (data.length < 2) return (false, "");

        (address t0, uint256 curr, uint256 bn0) = abi.decode(data[0], (address, uint256, uint256));
        (address t1, uint256 prev, uint256 bn1) = abi.decode(data[1], (address, uint256, uint256));

        if (t0 != t1) return (false, "");                // sanity: same target
        if (prev <= MIN_PREV_WEI) return (false, "");    // optional dust guard
        uint256 diff = curr > prev ? curr - prev : prev - curr;
        if (diff < MIN_ABS_DIFF) return (false, "");

        // 0.001% precision
        uint256 milliPct = (diff * 100_000) / prev;
        if (milliPct < THRESHOLD_MILLI_PERCENT) return (false, "");

        // reason: 0 = DROP, 1 = SPIKE
        uint8 reason = curr < prev ? 0 : 1;
        bytes memory payload = abi.encode(
            reason, t0, prev, curr, diff, milliPct, bn1, bn0
        );
        return (true, payload);
    }
}


```

---

### LogAlertReceiver.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    // reason: 0 = DROP, 1 = SPIKE
    event Alert(
        uint8 indexed reason,
        address indexed target,
        uint256 previous,
        uint256 current,
        uint256 diff,
        uint256 milliPercent,
        uint256 blockPrev,
        uint256 blockCurr
    );

    // Configure this as the response function in drosera.toml: response_function = "react(bytes)"
    function react(bytes calldata payload) external {
        (
            uint8 reason,
            address target,
            uint256 prev,
            uint256 curr,
            uint256 diff,
            uint256 milliPct,
            uint256 bnPrev,
            uint256 bnCurr
        ) = abi.decode(payload, (uint8, address, uint256, uint256, uint256, uint256, uint256, uint256));

        emit Alert(reason, target, prev, curr, diff, milliPct, bnPrev, bnCurr);
    }
}

```

---

## ⚙ Requirements

- [Foundry](https://book.getfoundry.sh/getting-started/installation) (forge & cast)
- Ethereum Hoodi RPC endpoint: `https://ethereum-hoodi-rpc.publicnode.com`
- Wallet with sufficient ETH for deployment gas fees

---

## 🚀 Build

```bash
forge build
```

---

## 📜 Deploy

Deploy `BalanceFluctuationTrap`:

```bash
forge create \
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \
  --broadcast \
  --private-key <YOUR_PRIVATE_KEY> \
  src/BalanceFluctuationTrap.sol:BalanceFluctuationTrap
```

Deploy `LogAlertReceiver`:

```bash
forge create \
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \
  --broadcast \
  --private-key <YOUR_PRIVATE_KEY> \
  src/LogAlertReceiver.sol:LogAlertReceiver
```

---

## 🧩 Usage

- `collect()` – retrieves the current ETH balance of the monitored wallet.  
- `shouldRespond()` – checks if the balance change is **≥ 0.3%** and returns `true` if triggered.  
- `LogAlertReceiver.logAnomaly()` – logs anomaly events that can be monitored externally.

Example:

```solidity
(bool triggered, bytes memory message) = trap.shouldRespond(data);
if (triggered) {
    logReceiver.logAnomaly("Balance fluctuation >= 0.3% detected");
}
```

---

## 📜 License

MIT
