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

interface ITrap {
    function collect() external returns (bytes memory);
    function shouldRespond(bytes[] calldata data) external view returns (bool, bytes memory);
}

contract BalanceFluctuationTrap is ITrap {
    /// @notice Wallet address to monitor
    address public constant target = 0xABcDEF1234567890abCDef1234567890AbcDeF12; 
    /// @notice Sensitivity threshold in thousandths of a percent (0.3% = 30 thousandths)
    uint256 public constant thresholdMilliPercent = 30; // 0.3%

    /**
     * @dev Collects the current ETH balance of the target wallet
     */
    function collect() external view override returns (bytes memory) {
        return abi.encode(target.balance);
    }

    /**
     * @dev Checks whether the balance change exceeds the 0.3% threshold
     */
    function shouldRespond(bytes[] calldata data) external pure override returns (bool, bytes memory) {
        if (data.length < 2) return (false, "Insufficient data");

        uint256 current = abi.decode(data[0], (uint256));
        uint256 previous = abi.decode(data[1], (uint256));

        // Prevent division by zero
        if (previous == 0) {
            if (current > 0) return (true, abi.encode("Balance changed from 0"));
            else return (false, "");
        }

        uint256 diff = current > previous ? current - previous : previous - current;

        // Using 100_000 for precision down to 0.001%
        uint256 milliPercent = (diff * 100_000) / previous;

        if (milliPercent >= thresholdMilliPercent) {
            return (true, abi.encode("Balance fluctuation >= 0.3% detected"));
        }

        return (false, "");
    }
}
```

---

### LogAlertReceiver.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    event Alert(string message);

    function logAnomaly(string calldata message) external {
        emit Alert(message);
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
