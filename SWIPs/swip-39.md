---
SWIP: 39
title: Balanced Neighbourhood Registry aka Smart Neighbourhood Management
author: Viktor Trón <viktor@ethswarm.org> (@zelig)
discussions-to: https://discord.gg/Q6BvSkCv
status: Draft
type: Standards Track
category: Core
created: 2025-07-21
---

# Balanced Neighbourhood Registry aka Smart Neighbourhood Management

## Abstract

This SWIP introduces a systematic way for operators to enter a service network in such a way that their overlay addresses are balanced within the address space. Importantly,  operators need to register their commitment to take part and are assigned a random neighbourhood (of depth $d=int(log_2(N))+1$). 

## Motivation

There are multiple considerations that motivate such a scheme: 
- **load-balancing**: any decentralised service network will be fair if tasks are distributed to nodes so that the workload assigned to each participant is roughly equal. Such load balancing is achieved if tasks are assigned based on uniform random label (i.e., the content hash of the descriptor)[^1] and nodes providing the service are balanced in the address space.

[^1]: and the average size (computation/storage requirement) of tasks over a typical period of payment has tolerable variance.

- **arbitrary neighbourhood assigment**: the system needs to make sure that assignment of an overlay address to participant nodes is arbitrary. In particular, it is impractical (expensive) for any operator to attempt to place several nodes in the same storage neighbourhood (neighbourhood of depth $d=int(log_2(N))+1$). Note that, this scheme constitutes an effective solution to the problem of "one operator, one node in a neighbourhood". Taking the storage incentive system as an example, this will take care of the sybil issue[^2] without resorting to the rather weak incentive of additive stake as a proof of redundancy.[^3]

[^2]: that operators run several nodes in one neighbourhood without truely replicating storage and yet get paid.

[^3]: the idea is that if stake is variable and earnings are linearly proportional to earnings then, mutatis mutandis, it is always more economical for one operator to run just one node with all the stake than several nodes due to the added operational costs.

## Architecture

The balanced neighbourhood assignment (associated with a service) is orchestrated by a smart contract which is deployed together with a [staking contract](https://github.com/ethersphere/storage-incentives/blob/master/src/Staking.sol). The contract API provides 2 transactional endpoints relevant for entry: 

One registers a node by its ether address ($a^\Xi$) in the *committers' list* $C$ that records nodes' commitment to participate as a provider in the associated decentralised service network. 

The other one is called by the staking contract after the service network stake is deposited with a valid nonce, i.e., one that will put the node in the right neighbourhood. This call will place the node among the active node set for the service, and removes the entry from the committers list. This function includes a read-only call that takes as argument a node's  ether address and returns a neighbourhood the node is assigned to. This call is public so that the client can enquire about the neighbourhood they are assigned to -- so that they can mine an overlay address into it, ie., find a nonce that is needed to generate the overlay address. 

## Specification

### Registration
An initially empty list (*committers' list*) of *entry struct* types holds the current committers. The struct holds information about the ether address of the node, the blockheight the address registered at. 
<!-- For each new registrant, the number $N$ is incremented.  -->


#### Deposit
In order for a node to get its address registered, an amount  of $S_0$ (stake zero) must be deposited. 

#### Uniqueness
In order to prevent repeated trials, each node must be registered only once. 

After checking the deposit amount and the uniqueness check on the ether address, the current blockheight is recorded with the address by pushing the entry struct ($e_n=<a^\Xi,h>$) to the end of committers list.

#### Validity 
The entry is valid for a period of $B$ blocks after the registration[^55].

[^55]: $B$ is less than $256$, the number of blocks available from within the EVM).

Since the blockheight values of the list items are monotonically increasing, entries at the beginning of the list expire first. By iterating upto the first valid entry, expired entries can be iterated on efficiently. 

### Expiry

This function call iterates through all expired entries, burns their deposit, and, by setting the head  of the list to the first valid item, removes them from the committer's list.

This is called by the assign function (itself called by the staking contract) before the read only call checking if the resulting overlay address falls into the neighbourhood that the registrant was assigned to, i.e., the correctness of the nonce submitted from the perspective of the staking contract. 

### Assign

The assign call is the second transactional endpoint called by the staking contract. It takes the provider's ether address and as well as the mined overlay as arguments.
After calling expiry, the validity of the registration is checked by finding the entry for the ether address in the committers' list.

#### Initialisation 

Let $\overline{N}=2^d$ be the lowest power of 2 that is  greater than $N$,    the number of already assigned registrants, and let $d$ be its exponent ($d=int(log_2(N))+1$). Let $R$ be the array of the remaining unassigned neighbourhoods of this level (i.e., $len(R)=\overline{N}-N$). Whenever $len(R)$ drops to $0$, $d$ is incremented and $\overline{N}=2^d$ adjusts. At any point in time, a $uint256$ array of length $\overline{N}$ is maintained, called the *assignments list* $A$ holding the registered nodes' overlay addresses.
Whenever $d$ changes, new arrays for assigments $A_d$ and remainders $R_d$ are created (both twice the size of the previous one $A_{d-1}$ and $R_{d-1}$). We iterate through the current array and for each overlay address $a^O_i$ at position $i$ copy $a^O_i$ to position $2\cdot i +b_i$ where $b_i=1$ if the $d$-th bit of the address is set:

$$
b_i=a^O_i[d/8]
$$

$$
b_i\gg=7-(d\mod 8)
$$


$$
b_i{\land\hspace{-3pt}=}1
$$

In fact, $A_d$ stands for nodes of a binary tree on level $d$ and the value at each position is the overlay address filling that neighbourhood. When $d$ is incremented, we will need to fill some of the neighbourhoods with the existing nodes. Each node will fill the left child node (when $b_i=0$) or the right one ($b_i=1$) depending on the subsequent bit in their overlay.

$$
\forall 0\leq i < 2^d, A_d[2\cdot i+b_i]=A_{d-1}[i]
$$

As we are filling the assignment list, we know that the whenever a neighbourhood is filled with an already existing node, its sister neighbourhood will be unassigned, therefore we can just record those in the remaining list.

$$
\forall 0\leq i < len(R_d), R_d[i]=2\cdot i+1-b_i
$$


#### Random seed

This internal read-only call takes as its single argument an ether address ($a^\Xi$) and returns a random $uint256$. 
After checking if the ether address is a valid registrant by finding the corresponding first (unique) $entry$ struct,[^33] the `difficulty` (randao) is called on the block subsequent to the blockheight registered. Append to it the ether address and hash it to yield what will serve as the random seed for this provider.[^4]

$$
\sigma:=\mathit{blockAtHeight}(\mathit{entry}(a^\Xi).\mathit{height}+1).\mathit{difficulty}()
$$

$$
\varrho:=\mathit{uint256}(H(\sigma|a^\Xi))
$$

[^33]: Since expiry is not necessarily called when the random seed it called, the blockheight needs to be checked: 1) if block after $h$ (the one in which the registration happened) is available, 2) that the height is not greater than $h+D$ with $D$ being the validity period in blocks.

[^4]: Even if on  a POA chain, and no randao, this seed cannot be known to the registering provider and their colluding  associates, but nonetheless should be deterministic once its set.

#### Neighbourhood

This component must be available as a public readonly endpoint taking a node's ($n$) ether address ($a^\Xi_n$)  as a single argument.
The random nonce $\varrho_n$ is used to select a neighbourhood $nh$ for a provider from the remaining unassigned neighbourhood list of level $d$:

$$
i:=\varrho\mod len(R_d)
$$

$$
nh=R[i]
$$

#### Checking the overlay

The overlay (obtained by mining the nonce) is checked to fall in the correct neighbourhood r:
The check validates the address $a^O_n$ if and only if:

$$
r=a^O_n\gg(255-d)
$$

#### Assignment

If the overlay check passes, 

- the nodes' overlay address is assigned to a neighbourhood of depth $d$.

$$
A_d[r]=a^O_n
$$

- $N$ is incremented
- the $i$-th item is removed from remaining open neighbourhood list $R_d$.
- if $R_d$ is now of zero length, the $d$ is incremented, and new assignment and remaining lists are initialised as per section 'initialisation' above.
- provider's entry is removed from the committers' list.


### Further endpoints

A public read-only endpoint exists for querying neighbourhoods as well as nodes. Accessor for $d$ and $N$ will return the current neighbourhood depth and the current number of assigned neighbourhoods. A public accessor for $A_d$ will  return for a neighbourhood (between $0$ and $2^d-1 inclusive) the overlay of the node assigned to that neighbourhood. Another endpoint will return for any overlay $o$ the closest node, so that the network service can find responsible nodes for any task with address in the space shared by overlays:

$$
g(a)=A_d[a\gg(255-d)]
$$

### Deregistration

Only called from the Staking contract, deregister deletes the entry for the neighbourhood belonging to the given address, makes the neighbourhood available in $R$


## Implementation notes

### Changes to the staking contract

### Changes to the bee client

A new endpoint to bee client must be added to register a node that is not yet registered to be assigned a neighbourhood. Once the neighbourhood is known, the client can mine the nonce needed to place the overlay in the required neighbourhood.

### Migration

Since a new updated staking contract, a stake migration will be needed for the upgrade. Before the change, all the simplification of the staking contract is recommended, especially to allow fixed stake  in order to realign redundancy 
of storage and monetary incentive: with a fixed amount staked, total stake is linearly proportional to the number of nodes, and therefore comparisons across neighbourhoods can be made based on the number of nodes. In particular, the arbitrary balanced assignment makes sense in terms of incentives (expected revenue).

### Putting a node in each neighbourhood.

## Contract

```sol	
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract BalancedNeighbourhoodRegistry {
    // --------------------
    // Configurable values
    // --------------------
    uint256 public constant STAKE = 10,000,000,000,000,000 wei;  // Stake required
    uint256 public constant VALID_FOR = 128;       // Validity window in blocks

    // --------------------
    // Structs and Storage
    // --------------------
    struct Entry {
        address committer;
        uint256 height;
    }

    Entry[] public committers; // List of committers (registered but as yet unassigned nodes)
    mapping(address => bool) public hasCommitted; // Track if an address has committed

    uint256 public N;      // Number of assigned nodes
    uint256 public d = 1;  // Neighbourhood depth
    uint256 public currentPower = 2; // 2^d

    // Assignments: overlay address for neighbourhoods of depth d
    bytes32[] public A;

    // Remaining unassigned neighbourhoods of depth d
    uint256[] public R;

    // --------------------
    // Events
    // --------------------
    event Registered(address indexed node, uint256 blockHeight);
    event Assigned(address indexed node, uint256 neighbourhood, bytes32 overlay);
    event DepthUpgraded(uint256 newDepth);
    event Unassigned(address indexed node, uint256 neighbourhood);

    // --------------------
    // Registration endpoint
    // --------------------
    function register() external payable {
        require(msg.value == STAKE, "Invalid stake");
        require(!hasCommitted[msg.sender], "Already registered");

        committers.push(Entry(msg.sender, block.number));
        hasCommitted[msg.sender] = true;

        emit Registered(msg.sender, block.number);
    }

    // --------------------
    // Expire old entries
    // --------------------
    function _expire() internal {
        while (committers.length > 0) {
            Entry storage e = committers[0];
            if (block.number <= e.height + VALID_FOR) {
                break;
            }
            hasCommitted[e.committer] = false;
            _removeCommitter(0);
            // Burn logic: funds stay locked. 
        }
    }

    // --------------------
    // Remove committer from the list
    // --------------------
    // This function is used internally to remove a committer from the list.
    // It shifts the elements to the left and pops the last element.
    function _removeCommitter(uint index) internal {
        if (index >= committers.length) return;

        for (uint i = index; i < committers.length - 1; i++) {
            committers[i] = committers[i + 1];
        }
        committers.pop();
    }

    // --------------------
    // Find entry for a committer in the committer list
    // Returns the index of the entry if found, otherwise reverts.
    // --------------------
    function _findEntryFor(address _a) internal view returns (uint) {
        for (uint i = 0; i < committers.length; i++) {
            if (committers[i].committer == _a) {
                return i;
            }
        }
        revert("Not registered");
    }

    // --------------------
    // Randomness
    // --------------------
    function _randomSeed(address _a) internal view returns (uint256) {
        uint i = _findEntryFor(_a);
        uint256 h = committers[i].height;
        // Ensure the block number is valid as expire may not have been called
        require(block.number > h + 1, "Too early");
        require(block.number <= h + VALID_FOR, "Registration expired"); 
        // Use blockhash to generate a random seed
        bytes32 bh = blockhash(h + 1);
        return uint256(keccak256(abi.encodePacked(bh, _a)));
    }

    // --------------------
    // Public View: Neighbourhood
    // --------------------
    function getNeighbourhood(address _a) public view returns (uint256) {
        uint256 r = _randomSeed(_a);
        require(R.length > 0, "No available neighbourhoods");
        return R[r % R.length];
    }

    // --------------------
    // Assign node overlay to neighbourhood
    // --------------------
    function assign(address _a, bytes32 _overlay) external {
        _expire();

        // Check registration
        uint256 nh = getNeighbourhood(_a);
        uint256 overlayNh = uint256(_overlay) >> (256 - d);

        require(overlayNh == nh, "Overlay doesn't match neighbourhood");

        A[nh] = _overlay;
        N++;

        // Remove neighbourhood from R
        _removeFromR(nh);

        // Check if R is empty
        if (R.length == 0) {
            _upgradeDepth();
        }

        hasCommitted[_a] = false;
        _removeEntry(_a);

        emit Assigned(_a, nh, _overlay);
    }

    //----------------------
    // unregister
    //----------------------
    function unregister(address _a, bytes32 _overlay) external {
        uint nh = uint256(_overlay) >> (256 - d);
        A[nh] = bytes32(0);
        R.push(nh);
        N--;
        // return funds to the committer
        payable(_a).transfer(STAKE);
        emit Unassigned(_a, nh);
    }   

    // --------------------
    // Internal functions to manage R and committers
    // --------------------
    function _removeFromR(uint256 nh) internal {
        for (uint i = 0; i < R.length; i++) {
            if (R[i] == nh) {
                // replace the removed element with the last element and pop               
                R[i] = R[R.length - 1];
                R.pop();
                return;
            }
        }
    }

    function _removeEntry(address _a) internal {
        uint i = _findEntryFor(_a);
        require(i < committers.length, "Entry not found");
        _removeCommitter(i);
    }

    // --------------------
    // Expand A and R when needed
    // --------------------
    function _upgradeDepth() internal {
        currentPower = 2 ** d;

        R = new uint[](currentPower);
        // A.length *= 2 // does this not work to extend to its double with zero values?
        for (uint i = 0; i < currentPower; i++) {
            // Ensure A has enough space for the new neighbourhoods);   
            A.push(bytes32(0));
        }
        for (uint i = currentPower - 1; i < currentPower; i--) {
            uint b = uint256(A[i]) >> (256 - d) % 2;
            A[2*i+b] = A[i];
            uint256 j = 2*i+1-b;
            if (2 * j < currentPower) {
              A[j] = bytes32(0); // Clear the old address
            }
            R[i] = j;
        }
        d++;
        require(d <= 32, "Maximum depth exceeded");
        emit DepthUpgraded(d);
    }

    // --------------------
    // Accessors
    // --------------------
    function getDepth() external view returns (uint) {
        return d;
    }

    function getN() external view returns (uint256) {
        return N;
    }

    function getOverlayForNeighbourhood(uint256 nh) external view returns (bytes32) {
        return A[nh];
    }

    function getClosestNode(bytes32 key) external view returns (bytes32) {
        uint256 prefix = uint256(key) >> (256 - d);
        return A[prefix];
    }
}
```

