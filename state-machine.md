# The Filecoin State Machine

The majority of Filecoin's user facing functionality (payments, storage market, power table, etc) is managed through the Filecoin State Machine. The network generates a series of blocks, and agrees which 'chain' of blocks is the correct one. Each block contains a series of state transitions called `messages`, and a checkpoint of the current `global state` after the application of those `messages`. 

The `global state` here consists of a set of `actors`, each with their own private `state`.

An `actor` is the Filecoin equivalent of Ethereum's smart contracts, it is essentially an 'object' in the filecoin network with state and a set of methods that can be used to interact with it. Every actor has a Filecoin balance attributed to it, a `state` pointer, a `code` CID which tells the system what type of actor it is, and a `nonce` which tracks the number of messages sent by this actor. (TODO: the nonce is really only needed for external user interface actors, AKA `account actors`. Maybe we should find a way to clean that up?)

### Method Invocation
There are two routes to calling a method on an `actor`.

First, to call a method as an external participant of the system (aka, a normal user with Filecoin) you must send a signed `message` to the network, and pay a fee to the miner that includes your `message`.  The signature on the message must match the key associated with an account with sufficient Filecoin to pay for the messages execution. The fee here is equivalent to transaction fees in Bitcoin and Ethereum, where it is proportional to the work that is done to process the message (Bitcoin prices messages per byte, Ethereum uses the concept of 'gas'. We will likely also use 'gas').

Second, an `actor` may call a method on another actor during the invocation of one of its methods.  However, the only time this may happen is as a result of some actor being invoked by an external users message (note: an actor called by a user may call another actor that then calls another actor, as many layers deep as the execution can afford to run for).

### State Representation

The `global state` is modeled as a map of actor addresses to actor structs. This map is implemented by an ipld HAMT (TODO: link to spec for our HAMT). Each actor's `state` is an ipld pointer to a graph that can be entirely defined by the actor.

### Execution

Message execution currently relies entirely on 'built-in' code, with a common external interface. All method invocations have a set of parameters, which are a cbor encoded array of the parameters set by the function definition.

These functions are given, as input, an 'ExecutionContext' containing useful information for their execution.

```go
type VMContext interface {
	// Message is the message that kicked off the current invocation
	Message() Message
	
	// Storage provides access to the VM storage layer
	Storage() Storage
	
	// Send allows the current execution context to invoke methods on other actors in the system
	Send(to address.Address, method string, value *types.AttoFIL, params []interface{}) ([][]byte, uint8, error)
	
	// AddressForNewActor (TODO: this seems unnecessary in this interface)
	AddressForNewActor() (address.Address, error)
	
	// BlockHeight returns the height of the block this message was added to the chain in
	BlockHeight() *types.BlockHeight
	
	// IsFromAccountActor (TODO: this also feels unnecessary)
	IsFromAccountActor() bool

	// CreateNewActor is used to create a new actor from the given code and constructor
    // parameters (TODO: address should probably not be a parameter)
	CreateNewActor(addr address.Address, code cid.Cid, initalizationParams interface{}) error
}
```



#### Storage

Actors are given acess to a `Storage` interface to fulfil their need for persistent storage. The `Storage` interface describes a content addressed block storage system (`Put` and `Get`) and a pointer into it (`Head` and `Commit`) that points to the actor's current state.

```go
type Storage interface {
	// Put writes the given object to the storage staging area and returns its CID
	Put(interface{}) (Cid, error)
    
	// Get fetches the given object from storage (either staging, or local) and returns
	// the serialized data.
	Get(Cid) ([]byte, error)
    
	// Commit updates the actual stored state for the actor. This is a compare and swap
	// operation, and will fail if 'old' is not equal to the current return value of `Head`.
	// This functionality is used to prevent issues with re-entrancy
	Commit(old Cid, new Cid) error
    
	// Head returns the CID of the current actor state
	Head() Cid
}
```

Actors can store state as a single block or implement any persistent 
data structure that can be built upon a content addressed block store. 
Implementations may provide data structure implementations to simplify 
development. The current interface only supports CBOR-IPLD, but this 
should soon expand to allow other types of IPLD data structures (as long
as the system has resolvers for them).

The current state of a given actor can be accessed first by calling `Head` to retrieve the CID of the root of the actors state, then by using `Get` to retrieve the actual object being referenced.

To store data, `Put` is used. Any number of objects may be `Put`, but only the object whose CID is committed, or objects that are linked to in some way by the committed object will be kept. All other objects are dropped after the method invocation returns. Objects stored via `Put` are first marshaled to CBOR-IPLD, and then stored, the returned CID is a 32 byte sha2-256 CBOR-IPLD content identifier.

