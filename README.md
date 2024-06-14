
## Building a Paymaster to Reward Staking in zkSync
![Multipurpose blue Zoom Background Template - Made with PosterMyWall](https://github.com/johnwinifred/Building-a-Paymaster-to-Reward-Staking-in-zkSync/assets/89465179/8a4f4b7f-5a2a-4007-9893-ca74b2dcb208)


> Paymasters are virtual accounts that can pay for transactions (gas fees) on behalf of other accounts. zkSync Era introduced the Paymaster feature under the Account Abstraction EIP Initiative, as a way to increase flexibility for developers and improve user experience with lower transaction fees. 

Read more about zkSync here.
[zkSync Official docs](https://docs.zksync.io/build/developer-reference/account-abstraction.html#sending-transactions-from-an-account)




This tutorial will guide you through implementing paymasters on zkSync to **reward users for staking** tokens by **covering their transaction fees**. This approach reduces the burden of gas fees on users and encourages more active participation in staking.

#### What You will do: 
- Create a paymaster that checks if a user has staked tokens before covering their transaction fee.
- Create a staking contract where users can stake their tokens and earn rewards.
- Deploy the staking and paymaster contracts, and stake tokens using the staking contract.
- Send a transaction that typically requires gas fees and have the paymaster cover the fees for stakers.

## Pre Requisites:
- Make sure your machine satisfies the [system requirements](https://github.com/matter-labs/era-compiler-solidity/tree/main#system-requirements).
- A Node.js installation running Node.js version 16.
- Some familiarity with deploying smart contracts on zkSync. If not, please refer to this [quickstart tutorial](https://docs.zksync.io/build/quick-start/hello-world.html).
- Some background knowledge on the concepts covered by the tutorial would be helpful too. Have a look at the following docs:
  - [Account abstraction protocol](https://docs.zksync.io/build/developer-reference/account-abstraction.html).
  - [Introduction to system contracts](https://docs.zksync.io/build/developer-reference/system-contracts.html).
  - [Smart contract deployment on zkSync Era](https://docs.zksync.io/build/developer-reference/system-contracts.html).
  - [Gas estimation for transactions guide](https://docs.zksync.io/build/developer-reference/fee-model.html#gas-estimation-for-transactions).
- You should also know [how to get your private key from your MetaMask wallet](https://support.metamask.io/hc/en-us/articles/360015289632-How-to-export-an-account-s-private-key)

> Once you have gotten all these requirements, proceed to building 
Have fun builder 👨‍💻 !!!


---


### 1. Project Setup

Open your terminal or command prompt.

#### 1.1 Create the Project Using zkSync CLI:

```bash
npx zksync-cli create staking-reward-paymaster --template hardhat_solidity
```
*This creates a new zkSync Era project called staking-reward-paymaster with a basic Greeter contract.*

#### 1.2 Navigate into the Project Directory:

```bash
cd staking-reward-paymaster
```
*Change the current directory to the project directory.*

#### 1.3 Remove Example Contracts and Deploy Files:

```bash
rm -rf ./contracts/*
rm -rf ./deploy/*
```
if you are running in PowerShell use this instead
```bash
Remove-Item -Recurse -Force ./contracts/*
Remove-Item -Recurse -Force ./deploy/*
```
*Remove any example contracts and deploy scripts to start with a clean slate.*

#### 1.4 Add Required Libraries:

```bash
yarn add -D @matterlabs/zksync-contracts @openzeppelin/contracts@4.9.5
yarn add --dev hardhat @nomiclabs/hardhat-waffle @nomiclabs/hardhat-ethers ethers @matterlabs/hardhat-zksync-deploy

```
*Install the zkSync contracts and OpenZeppelin contracts libraries.*

#### 1.5 Configure Hardhat for zkSync in `hardhat.config.ts`:
> We used `hardhat.config` to manage, compile and deploy this project. There are alternative options like using [zkSync CLI](https://docs.zksync.io/build/tooling/zksync-cli/getting-started.html)

Read more about [zkSync Hardhat Configuration](https://docs.zksync.io/build/tooling/hardhat/getting-started.html)

Create or edit `hardhat.config.ts` with the following configuration:

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@matterlabs/hardhat-zksync-deploy";
import "@matterlabs/hardhat-zksync-solc";
import "@matterlabs/hardhat-zksync-verify";

// dynamically alters endpoints for local tests
const zkSyncTestnet =
  process.env.NODE_ENV == "test"
    ? {
        url: "http://localhost:3050",
        ethNetwork: "http://localhost:8545",
        zksync: true,
      }
    : {
        url: "https://sepolia.era.zksync.dev",
        ethNetwork: "sepolia",
        zksync: true,
        verifyURL: "https://explorer.sepolia.era.zksync.dev/contract_verification", // Verification endpoint
      };

const config: HardhatUserConfig = {
  zksolc: {
    version: "latest", // Uses latest available in https://github.com/matter-labs/zksolc-bin/
    settings: {},
  },
  defaultNetwork: "zkSyncTestnet",
  networks: {
    hardhat: {
      zksync: false,
    },
    zkSyncTestnet,
  },
  solidity: {
    version: "0.8.17",
  },
};

export default config;
```
*This configuration sets up Hardhat to work with zkSync, specifying the necessary plugins and network details.*

### 2. Create the Staking Contract

Create a new file `Staking.sol` in the `contracts` directory and add the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract Staking is Ownable {
    IERC20 public stakingToken;
    mapping(address => uint256) public balances;
    mapping(address => uint256) public rewards;

    event Staked(address indexed user, uint256 amount);
    event Unstaked(address indexed user, uint256 amount);
    event RewardSet(address indexed user, uint256 reward);

    constructor(IERC20 _stakingToken) {
        stakingToken = _stakingToken;
    }

    function stake(uint256 amount) external {
        require(amount > 0, "Amount must be greater than 0");
        stakingToken.transferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
        emit Staked(msg.sender, amount);
    }

    function unstake(uint256 amount) external {
        require(amount > 0, "Amount must be greater than 0");
        require(balances[msg.sender] >= amount, "Insufficient balance to unstake");
        balances[msg.sender] -= amount;
        stakingToken.transfer(msg.sender, amount);
        emit Unstaked(msg.sender, amount);
    }

    function getReward(address account) external view returns (uint256) {
        return rewards[account];
    }

    function setReward(address account, uint256 reward) external onlyOwner {
        rewards[account] = reward;
        emit RewardSet(account, reward);
    }
}

```
*This contract allows users to stake tokens, unstake them, and get rewards. The contract owner can set rewards for users.*

### 3. Create the Paymaster Contract

Create a new file `StakingRewardPaymaster.sol` in the `contracts` directory and add the following code:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IPaymaster.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract StakingRewardPaymaster is IPaymaster, Ownable {
    IERC20 public allowedToken;
    address public stakingContract;

    constructor(address _allowedToken, address _stakingContract, address _zkSyncAddress) 
        IPaymaster(_zkSyncAddress) 
    {
        allowedToken = IERC20(_allowedToken);
        stakingContract = _stakingContract;
    }

    function validateAndPayForPaymasterTransaction(
        bytes32,
        bytes32,
        IPaymaster.Transaction calldata _transaction
    ) external payable onlyOwner returns (bytes4 magic, bytes memory context) {
        address user = _transaction.from;

        (bool success, bytes memory result) = stakingContract.staticcall(
            abi.encodeWithSignature("getReward(address)", user)
        );
        require(success, "Failed to query staking contract");

        uint256 reward = abi.decode(result, (uint256));
        require(reward > 0, "User has no staking rewards");

        return (IPaymaster.PAYMASTER_SUCCESS(), "");
    }

    function postTransaction(
        bytes calldata,
        IPaymaster.Transaction calldata _transaction,
        bytes32,
        bytes32,
        IPaymaster.ExecutionResult calldata,
        uint256
    ) external payable onlyOwner {
        // Implement post-transaction processing logic here if needed
    }
}

```
*This contract interacts with the staking contract to determine if a user has rewards and covers their transaction fees if they do.*

### 4. Deploy the Contracts

Create a new deploy script `deploy.ts` in the `deploy` directory:

```typescript
import { ethers, network } from "hardhat";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";
import { Wallet } from "ethers";
import { Provider } from "@matterlabs/hardhat-zksync-ethers";

async function main() {
    const zkSyncProvider = new Provider("https://sepolia.era.zksync.dev");
    const wallet = new Wallet("<YOUR_PRIVATE_KEY>", zkSyncProvider);
    const deployer = new Deployer(network, wallet);

    const stakingArtifact = await deployer.loadArtifact("Staking");
    const stakingContract = await deployer.deploy(stakingArtifact, [
        "<STAKING_TOKEN_ADDRESS>", // Replace with the actual staking token address
        100 // Reward

```
*This script deploys both the staking and paymaster contracts. Replace `<STAKING_TOKEN_ADDRESS>` with the actual staking token address.*

### 5. Stake Tokens and Use the Paymaster

#### 5.1 Stake Tokens

To stake tokens, interact with the `Staking` contract using a script or through a frontend. Here is an example of a script to stake tokens:

```typescript
import { ethers } from "hardhat";

async function main() {
  const [staker] = await ethers.getSigners();
  const stakingContractAddress = "<STAKING_CONTRACT_ADDRESS>"; // Replace with the deployed staking contract address
  const stakingTokenAddress = "<STAKING_TOKEN_ADDRESS>"; // Replace with the staking token address

  const staking = await ethers.getContractAt("Staking", stakingContractAddress);
  const token = await ethers.getContractAt("IERC20", stakingTokenAddress);

  const stakeAmount = ethers.utils.parseUnits("10", 18); // Example amount to stake

  // Approve the staking contract to spend tokens
  await token.approve(stakingContractAddress, stakeAmount);
  
  // Stake tokens
  await staking.stake(stakeAmount);
  
  console.log("Staked", ethers.utils.formatUnits(stakeAmount, 18), "tokens");
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
*Ensure the staking token address and staking contract address are correct.*

#### 5.2 Send a Transaction with Paymaster Covering the Fees

To send a transaction with the paymaster covering the fees, you would need to integrate the paymaster logic into your transaction flow. This typically involves specifying the paymaster address in the transaction metadata. Below is an example of how you can structure the a transaction:

```typescript
import { ethers } from "hardhat";
import { Wallet, Provider } from "ethers";
async function main() {
  const provider = new Provider("https://testnet.era.zksync.dev"); // zkSync testnet provider
  const [sender] = await ethers.getSigners();
const paymasterAddress = "<PAYMASTER_CONTRACT_ADDRESS>"; // Replace with the deployed paymaster contract address
// Example transaction to be sent
  const tx = {
    to: "<RECIPIENT_ADDRESS>", // Replace with the recipient address
    value: ethers.utils.parseEther("0.01"), // Amount to send
    gasPrice: await provider.getGasPrice(),
    gasLimit: ethers.BigNumber.from("21000"), // Estimate gas limit for a simple transfer
    customData: {
      paymasterParams: {
        paymaster: paymasterAddress,
      },
    },
  };
// Sign and send the transaction using the paymaster
  const signedTx = await sender.sendTransaction(tx);
await signedTx.wait();
console.log("Transaction sent with paymaster covering fees:", signedTx.hash);
}
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
### 6. Compile Your Contracts:


```bash
npx hardhat compile
```
### 6.1 Deploy Contracts:

Write your deployment scripts in the scripts or deploy directory.
Deploy using Hardhat:

```
npx hardhat run deploy/deploy.ts --network zkSyncTestnet
```
### 6.2 Run Tests:
```
npx hardhat test

```

> Note: If contracts fail to deploy, double-check the configuration in `hardhat.config.ts` and ensure the zkSync network is correctly set up.
Verify that all token and contract addresses are correct and match the deployed addresses.

Read the tutorial [Building Custom Paymasters](https://docs.zksync.io/build/tutorials/smart-contract-development/paymasters/custom-paymaster-tutorial.html) to learn how to build paymasters.


## Resources
- [zkSync Official Docs](https://docs.zksync.io/build/developer-reference/account-abstraction.html)
- [Building a Custom Paymaster](https://docs.zksync.io/build/tutorials/smart-contract-development/paymasters/custom-paymaster-tutorial.html)
- [zkSync Hardhat Pluggins](https://docs.zksync.io/build/tooling/hardhat/getting-started.html)
- [Account Abstraction in zkSync](https://docs.zksync.io/build/developer-reference/account-abstraction.html) 





