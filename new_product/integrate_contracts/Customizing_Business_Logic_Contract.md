<h1 align="center">Guidelines for Developing</h1>

Chains must only be deployed with one CCM contract to implement cross-chain features. For normal running, all the business logic contracts have to interconnect with the CCM contract via the interfaces offered by the CCM contract. See the following for a detailed description or reference to the complete code of the CCM contract.
> [!Note|style:flat|label:Notice]
> To implement cross-chain features, you need to ensure that the cross-chain methods in your business logic contracts have authorized CCM of Poly.

### Step1. Mapping asset

In addition to verifying the existence of transactions through the CCM contract, the business logic contract needs to ensure the accuracy of the asset relationship in the transaction. 
Therefore, the business contract should maintain both the **asset mapping** and **business logic contract mapping**. 
Asset hash is mapped from the source chain to the target chain, and target chain ID is mapped to the business logic contract address on the target chain.

#### Example:

The asset mapping relationship stored in the business logic contract will help complete transaction data. Binding actions also prevent the wrong input from users, leading to the transfer of assets to the incorrect asset contract address.

```solidity
pragma solidity ^0.5.0;

import "./../../libs/ownership/Ownable.sol";

contract LockProxy is Ownable {
    address public managerProxyContract;
    mapping(uint64 => bytes) public proxyHashMap;
    mapping(address => mapping(uint64 => bytes)) public assetHashMap;
    
    // toChainId: the target chain id
    // targetProxyHash: the address of business logic contract on target chain
    function bindProxyHash(uint64 toChainId, bytes memory targetProxyHash) onlyOwner public returns (bool) {
        proxyHashMap[toChainId] = targetProxyHash;
        emit BindProxyEvent(toChainId, targetProxyHash);
        return true;
    }
    
    // fromAssetHash: asset hash on source chain 
    // toAssetHash: asset hash on target chain
    function bindAssetHash(address fromAssetHash, uint64 toChainId, bytes memory toAssetHash) onlyOwner public returns (bool) {
        assetHashMap[fromAssetHash][toChainId] = toAssetHash;
        emit BindAssetEvent(fromAssetHash, toChainId, toAssetHash, getBalanceFor(fromAssetHash));
        return true;
    }
}
```

### Step2. Requesting transaction



This interface creates cross-chain transactions invoked by business logic contracts when a cross-chain function is carried out in the logic contract.

````solidity
/*  
 *  @param toChainId              The target chain id
 *  @param toAddress              The address in bytes format to receive the same amount of tokens in the target chain
 *  @param toContract             Target smart contract address in bytes in the target blockchain
 *  @param txData                 Transaction data for target chain, include toAssetHash, toAddress, amount
 *  @return                       true or false 
*/
function crossChain(uint64 toChainId, bytes calldata toContract, bytes calldata method, bytes calldata txData) whenNotPaused external returns (bool)
````

- This method constructs the `rawParam`, which contains **transaction hash**, `msg.sender`, **target chain ID**, **business logic contract** on target chain, the **target method** to be invoked, and the **serialized transaction data** has already been constructed in the business logic contract. 
- Then put the hash of `rawParam` into storage to prove the cross-chain transaction.

#### Example:

```solidity
/*  
 *  @param fromAssetHash     The asset address in the current chain
 *  @param toChainId         The target chain id
 *  @param toAddress         The address in bytes format to receive the same amount of tokens in the target chain 
 *  @param amount            The number of tokens to be crossed from Ethereum to the chain with chainId
*/
function lock(address fromAssetHash, uint64 toChainId, bytes memory toAddress, uint256 amount) public payable returns (bool) {
    require(amount != 0, "amount cannot be zero!");
    require(_transferToContract(fromAssetHash, amount), "transfer asset from fromAddress to lock_proxy contract failed!");
        
    bytes memory toAssetHash = assetHashMap[fromAssetHash][toChainId];
    require(toAssetHash.length != 0, "empty illegal toAssetHash");

    TxArgs memory txArgs = TxArgs({
        toAssetHash: toAssetHash,
        toAddress: toAddress,
        amount: amount
    });
    bytes memory txData = _serializeTxArgs(txArgs);
        
    IEthCrossChainManagerProxy eccmp = IEthCrossChainManagerProxy(managerProxyContract);
    address eccmAddr = eccmp.getEthCrossChainManager();
    IEthCrossChainManager eccm = IEthCrossChainManager(eccmAddr);
        
    bytes memory toProxyHash = proxyHashMap[toChainId];
    require(toProxyHash.length != 0, "empty illegal toProxyHash");
    require(eccm.crossChain(toChainId, toProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

    emit LockEvent(fromAssetHash, _msgSender(), toChainId, toAssetHash, toAddress, amount);
        
    return true;
    
}
```

- This function is invoked by **users**. Users can request a cross-chain transaction through the dApps, which works with the source chain. The business logic contract then gets the transaction information which involves **asset contract address** on the source chain, **target chain ID**, **target address,** and **amount of token** to be transferred. By calling this method, the business logic contract will **lock** a certain amount to the asset contract;
- Then the transaction data is packed, which invokes the CCM contract. 
- The CCM contract transfers the parameters of transaction data to the target chain based on block generation on the source chain;
- The serialized **transaction data**, chain ID**, **business logic contract address** of target chain, and the method needing to be called on the target chain is sent through `crossChain()` in the CCM contract.

### Step3. Verifying and executing

This method is invoked by the relayer, but in some cases, users could also invoke this method by themselves if they get the valid block information from Poly.

````solidity
/*  
 *  @param proof                  Poly chain transaction Merkle proof
 *  @param rawHeader              The header containing crossStateRoot to verify the above tx Merkle proof
 *  @param headerProof            The header Merkle proof used to verify rawHeader
 *  @param curRawHeader           Any header in current epoch consensus of Poly chain
 *  @param headerSig              The converted signature variable for solidity derived from Poly chain consensus nodes' signature 
 *                                used to verify the validity of curRawHeader
 *  @return                       true or false
*/
function verifyHeaderAndExecuteTx(bytes memory proof, bytes memory rawHeader, bytes memory headerProof, bytes memory curRawHeader, bytes memory headerSig) whenNotPaused public returns (bool)
````

- This method fetches and processes **cross-chain transactions**, finds the **Merkle root** of a transaction based on the block height (in the block header), and verifies the **transaction's legitimacy** using the transaction parameters.
- After verifying the Poly chain block header and proof, it will invoke the business logic contract deployed on the target chain. Invoking will be processed through the internal method `_executeCrossChainTx()`: 
  - This method is meant to invoke the target contract and trigger the execution of cross-chain tx on the target chain. 
  - First, you need to ensure that the target contract is waiting to be invoked as opposed to a standard account address. 
  - Next construct a method for calling the target business logic contract: 
      1. You need to `encodePacked` the `_method` and the format of input data `"(bytes,bytes,uint64)"`;
      2. Then it would `keccak256` the encoded string, using `bytes4` to take the first four bytes of the call data for a function call specifies the function to be called. 
      3. Parameter `_method` is from the `toMerkleValue`, which is parsed from `proof`. And the input parameters format is restricted as (bytes `_args`, bytes `_fromContractAddr`, uint64 `_fromChainId`). These two parts are `encodePacked` as a method call.  
  - After calling the method, you need to check the return value. Only if the return value is true will the whole cross-chain transaction be executed successfully. 

#### Example:

```solidity
/*  
 *  @param argsBs            The argument bytes received by the lock proxy contract on source chain, 
 *                           need to be deserialized based on the way of serialization in the 
 *                           lock proxy contract on source chain.
 *  @param fromContractAddr  The source chain contract address
 *  @param fromChainId       The source chain id
*/
function unlock(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract public returns (bool) {
    TxArgs memory args = _deserializeTxArgs(argsBs);
    require(fromContractAddr.length != 0, "from proxy contract address cannot be empty");
    require(Utils.equalStorage(proxyHashMap[fromChainId], fromContractAddr), "From Proxy contract address error!");
        
    require(args.toAssetHash.length != 0, "toAssetHash cannot be empty");
    address toAssetHash = Utils.bytesToAddress(args.toAssetHash);

    require(args.toAddress.length != 0, "toAddress cannot be empty");
    address toAddress = Utils.bytesToAddress(args.toAddress);

    require(_transferFromContract(toAssetHash, toAddress, args.amount), "transfer asset from lock_proxy contract to toAddress failed!");
        
    emit UnlockEvent(toAssetHash, toAddress, args.amount);
    return true;
}
```

- This function is invoked by the **CCM contract**. It deserializes the transaction data and invokes the asset contract to release the tokens to the target address.
- `verifyHeaderAndExecuteTx()` in CCM contracts determine the **legitimacy** of cross-chain transaction information and resolves the parameters of transaction data from the Poly chain transaction Merkle proof and `crossStateRoot` contained in the block header. 
- After verification through Poly, the packed transaction data can be executed on the target chain.
- Then call the function `unlock()` to deserialize the transaction data, transfer a certain amount of token to the target address on the target chain, and complete the cross-chain contract invocation.

