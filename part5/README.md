Until the assignment is posted on Canvas, this description is subject to change.

# Midterm Project (Part 5)

This part of the project will deal with transactions and the ledger state. You need to integrate the transaction structure inside the block content, add network functionality to transaction propagation, and add a transaction mempool to be used by the miner to include transaction content in the block being mined. You also need to maintain a state for the ledger that the blockchain creates and add all the necessary checks related to it.

This is the last part of the project, and you are going to finish the Bitcoin client.

## Code provided
We provided some reference codes for you to start with. It is optional to use the code we provided. You can also start from scratch.
- `mempool.rs`: a simple mempool structure that supports adding and removing transactions.
- `transaction_generator.rs`: a simple transaction generator that runs in a separate thread and periodically generates transactions and adds them to the mempool.
- `address.rs`: a simple H160 struct that represents a 20-byte array. It is used to represent the address in Bitcoin/Ethereum.

Make sure that you add any new files inside the `src/` folder. Code in new files does not automatically become a part of the Rust project. Instead, (assuming you add all the above files in the _root_ of the `src/` folder), you need to add the following lines to `src/main.rs`:
```rust
pub mod address;
pub mod mempool;
pub mod transaction_generator;
```
Suggestion: For better readability and code organization, you can add `address.rs` to the `src/crypto` directory.
---

## Programming

### 1. Transaction format

There are two main approaches to handle transactions: UTXO model and account-based model.
In this project, we recommend using a transaction structure that is compatible with the account-based model in Ethereum. 

- Account-based model transaction: it should contain a recipient address, a value, and an account nonce. It only supports single sender and single receiver. This should be simpler to implement than UTXO model.
Please make sure the `RawTransaction` struct has the following form:
```rust
/// Account-based transaction
#[derive(Serialize, Deserialize, Debug, Default, Clone)]
pub struct RawTransaction {
    pub from_addr: H160,
    pub to_addr: H160,
    pub value: u64,
    pub nonce: u32,
}
```

Note: An address in Bitcoin and Ethereum is a 20-byte array, a.k.a. H160. We have provided a struct H160 in `address.rs`. The conversion from public key to H160 is different in Bitcoin and Ethereum. We suggest to use the following simple conversion:

> first hash the public key, then take the last 20 bytes. (We've already implemented this for you as `H160::from_pubkey`).

Now it's time to add **Signature** to transaction. You need to convert the **Signature** struct of *ring* to bytes since it is not serializable. You also need to do the same for public key. Then append the public key and the signature to transaction by creating a struct **SignedTransaction** that contains the transaction, the public key, and the signature,

A solution might look like this:
```rust
/// A signed transaction
#[derive(Serialize, Deserialize, Debug, Default, Clone)]
pub struct SignedTransaction {
    // to avoid name confusion, we recommend renaming `Transaction` to `RawTransaction`:
    pub raw: RawTransaction,  
    pub pub_key: Vec<u8>,
    pub signature: Vec<u8>,
}

impl SignedTransaction {
    /// Create a new transaction from a raw transaction and a key pair
    pub fn from_raw(raw: RawTransaction, key: &Ed25519KeyPair) -> SignedTransaction {
        let pub_key = key.public_key().as_ref().to_vec();
        let signature = sign(&raw, key).as_ref().to_vec();
        SignedTransaction { raw, pub_key, signature }
    }

    /// Verify the signature of this transaction
    pub fn verify_signature(&self) -> bool {
        let serialized_raw = bincode::serialize(&self.raw).unwrap();
        let public_key = ring::signature::UnparsedPublicKey::new(
            &ring::signature::ED25519, &self.pub_key[..]);
        public_key.verify(&serialized_raw, self.signature.as_ref()).is_ok()
    }
}
```

---
### 2. Transaction Mempool

We need a transaction **Mempool** structure to store all the received valid transactions that have not been included in the blockchain yet. (A reference implementation is provided in `mempool.rs`.)

**Mempool** will be used by the workers and the transaction generator. If a newly generated/received transaction passes the validity checks (see below sections), we need to add it to the mempool.

**Mempool** will also be used by the miner to include transactions in the blocks being mined. The miner will add transactions in the mempool to the block till it reaches the block size limit. You are free to choose the size limit on the blocks. On processing a new block (which is not an orphan or stale), remove corresponding transactions from the mempool. (You probably want to modify the function `miner_loop` in `miner.rs` to reflect this.)

Similar to **Blockchain**, you need the thread-safe wrapper: `Arc<Mutex<Mempool>>`. The mempool should be initialized in `main.rs` and passed to the workers, the miner, and the transaction generator.

---
### 3. Transaction network messages

Add the following new messages to `src/network/message.rs` and add the corresponding code to handle these messages:
1. `NewTransactionHashes(Vec<H256>)`, similar to NewBlockHashes. Such messages are initiated by the transaction generator and relayed by the workers. When a worker receives this message, it should request from the sender the transactions not yet in the mempool.
2. `GetTransactions(Vec<H256>)`, similar to GetBlocks. When a worker receives this message, it should send the corresponding transactions to the sender.
3. `Transactions(Vec<Transaction>)`, similar to Blocks. When a worker receives this message, it should validate the transactions (see below sections), add the valid ones to the mempool, and broadcast them. Here `Transaction` type means the 
`SignedTransaction` type. You can use 

```rust
use crate::transaction::SignedTransaction as Transaction;
```
and use `Transaction` as an alias to `SignedTransaction`.

---
### 4. Transaction validity checks
When receiving and processing a new transaction in *src/network/worker.rs*, please add the following checks.

#### Transaction signature checks:

- First, check if the transaction is signed correctly by the public key(s).
- In the account-based model, check if the public key matches the owner's address of the withdrawing account.

#### Double spend checks:

- In the account-based model, check if the balance is enough and the suggested account nonce is equal to one plus the account nonce. This check also needs **State** (see below).

#### Add those checks when processing _blocks_:

- When receiving and processing a _block_, also check transactions inside it.

---
### 5. State

Ledger state, or **State**, is a collection of all the required information to check transactions.

- In the account-based model, **State** should contain all the accounts' information. It may contain *(account address, account nonce, balance)*.

To access data conveniently, we recommend using HashMap to store State. In the account-based model, we recommend `HashMap<account address, (account nonce, balance)>`.

#### State update:
When executing a block, i.e., executing transactions in that block, we need to update the state.
- In the account-based model, change account's nonce and balance. Create new accounts if you need to.

#### Initial state (ICO):
You can do initial coin offering (ICO) by inserting entries into **State** struct.
- In the account-based model, create accounts whose addresses are under your control, and give them non-zero balances.
You can do the ICO as follows:
In the `address.rs` file, add the following function:
```rust
// for Initial coin offering:
/// Get a deterministic keypair from a nonce:
pub fn get_deterministic_keypair(nonce: u8) -> Ed25519KeyPair {
    let mut seed = [0u8; 32];
    seed[0] = nonce;
    let keypair = Ed25519KeyPair::from_seed_unchecked(&seed).unwrap();
    keypair
}
```
Then, in the `blockchain.rs` file, you can have:
```rust
#[derive(Clone)]
pub struct State {
    map: HashMap<H160, (u32, u64)>
}

impl State {
    /// Initial coin offering; generate an initial state.
    fn ico() -> Self {
        let mut state = HashMap::new();
        // give the i-th account 1000 * (10 - i) coins, i = 0, 1, 2, ..., 9
        for i in 0..10 {
            let pair = get_deterministic_keypair(i);
            let address = H160::from_pubkey(pair.public_key().as_ref());
            let balance: u64 = 1000 * ((10 - i) as u64);
            let nonce: u32 = 0;
            state.insert(address, (nonce, balance));
        }
        State { map: state }
    }

    // other methods ...
}
```

#### State per block:
Since there is branching/forking in the blockchain, and the longest chain may change, you need to store one copy of **State** for each block. A copy of **State** for a block refers to the state after executing the block. We recommend using a HashMap-like storage, e.g., `HashMap<block hash, state>`. When you check transactions, you can get the corresponding state from it. When you update state, you do the update on a new state copy, and insert it.

The state could be shared by multiple threads. Since it is closely related to blocks, the easiest way is probably to make it a part of the `Blockchain` struct, e.g. add `hash_to_state: HashMap<H256, State>,` to the struct.

**Note**: the above paragraphs about **State** are one feasible way to implement it. You have the freedom to do it another way, as long as it enables transaction checks.


#### Transaction Mempool update:
After implementing state transition, ensure that the transactions in the mempool are valid with respect to the new state, this is necessary since some transactions may classify as double-spent after the state update, and you may need to remove those transactions.

---
### 6. Transaction generator
To demonstrate transaction is working well with the client, you need to add transactions to your running client. The transactions can be a simple payment in the account-based model. You are free to choose the sender and recipient.

In order to do that, you need to write a transaction generator. One recommended way to do that is to create a new thread, and generate a transaction periodically (we have provided a template in `src/transaction_generator.rs`; be sure to initialize it in `main.rs`). You may use other methods too, like writing an API in *src/api/* and call this API externally.

When a transaction is generated, add the transactions to mempool and broadcast the hash to the network.


**Note**: We do not ask you to implement transaction fees and mining rewards and the corresponding coinbase transaction for this project.

---
## Conclusion

Now that you have finished the last part, you have a simplified Bitcoin client! With transaction generator simulating user's transactions, the system should run smoothly and securely.

## Submission
The final grading of the project will be based on your demo video. In addition, please submit your code along with the demo video.


### Demo

You need to submit a two-minute video that demonstrates the execution of your code. (For example, if you use a Mac, you can use QuickTime Player to make a video recording of your computer screen.) For the demo you need to run your code on 3 nodes on the same computer and display the status at each node. Run the experiments at a suitable mining rate and a suitable transaction generation rate so that the video demonstrates the growth of the blockchains at the three nodes. Your video should be at a reasonable pace for the graders to follow without using slow motion. At the same time, your video should provide enough information for the grader to judge what requirements you have completed.

To get full credit, it is sufficient to show **any 4 of the following 5** bulleted items:
1. Initializing all three nodes and setting them in motion; showing communications between them;
2. periodical traces of a working miner and mempool on each node (which should be succinct and not overwhelm the screen, e.g., printing a one-line trace when a new block is mined/received, or some selected transactions are received);
3. the growth of the blockchain on each node, and that all nodes agree with each other (e.g., showing the hash and height of the tip, and the total count of the blocks);
4. the evolution of the ledger state on each node, and that all nodes agree with each other (e.g., showing the balances of certain accounts);
5. occasional invalid transactions are properly rejected and do not crash the program (i.e. the generators should take a small chance to generate invalid transactions).

If your bitcoin client is not fully completed, you can still get partial credits by showing how the individual parts work, for example:

6. traces of the generator generating random account-based transactions, signing them, adding them to the mempool, and broadcasting their hashes;
7. traces of the network exchanging messages;
8. traces of all sorts of validations when receiving a new block/transaction;
9. traces of one process working alone (instead of three processes working in harmony);
and so on.
