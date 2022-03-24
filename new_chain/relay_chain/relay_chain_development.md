<h1 align="center">Develop for Poly Chain</h1>

## 1. Requirements

To develop for Poly chain, there are three **prerequisites**.

### 1.1 Light client verification

The block header must contain the following information：
- Hash of the previous block header
- Merkle state root hash
- Necessary information to prove the legitimacy of the block header varies for different consensus algorithms.

> [!Note|style:flat|label:Notice]
> If your chain **doesn't** support techniques like Simple Payment Verification (SPV) protocol in Bitcoin or Light Ethereum Subprotocol (LES) in Ethereum, get in touch with the Poly team via <a class="fab fa-discord" href= "https://discord.com/invite/y6MuEnq"></a> for more support.

### 1.2 Block header structure and verification methods

The following information is necessary：
- Block header structure
- Serialization and Deserialization methods
- Block header verification methods

### 1.3 Verifiable state root

The following information is necessary：
- Merkle tree structure
- State root verification methods

## 2. Development Specifications

With the aforementioned prerequisites, you can start developing methods for Poly chain following the guideline below based on your own requirements. 

### 2.1 Synchronize block headers

#### Block Header Synchronization Methods

| Method                | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| **SyncGenesisHeader** | Stores and handles the initial block header so that the subsequent block headers contain cross-chain events that can be verified and synchronized. This method will only be called for **once** when initializing the new chain. Please refer to the [code](https://github.com/polynetwork/poly/blob/master/native/service/header_sync/eth/header_sync.go#L61) for more details. |
| **SyncBlockHeader**   | Consistently synchronizes block cycle changes and cross-chain transaction block headers from the new chain to the Poly chain. Please refer to the [code](https://github.com/polynetwork/poly/blob/master/native/service/header_sync/eth/header_sync.go#L99) for more details. |


#### Block Header Synchronization Entrance Method

| Method                           | Description                                                                                                                                                                                                                                                                                                          |
|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **SyncSideChainGenesisHeader**   | The entrance method for synchronizing the genesis block header of the new chain to Poly chain and synchronizing the genesis header of Poly chain to the CCM contract of the new chain. Please refer to the [code](https://github.com/polynetwork/poly-io-test/blob/master/cmd/tools/run.go#L607) for more details. |

The **key information** for this method (submitted by .config):
- Service provider (endpoint) URL of the new chain.
- Genesis block height (normally the most recent).
- Essential information for verifying genesis headers may already exist in the header information or it may need to be fetched from block headers of other block heights.
- Information required for the new chain block header verification.


### 2.2 Verify cross-chain transactions

#### Cross Chain Management

| Method                  | Description                                                  |
| ----------------------- | :----------------------------------------------------------- |
| **MakeDepositProposal** | Acts as entrance of verifyFromTx, **verifying**, **storing**, and **returning** MakeTxParam for processing cross-chain steps, verifies cross-chain transactions, and brings legitimate transactions to the poly chain.  Please refer to the [code](https://github.com/polynetwork/poly/blob/master/native/service/cross_chain_manager/eth/eth_handler.go#L34) for more details. |

```go
MakeDepositProposal:
Requires:
service *native.NativeService   //Native Service that carries values of information of cross-chain events
Returns:
type verifyFromTx struct {
	TxHash              []byte    
	CrossChainId        []byte    //ChainId of source chain
	FromContractAddress uint64    //Cross Chain Management Contarct address of source chain
	ToChainId           string    //ChainId of target chain
	ToContractAddress   uint64    //Cross Chain Management Contarct address of target chain
	Method              []byte    //Unlock or lock
	Args                []byte
}
```
| Method           | Description                                                  |
| ---------------- | :----------------------------------------------------------- |
| **verifyFromTx** | Prepares block header and deserialized proof for verifyMerkleProof, decodes the extra data from tx, and constructs MakeTxParam. Please refer to the [code](https://github.com/polynetwork/poly/blob/4323af5cfcd2a3277653d5bdc4db015cd9755fee/native/service/cross_chain_manager/eth/utils.go#L41) for more details. |

```go
verifyFromTx:
Requires:
service               *native.NativeService  
proof                 []byte    //the proof to be serialized and verified
extra                 []byte    //the transaction information that will be used for constructing verifyFromTx
fromChainID           uint64,   //ChainId of source chain
height                uint32,   //the block height corresponding to current transaction event
sideChain             *side_chain_manager.SideChain //source chain information that contains ccm contract address
Returns:
txParam               *scom.MakeTxParam 
```
| Method                | Description                                                                                                                                                                                                                                                                                                                                                                            |
|-----------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **verifyMerkleProof** | Verifies the Merkle proof obtained by the relayer generated from the source chain to ensure that all transactions included in this block header have been created and are visible on the Poly chain. Please refer to the [code](https://github.com/polynetwork/poly/blob/4323af5cfcd2a3277653d5bdc4db015cd9755fee/native/service/cross_chain_manager/eth/utils.go#L88) for details. |

```go
verifyMerkleProof:
Requires:
blockData             *types.Header //the blockheader corresponding to current transaction event  
proof                 []byte    //the serialized proof
Returns:
Val                   []byte    //the proof result for checking extra before constructing verifyFromTx
```
