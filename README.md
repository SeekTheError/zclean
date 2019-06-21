# ZClean

This is the repository for ZClean, a hackaton project run at (twitter) @BinaryDistrict ZeroKnowledgeWorkshop(17/06/19 to 21/06/19)

### Goal

Making a very rough reimplementation of zero-cash like, adding a feature that coins are tainted (with a bit), and that while transferring in zero-knowledge we also prove that the taint is preserved. We only do shielded transactions, and everything has value 1.
The state of the blockchain is a merkle tree containing commitements and a list of nullifier.

Ref papers:

- [ZeroCash paper](http://zerocash-project.org/media/pdf/zerocash-oakland2014.pdf)

## Concept

### Components to the blockchain peer + wallet:

- JSON API
- Merkle Tree server
- Snarky Circuit to generate the proofs
- A Block
- a wallet with the secrets necessary for the peer to spend the notes
- a config with the port list

### Adding a block(transferring something):

- I want to transfer a note, I ask my peer to spend it by giving him the neccessary witness informations (aka I use my client, similar to using a full bitcoin node as a wallet) and he creates his own committement.
- it generates the necessary ZKproofs
- sends that to the receiver peer
- the receiver verify all proofs and
  - if successful broadcast it to the network
  - else do nothing/reject
- if it's accepted, the blockchain is now one block larger


### Design Principle

- utxo with 1 in 1 out, no values(or v = 1)
- to transfer, share a zk with the person you are transferring proving that you know the preimage


### Content of a Block:

- root
- nullifier
- id
- parentId
- parentHash
- new cm
- proof the the nullifier is computed correctl,that old commitment existed in the merkle tree, that the new commitment is well formed, that the flag is consistent (b_old = b_new)

### Prover file content:
Prover file `transfer_secrets` takes a lines of unlabelled values (some are tuples):

*  *r* - our secret with which to spend this commitment
*  *v* - the value of the spent commitment (1)
*  *flag* - the 'clean' flags of the spent commitment, eg 1 or 1155 ( =3*5*7*11 )
*  *l1_sibling* - a tuple of the field value of the sibling of the ancestor on the first level of the Merkel tree, along with the direction in the tree of this value (true/false), eg 34567890098765434567,false
*  *l2_sibling* - " " second level below root
*  *l3_sibling* - " " third level below root, ie the sibling of our commitment
*  *r'* - secret to attach to new commitment
*  *v'* - the value of the new commitment (1)
*  *flag'* - the 'clean' flags of the new commitment, should be same as old one, eg 1 or 1155 ( =3*5*7*11 )

The discards flag does not work in the current version of snarky and will cause a `Match_failure` exception due to unexpected file length
*  *discards* - always 1 in the case that the same flags are carried through


Something like:
```
998
1
0
4567,false
7385,true
2372,false
999
1
0
```

## Running the code

#### testing the raw commitment, importing your own commitment and Merkel tree data
`snarky_cli generate-keys create-coin-commitment.zk --curve Bn128`

`snarky_cli prove create-coin-commitment.zk 12123375568978657359272076418017180258742331462919823544269692050729161753928 11431946377964512669499131572242457906889915470048508955778605307618910933504
18327458394584782076418017180258742331462919823545938573930575720020284562850`

`snarky_cli verify create-coin-commitment.zk --proof create-coin-commitment_gen.zkp 12123375568978657359272076418017180258742331462919823544269692050729161753928 11431946377964512669499131572242457906889915470048508955778605307618910933504 15395739576093065984065038593017180258742331462919823544269487385743839305867`

The first paramteter to the proof is the root of a Merkle tree;
The second is the nullifier, ie your secret, hashed with the hash of (your secret concatenated with your spent commitment).
The third is the new commitment (does not need to be in the tree yet)

Other than requiring the correct chosen inputs, parameter 3 is independent from the others and parameter 1 depends on a tree calclulated  from parameter 2.


#### NB:
on first run, `snarky_cli generate-keys create-coin-commitment.zk --curve Bn128` gives error:
```
File "create-coin-commitment_gen.ml", line 4, characters 24-53:
Error: Signature mismatch:
       ...
       In module R1CS_constraint_system:
       Values do not match:
         val digest : t -> Core.Md5.t
       is not included in
         val digest : t -> Core_kernel.Md5.t
       File "src/backend_intf.ml", line 91, characters 4-27:
         Expected declaration
       File "src/libsnark.ml", line 945, characters 4-27: Actual declaration
```
Don't worry - run it again.

## Transport layer

The merkle tree and snarky circuit interface using a basic blockchain which is:
- Buit with Node.js
- Uses Redis as a datastore
- Uses the [buslane](https://www.npmjs.com/package/buslane) library for Wallet to Peer and Peer to Peer communication.

WARNING: as of time of writing, each component work separatly, but are not integrated

### Requirements:

- Docker and Docker Compose(to run the blockchain redis instances)
- Ocaml via opam, switched to 4.07.1
- Snarky installed ( good luck with that :D )
- node.js v12.*.* to run the client and the peer

### Starting the environment

- run `./install.sh` to install the node dependencies
- `docker-compose up -d` to start the redis instances
- in 3 of them, run `./startPeer.sh 0, 1 and 2` to run the peers
- in 3 of them, run `./startWallet.sh 0, 1 and 2` to start the wallets
- in 1 of them, run `./startExplorer.sh` to start the block explorer

### Usage of the wallet:

- To transfer a coin, type `t 0/1` in a wallet to transfer one coin to a peer
- To get the balance, type `b`
- To check the status or the peer your wallet is talking to, type `s`

- To reset the state, run `docker-compose down -v`


...
