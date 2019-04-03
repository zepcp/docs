# Sequence diagrams

Traditional systems like API's present simple scenarios, in which a centralized service defined how data should be encoded.

However, decentralized ecosystems like a distributed vote system need much stronger work on defining every interaction between any two peers on the network.

- [Sequence diagrams](#sequence-diagrams)
  - [Prior to voting](#prior-to-voting)
    - [Contract deployment (Entity)](#contract-deployment-entity)
    - [Contract deployment (Process)](#contract-deployment-process)
    - [Set Entity metadata](#set-entity-metadata)
    - [Entity subscription](#entity-subscription)
    - [Custom requests to an Entity](#custom-requests-to-an-entity)
      - [Sign up](#sign-up)
      - [Submit a picture](#submit-a-picture)
      - [Make a payment](#make-a-payment)
      - [Resolve a captcha](#resolve-a-captcha)
    - [Adding users to a census](#adding-users-to-a-census)
  - [Voting](#voting)
    - [Voting process creation](#voting-process-creation)
    - [Voting process retrieval](#voting-process-retrieval)
    - [Check census inclusion](#check-census-inclusion)
    - [Casting a vote with ZK Snarks](#casting-a-vote-with-zk-snarks)
    - [Casting a vote with Linkable Ring Signatures](#casting-a-vote-with-linkable-ring-signatures)
    - [Registering a Vote Batch](#registering-a-vote-batch)
  - [After voting](#after-voting)
    - [Checking a submitted vote](#checking-a-submitted-vote)
    - [Closing a Voting Process](#closing-a-voting-process)
    - [Vote Scrutiny](#vote-scrutiny)

---

## Prior to voting

--------------------------------------------------------------------------------

### Set Entity metadata
An Entity starts existing at the moment it has certain metadata stored on the [EntityResolver](/architecture/components/entity?id=entityresolver) smart contract. 

```mermaid

sequenceDiagram
    participant PM as Process Manager
    participant DV as dvote-js
    participant GW as Gateway/Web3
    participant ER as Entity Resolver contract

    PM->>DV: getEntityId(entityAddress)
    DV-->>PM: entityId

    alt has a Gateway
        Note right of PM: Set Metamask <br/> to the Gateway
    else
        Note right of PM: Boot a GW and tell <br/> Metamask to use it
    end

    PM->>DV: getDefaultResolver()
        Note right of DV: Hardcoded default Entity<br/>Resolver instance
    DV-->>PM: resolverAddress


    PM->>PM: Fill-up name
    PM->>+DV: EntityResolver.setName(entityId, name)
        DV->>GW: setText(entityId, "name", name)
            GW->>ER: <transaction>
            ER-->>GW: 
        GW-->>DV: 
    DV-->>-PM: 

    loop additional key/values
        PM->>PM: Fill-up key-value
        PM->>+DV: EntityResolver.set(entityId, key, value)
            DV->>GW: setText(entityId, key, value)
                GW->>ER: <transaction>
                ER-->>GW: 
            GW-->>DV: 
        DV-->>-PM: 
    end

```

**Used schemas:**

- [Entity metadata](/architecture/components/entity.md)

### Entity subscription

#### Initial discovery

A user wants to locate an initial list of entities without any prior information.

```mermaid

sequenceDiagram
    participant App
    participant DV as dvote-js
    participant BS as Bootstrap Server
    participant GW as Gateway/Web3
    participant ER as Entity Resolver contract

    alt has predefined entityId and resolver
        App->>App: readConfigParams()
    else 
        App->>DV: getBootGateways()
        DV->>BS: GET /gateways
        note right of BS: Hardcoded addresses<br/>Provided by Vocdoni
        BS-->>DV: gatewayRef[]
        DV-->>App: gatewayRef[]

        App->>DV: getBootSettings()
        DV-->>App: (entityId, resolver)
    end
    App->>DV: getBootEntities(resolver, entityId)
        DV->>GW: text(entityId, "vndr.vocdoni.entities.boot")
            GW->>ER: text(entityId, "vndr.vocdoni.entities.boot")
            ER-->>GW: entityRef[]
        GW-->>DV: entityRef[]
    DV-->>App: 

```

#### Listing boot entities

A user wants to visualize a list of entities so he/she can eventually subscribe to one.

```mermaid

sequenceDiagram
    participant App
    participant DV as dvote-js
    participant GW as Gateway/Web3
    participant ER as Entity Resolver contract
    participant IPFS as Ipfs/Swarm

    loop entityRef[]
        App->>DV: Entity.fetch(entityId, resolver)
        alt Blockchain and Swarm respond
            loop
                DV->>GW: text(entities[i], "vndr.vocdoni.meta")
                    GW->>ER: text(entities[i], "vndr.vocdoni.meta")
                    ER-->>GW: metadataContentUri
                GW-->>DV: metadataContentUri
                DV->>GW: fetchFile(metadataContentUri)
                    GW->>IPFS: Ipfs.get(metadataContentUri)
                    IPFS-->>GW: entityMetadata
                GW-->>DV: entityMetadata
            end
        else P2P fetching fails
            DV->>GW: text(entities[i], "vndr.vocdoni.name")
            GW->>ER: text(entities[i], "vndr.vocdoni.name")
            ER-->>GW: entityName
            GW-->>DV: entityName

            loop core values
                DV->>GW: text(entities[i], key)
                GW->>ER: text(entities[i], key)
                ER-->>GW: data
                GW-->>DV: data
            end
        end
        DV-->>App:  entityMetadata
    end

    App->>App: Display Entites
    App->>App: Subscribe
```

**Used schemas:**

- [Entity metadata](/architecture/components/entity.md)

**Related:**

- [Gateway Boot Nodes](/architecture/components/entity?id=gateway-boot-nodes)

**Notes:**

- In the case of React Native apps, DVote JS will need to run on the [Web Runtime component](/architecture/general?id=web-runtime-for-react-native)

### Custom requests to an Entity

An Entity may have specific requirements on what users have to accomplish in order to join a be part of its user registry.

Some may require filling a simple form. Some others may ask to log in from an existing HTTP service. Uploading ID pictures, selfies or even making payments need custom implementations that decide that a user must eventually be added to a census.

Below are some examples:

#### Sign up

The user selects an action from the entityMetadata > actions available.

```mermaid
sequenceDiagram
    participant App
    participant UR as WebView<br/>User Registry
    participant DB as Internal Database

    App->>UR: Go to: <action-url>?publicKey=0x1234&censusId=0x4321
        activate UR
            Note right of UR: Fill the form
        deactivate UR

        UR->>UR: signUp(name, lastName, publicKey, censusId)

        UR->>DB: insert(name, lastName, publicKey, censusId)
    UR-->>App: success
```

**Used schemas:**

- [Entity metadata](/architecture/components/entity?id=entity-metadata)

**Notes:**

- `ACTION-URL` is defined on the metadata of the contract. It is expected to be a full URL to which GET parameters will be appended (`publicKey` and optionally `censusId`)

#### Submit a picture

#### Make a payment

#### Resolve a captcha

#### Adding users to a census

Depending on the activity of users, an **Entity** may decide to add public keys to one or more census.

```mermaid
sequenceDiagram
    participant PM as Process Manager
    participant DB as Internal Database
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant CS as Census Service

    PM->>DB: getUsers({ pending: true })
    DB-->>PM: pendingUsers


    loop pendingUsers
        PM->>DV: Census.addClaim(censusId, censusMessagingURI, claimData, web3Provider)
            activate DV
                DV->>DV: signRequestPayload(payload, web3Provider)
            deactivate DV

            Note right of DV: TO DO: Review calls

            DV->>GW: addCensusClaim(addClaimPayload)
                GW->>CS: addClaim(claimPayload)
                CS-->>GW: success
            GW-->>DV: success
        DV-->>PM: success
    end
```

**Used schemas:**

- [Census Service - addClaim](/architecture/components/census-service?id=census-service-addclaim)
- [Census Service - addClaimBulk](/architecture/components/census-service?id=census-service-addclaimbulk)

--------------------------------------------------------------------------------

## Voting

### Voting process creation

```mermaid
sequenceDiagram
    participant PM as Process Manager
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant CS as Census Service
    participant SW as Swarm
    participant BC as Blockchain

    Note right of DV: TO DO: Review calls

    PM->>+DV: Process.create(processDetails)

        DV->>+GW: censusDump(censusId, signature)
            GW->>+CS: dump(censusId, signature)
            CS-->>-GW: merkleTree
        GW-->>-DV: merkleTree

        alt Linkable Ring Signatures
            loop modulusGroups
                DV-->>GW: addFile(modulusGroup) : modulusGroupHash
                    GW-->SW: Swarm.put(modulusGroup) : modulusGroupHash
                GW-->>DV: 
            end
        else ZK-Snarks
            DV-->>GW: addFile(merkleTree) : merkleTreeHash
                GW-->SW: Swarm.put(merkleTree) : merkleTreeHash
            GW-->>DV: 
        end

        DV->>+GW: getCensusRoot(censusId)
        GW->>+CS: getRoot(censusId)
        CS-->>-GW: rootHash
        GW-->>-DV: rootHash

        DV-->>GW: addFile(processMetadata) : metadataHash
            GW-->SW: Swarm.put(processMetadata) : metadataHash
        GW-->>DV: 

        DV->>+GW: Process.create(entityId, name, metadataContentUri)
            GW->>+BC: <transaction>
            BC-->>-GW: txId
        GW-->>-DV: txId

        DV->>+GW: EntityResolver.set(entityId, 'vndr.vocdoni.processess.active', activeProcesses)
            GW->>+BC: <transaction>
            BC-->>-GW: txId
        GW-->>-DV: txId

    DV-->>-PM: success
```

**Used schemas:**

- [Process Metadata](/architecture/components/process?id=process-metadata-json)
- [Modulus group list](/architecture/components/census-service?id=modulus-group-list)
- [Census Service - addClaimBulk](/architecture/components/census-service?id=census-service-addclaimbulk)
- [Census Service - getRoot](/architecture/components/census-service?id=census-service-getroot)
- [Census Service - setParams](/architecture/components/census-service?id=census-service-setparams)
- [Census Service - dump](/architecture/components/census-service?id=census-service-dump)


### Voting process retrieval

A user wants to retrieve the voting processes of a given Entity

```mermaid
sequenceDiagram
    participant App as App user
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant BC as Blockchain
    participant SW as Swarm

    App->>+DV: Process.fetchByEntity(entityAddress, resolver)

        DV->>GW: EntityResolver.text(entityId, "vndr.vocdoni.processes.active")
            GW->>BC: text(entityId, "vndr.vocdoni.processes.active")
            BC-->>GW: processId[]
        GW-->>DV: processId[]

        loop processIDs

            DV->>GW: getMetadata(processId)
                GW->>BC: getMetadata(processId)
                BC-->>GW: (name, metadataContentUri, merkleRootHash, relayList, startBlock, endBlock)
            GW-->>DV: (name, metadataContentUri, merkleRootHash, relayList, startBlock, endBlock)

            alt Process is active or in the future
                DV->>GW: fetchFile(metadataHash)
                GW->>SW: Swarm.get(metadataHash)
                SW-->>GW: processMetadata
                GW-->>DV: processMetadata
            end
        end

    DV-->>-App: processesMetadata
```

**Used schemas:**

- [Process Metadata](/architecture/components/process?id=process-metadata-json)

### Check census inclusion

A user wants to know whether he/she belongs in the census of a process or not.

The request can be sent through HTTP/PSS/PubSub. The response may be fetched by subscribing to a topic on PSS/PubSub.

```mermaid
sequenceDiagram
    participant App as App user
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant CS as Census Service

    Note right of DV: TO DO: Review calls (LRS)

    App->>+DV: Census.hasClaim(publicKey, censusId, censusMessagingURI)

        DV->>+GW: genCensusProof(censusId, publicKey)
        GW->>+CS: genProof(censusId, publicKey)
        CS-->>-GW: isInCensus
        GW-->>-DV: isInCensus

    DV-->>-App: isInCensus
```

**Used schemas:**

- [Census Service - generateProof](/architecture/components/census-service?id=census-service-generateproof)

**Notes:**

- `generateProof` may be replaced with a call to `hasClaim`, for efficiency
- The `censusId` and `censusMessagingURI` should have been fetched from the [Process Metadata](/architecture/components/process)

### Casting a vote with ZK Snarks

Requests can be sent through HTTP/PSS/PubSub. Responses may be fetched by subscribing to a topic on PSS/PubSub.

```mermaid
sequenceDiagram

    participant App
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant CS as Census Service
    participant RL as Relay

    App->>+DV: Process.castVote(vote, processMetadata, merkleProof?)

        alt merkleProof not provided

            DV->>+GW: generateProof(processMetadata.census.id, publicKey)

            GW->>+CS: PSS.broadcast(<generateProofData>)
            CS-->>-GW: merkleProof

            GW-->>-DV: merkleProof

        end

        DV->>DV: computeNullifier()

        DV->>DV: encrypt(vote, processMetadata.publicKey)

        DV->>DV: generateZkProof(provingK, verificationK, signals)

        DV->>DV: encryptVotePackage(package, relay.publicKey)

        DV->>GW: submitVote(encryptedVotePackage, relay.messagingUri)

            GW->>RL: transmitVoteEnvelope(encryptedVotePackage)
            RL-->>GW: ACK

        GW-->>DV: submitted

    DV-->>-App: submitted
```

**Used schemas:**

- [Process Metadata](/architecture/components/process?id=process-metadata-json)
- [Census Service - generateProof](/architecture/components/census-service?id=census-service-generateproof)
- [Vote Package - ZK Snarks](/architecture/components/relay?id=vote-package-zk-snarks)

**Notes:**

- The Merkle Proof could be retrieved and stored beforehand

### Casting a vote with Linkable Ring Signatures

Requests can be sent through HTTP/PSS/PubSub. Responses may be fetched by subscribing to a topic on PSS/PubSub.

```mermaid
sequenceDiagram

    participant App
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant CS as Census Service
    participant RL as Relay

    App->>+DV: Process.castVote(vote, processMetadata, censusChunk?)

        Note right of DV: TO DO: Review calls

        alt censusChunk not provided

            DV->>+GW: getChunk(publicKeyModulus)

            GW->>+CS: getChunkData(modulus)
            CS-->>-GW: censusChunk

            GW-->>-DV: censusChunk

        end

        DV->>DV: encrypt(vote, processMetadata.publicKey)

        DV->>DV: sign(processMetadata.address, privateKey, censusChunk)

        DV->>DV: encryptVotePackage(package, relay.publicKey)

        DV->>GW: submitVote(encryptedVotePackage, relay.messagingUri)

            GW->>RL: transmitVoteEnvelope(encryptedVotePackage)
            RL-->>GW: ACK

        GW-->>DV: submitted

    DV-->>-App: submitted
```

**Used schemas:**

- [Process Metadata](/architecture/components/process?id=process-metadata-json)
<!-- - [getChunk](/architecture/components/census-service?id=census-service-getchunk) -->
- [Vote Package - Ring Signature](/architecture/components/relay?id=vote-package-ring-signature)

**Notes:**

- The `publicKeyModulus` allows to segment the whole census into `N` polling stations. Every public key is assigned to exactly one, depending on the modulus that yields a division by `processMetadata.census.modulusSize`.

### Registering a Vote Batch

```mermaid
sequenceDiagram
    participant RL as Relay
    participant SW as Swarm
    participant BC as Blockchain Process

    activate RL
        RL->>RL: loadPendingVotes()
        RL->>RL: skipInvalidVotes()
    deactivate RL

    RL-->SW: Swarm.put(voteBatch) : batchHash

    RL->>+BC: Process.registerBatch(batchContentUri)
    BC-->>-RL: txId
```

**Used schemas:**

- [Vote Batch](/architecture/components/relay?id=vote-batch)

## After voting

### Checking a submitted vote

The sequence diagram applies to both **ZK Snarks** and **LRS** Vote Packages. `nullifierOrSignature` will be interpreted according to the process' `type` on its metadata.

```mermaid
sequenceDiagram
    participant App
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant RL as Relay
    participant BC as Blockchain Process
    participant SW as Swarm

    App->>DV: checkVoteStatus(processAddress, relayMessagingUri)

        DV->>DV: computeNullifierOrSignature()

        DV->>+GW: getVoteStatus(processAddress, nullifierOrSignature, relayMessagingUri)

            GW-->>RL: requestVoteStatus(processAddress, nullifierOrSignature)
            RL-->>GW: (batchId?, batchContentUri?)

        GW-->>-DV: (batchId?, batchContentUri?)

        alt it does not trust the batchContentUri

            DV->>+GW: Process.getBatch(batchId)
            GW->>+BC: Process.getBatch(batchId)
            BC-->>-GW: (processAddress, batchContentUri)
            GW-->>-DV: (processAddress, batchContentUri)

        end

        DV->>+GW: fetchFile(batchHashURI)
        GW->>+SW: Swarm.get(batchHash)
        SW-->>-GW: batch
        GW-->>-DV: batch

        DV->>DV: checkWithinBatch(nullifierOrSignature, batch)

    DV-->>App: isRegistered
```

**Used schemas:**

- [Vote Batch](/architecture/components/relay?id=vote-batch)

**Notes:**

- `nullifierOrSignature` is expected to contain a nullifier when the process `type` is `zk-snarks`
- `nullifierOrSignature` is expected to contain a ring signature when the process `type` is `lrs`

### Closing a Voting Process

```mermaid
sequenceDiagram
    participant PM as Process Manager
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant BC as Blockchain Process

    PM->>DV: Process.close(processAddress, privateKey)

        DV->>+GW: Process.close(processAddress, privateKey)
        GW->>+BC: Process.close(processAddress, privateKey)

            BC->>BC: checkPrivateKey(privateKey)
            BC->>BC: closeProcess(processAddress)

        BC-->>-GW: success
        GW-->>-DV: success

    DV-->>PM: success
```

### Vote Scrutiny

Anyone with internet access can compute the scrutiny of a given processAddress. However, the vote batch data needs to be pinned online for a certain period of time.

```mermaid
sequenceDiagram
    participant SC as Scrutinizer
    participant DV as DVote JS
    participant GW as Gateway/Web3
    participant BC as Blockchain Process
    participant SW as Swarm

    SC->>+DV: Process.get(processAddress)

        DV->>+GW: Process.get(processAddress)
        GW->>+BC: Process.get(processAddress)
        BC-->>-GW: (name, metadataContentUri, privateKey)
        GW-->>-DV: (name, metadataContentUri, privateKey)

    DV-->>-SC: (name, metadataContentUri)

    SC->>+DV: Swarm.get(metadataHash)

        DV->>+GW: fetchFile(metadataHash)
        GW->>+SW: Swarm.get(metadataHash)
        SW-->>-GW: processMetadata
        GW-->>-DV: processMetadata

    DV-->>-SC: processMetadata

    SC->>+DV: Process.getVoteBatchIds(processAddress)

        DV->>+GW: Process.getVoteBatchIds(processAddress)
        GW->>+BC: Process.getVoteBatchIds(processAddress)
        BC-->>-GW: batchIds
        GW-->>-DV: batchIds

    DV-->>-SC: batchIds

    SC->>+DV: Process.fetchBatches(batchIds)
        loop batchIds

            DV->>+GW: Process.getBatch(batchId)
            GW->>+BC: Process.getBatch(batchId)
            BC-->>-GW: (type, relay, batchContentUri)
            GW-->>-DV: (type, relay, batchContentUri)

            DV->>+GW: fetchFile(batchHash)
            GW->>+SW: Swarm.get(batchHash)
            SW-->>-GW: voteBatch
            GW-->>-DV: voteBatch

        end
    DV-->>-SC: voteBatches

    SC->>+DV: skipInvalidRelayBatches(voteBatches, processMetadata.relays)
    DV-->>-SC: validRelayBatches

    SC->>+DV: skipInvalidTypeBatches(validRelayBatches, processMetadata.type)
    DV-->>-SC: validTypeBatches

    SC->>SC: sort(merge(validTypeBatches))

    SC->>+DV: resolveDuplicates(voteBatches)
    DV-->>-SC: uniqueVotePackages

    alt type=zk-snarks
        loop uniqueVotePackages

            SC->>+DV: Snark.check(proof, votePackage.publicSignals)
            DV-->>-SC: valid

        end
    else type=lrs

        SC->>SC: groupByModulus(uniqueVotePackages)
        loop voteGroups

            SC->>+DV: LRS.check(signature, voteGroup.pubKeys, processAddress)
            DV-->>-SC: isWithinGroup

            SC->>+DV: LRS.isUnseen(signature, processedVotes.signature)
            DV-->>-SC: isUnseen

        end
    end

    loop validVotes

        SC->>+DV: decrypt(vote.encryptedVote, privateKey)
        DV-->>-SC: voteValue

        SC->>SC: updateVoteCount(voteValue)

    end

    SC->>+DV: addFile(voteSummary)
        DV-->GW: addFile(voteSummary)
        GW-->SW: Swarm.put(voteSummary)
        SW-->>GW: voteSummaryHash
        GW-->>DV: voteSummaryHash
    DV-->>-SC: voteSummaryHash

    SC->>+DV: addFile(voteList)
        DV-->GW: addFile(voteList)
        GW-->SW: Swarm.put(voteList)
        SW-->>GW: voteListHash
        GW-->>DV: voteListHash
    DV-->>-SC: voteListHash
```

**Used schemas:**

- [Process Metadata](/architecture/components/process?id=process-metadata-json)
- [Vote Package - ZK Snarks](/architecture/components/relay?id=vote-package-zk-snarks)
- [Vote Package - Ring Signature](/architecture/components/relay?id=vote-package-ring-signature)
- [Vote Batch](/architecture/components/relay?id=vote-batch)
- [Vote Summary](/architecture/components/relay?id=vote-summary)
- [Vote List](/architecture/components/relay?id=vote-list)