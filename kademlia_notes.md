# notes from _Kademlia: A Peer-to-peer Information System Based on the XOR Metric_

* Kademlia is a p2p distributed hash table (DHT)
  * minimizes amount of configuration messages nodes send to learn about each
    other
  * configuration spreads automatically via key lookups
  * nodes have the knowledge and flexibility to route queries through low
    latency paths
  * uses parallel asynchronous queries to avoid timeout delays from failed nodes
  * the algorithm which nodes use to record each other's existence is resistent
    to some denial of service attacks
  * uses novel XOR metric for distance between points in the key space
    * XOR is symmetric
      * Kademlia participants receive lookup queries from precisely the same
        distribution of nodes contained in their routing tables
      * Kademlia can send a query to any node within an interval
        * allows it to select routes based on latency
        * allows to send parallel asynchronous queries to several equally
          appropriate nodes
* Kademlia uses a single routing algorithm from start to finish in order to
  locate nodes near a particular ID
* assigns 160-bit opaque IDs to nodes
  * provides a lookup algorithm that locates successively closer nodes to any
    desired ID - converges to the lookup target in logarithmically many steps
* nodes are treated as leaves on a binary tree
  * each node's position is determined by the shortest unique prefix of its ID
  * for any given node - divide the binary tree into a series of successively
    lower subtrees that don't contain the node
  * the highest subtree consists of the half of the binary tree not containing
    the node
  * the nest subtree consists of the half of the remaining tree not containing
    the node
    * this pattern repeats
  * ensures every node knows of at least 1 node in each of its subtrees
    * this guarantees that any node can locate any other node by its ID

## XOR metric

* each Kademlia node has a 160-bit node ID
* every message that a node transmits includes its node ID
* keys are also 160-bit identifiers
  * assign ```(key, value)``` pairs to particular nodes
    * relies on a notion of distance between 2 identifiers
* in a fully populated binary tree of 160-bit IDs - the magnitude of the
  distance between 2 IDs = height of the smallest subtree containing both IDs

## node state

* Kademlia nodes store contract information about each other to route query
  messages
  * for each ```0 <= i < 160``` every nodes keeps a list of IP address, UDP
    port, and node ID for nodes of distance between _2<sup> i</sup>_ and
    _2<sup> i+1</sup>_ from it self
    * these lists are called k-buckets
* each k-bucket is kept sorted by time last seen
  * the least recently seen node at the head and the most recently seen node at
    the tail end
  * if the values of _i_ are small the k-buckets are generally empty
    * no appropriate nodes will exist
  * if the values of _i_ are large the lists can grow up to size _k_
    * _k_ is a system wide replication parameter
    * _k_ is chosen so that any given _k_ nodes are very unlikely to fail within
      an hour of each other
* when a Kademlia node receives any message (request or reply) from another node:
  * updates the appropriate k-bucket for the sender's node ID
    * if sending node already exists in recipient's k-bucket
      * then the recipient moves it to the tail of the list
    * if the node is not already in the appropriate k-bucket and the bucket
      has fewer than _k_ entries
      * then the recipient inserts the new sender at the tail of the list
    * if appropriate k-bucket is full
      * then the recipient pings the k-bucket's least recently seen node
        and decides what to do from there
    * if least recently seen node fails to respond
      * then it is evicted from the k-bucket and new sender is inserted at
        the tail end
    * if least recently seen node responds
      * then it is moved to the tail of the list and new sender's contract
        is discarded
* k-buckets implement a least recently seen eviction policy
  * but live nodes are never removed
* the longer a node has been up - the more likely it is to remain up for
  another hour
  * by keeping the oldest live contacts around - k-buckets maximize the
    probability that the nodes they contain will remain online
* k-buckets provide resistance to certain DoS attacks
  * the routing state of the nodes cannot be flushed by flooding the system
    with new nodes
  * Kademlia nodes will only insert the new nodes in the k-buckets when old
    nodes leave the system

## Kademlia protocol

* consists of 4 RPCs
  1. ```PING```
     * probes nodes to check if it is online
  2. ```STORE```
     * instructs a node to store a ```(key, value)``` pair for later retrieval
  3. ```FIND_NODE```
     * takes a 160-bit ID as an argument
     * recipient returns ```(IP address,UDP port, Node ID)``` for the _k_ nodes
       it knows about which are closest to the target ID
       * these can come from a single k-bucket or from multiple k-buckets if the
         closest k-bucket is not full
     * if there are fewer than _k_ nodes in all of the recipients k-buckets
       combined - it returns every node it knows about
  4. ```FIND_VALUE```
     * returns ```(IP address, UDP port, Node ID)
     * if the RPC recipient has received a ```STORE``` RPC for the key it only
       returns the stored value
* in all RPCs - the recipient must echo a 160-bit random RPC ID
  * provides some resistance to address forgery
* ```PINGS``` can be piggy backed on RPC replies for the RPC recipient to obtain
  additional assurance of the sender's network address

### node lookup

* most important procdure which Kademlia participants perform
* locates the _k_ closest nodes to some given node ID
* employs a recursive algorithm for node lookups
* lookup initiator starts by picking ```a``` nodes from its closest non empty
  k-bucket
  * if the bucket has fewer than ```a``` entries - it takes the ```a``` closest
    nodes it knows of
  * the initiator then sends parallel asynchronous ```FIND_NODE``` RPC to the
    ```a``` nodes it has chosen
  * is a system wide concurrency parameter

### recursive step

* initiator resends the ```FIND_NODE``` to nodes it has learned about from
  previous RPCs
* recursion can begin before all ```a``` od the previous RPCs have returned
* of the _k_ nodes - the initiator has heard of closest to the target - it picks
  ```a``` that it has not yet queried and resends the ```FIND_NODE``` RPC to them
* nodes that fail to respond quickly are removed from consideration until and
  unless they do not respond
* if a round of ```FIND_NODES``` fails to return a node any closer than the
  closet already seen node
  * the initiator resends the ```FIND_NODE``` to all pof the _k_ closest nodes
    it has not already queried
  * the lookup terminates when the initiator has queried and received responses
    from the _k_ closest nodes it has seen
* to store ```(key, value)``` pairs - participants locate the _k_ closest nodes
  to the key and sends them ```STORE``` RPCs
* each node re-publishes ```(key, value)``` pairs as necessary
  * this keeps them alive and ensures persistence of ```(key, value)``` pairs
    with high probability
* every original publisher of the ```(key, value)``` pair is required to
  republish it every 24 hours (can be changed to a different scale)
  * if not republished - it expires 24 hours after publication
    * limits stale index information in the system

### to find a (key, value) pair

* a node starts by performing a lookup to fond the _k_ nodes with IDs closest to
  the key
* value lookups use ```FIND_VALUE``` rather than ```FIND_NODE``` RPCs
* procedure halts immediately when any node returns the value
* once a lookup succeeds - requesting node stores the ```(key, value)``` pair at
  the closest node it observed to the key that did not return the value
* the topology is unidirectional
  * future searches for the same key are likely to hit cached entries before
    querying the closest node
  * while a certain key is highly popular the system may cache it on many nodes
    * to avoid over-caching:
      * the expiration time of a ```(key, value)``` pair in any node's database
        exponentially inversely proportional to the # of nodes which are between
        the current node and the node whose ID is closest to the key ID
* each node refreshes any bucket that has not performed a node lookup in the
  past hour
* to join the network - a node ```u``` must have a contact to an already
  participating node ```w```
  * ```u``` inserts ```w``` into the appropriate k-bucket
  * ```u``` performs a node lookup for its own node ID
  * ```u``` refreshes all k-buckets further away than its closest neighbor
    * during these refreshes ```u``` populates its own k-bucket and inserts
      itself into the k-buckets of other nodes as necessary

## routing table

* Kademlia's basic routing table structure is a binary tree
  * the leaves are represented by k-buckets
  * each k-bucket contains nodes with some common prefix of their IDs
    * this prefix is the k-bucket's position in the binary tree
  * each k-bucket covers some range of the ID space
  * together the k-buckets cover the entire 160-bit ID space with no overlap
* nodes in the routing tree are allocated dynamically as needed

## efficient key re-publishing

* nodes must periodically republish keys in order to ensure persistence of 
  ```(key, value)``` pairs
* Kademlia republishes each ```(key, value)``` pair once an hour to compensate
  for nodes leaving the network
* given ```k``` number of nodes storing ```(key, value)``` pairs
  * perform a node look up followed by ```k - 1 STORE``` RPCs every hour 
  * when a node receives a ```STORE``` RPC for a given ```(key, value)``` pair
    it assumes the RPC was also issued to the other ```k - 1``` nodes
    * recipient will not republish the ```(key, value)``` pair in the next hour
  * ensures that as long as republication intervals are not exactly synchronized
    * only 1 node will republish a given ```(key, value)``` pair every hour
* to counter unbalanced trees and avoid performing node lookups before
  republishing keys - the nodes split k-buckets as required
  * this ensures they have complete knowledge of a surrounding subtree with at
    least ```k``` nodes
  * if a node ```u``` refreshes all k-buckets in a subtree of ```k``` nodes
    before republishing ```(key, value)``` pairs - the node will automatically
    be able to figure out the ```k``` closest node to a given key
    * bucket refreshes can be amortized over the republication of many keys
* when a new node joins a system it must store any ```(key, value)``` to which
  it is one of the ```k``` closest
  * existing nodes know which ```(key, value)``` pairs the new node should store
  * any node learning of a new node will issue ```STORE``` RPCs to transfer
    relevant ```(key, value)``` pairs to the new node
  * a node only transfers a ```(key, value)``` pair if it's own ID is closer to
    the key than the IDs of other nodes
    * this avoids redundant ```STORE``` RPCs

## sketch of proof

* for the Kademlia system to work - it must be proven that most operations take
  _[log n] + c_ time for some small constant _c_ and a _(key, value)_
  lookup returns a key stored in the system with overwhelming probability
* for k-bucket covering the distance range _[2<sup> i</sup>, 2<sup> i+1</sup>]_
  * define the index of the bucket - as _i_
  * define the depth _h_ of a node - as _160 - i_
    * _i_ is the smallest index of a non-empty bucket
  * define node _y_'s bucket height (in node _x_) - as the index of the bucket
    where _x_ inserts _y_ minus the index of _x_'s least significant empty
    bucket
  * it is probable that the height of any given node will be within a constant
    of _log n_ for a system with _n_ nodes
    * bucket height of the closest node to an ID in the _k_<sup>th</sup> closest
      node will likely be within a constant of _log k_

### assume the invariant

* every k-bucket of every node contains at least 1 contact - if a node exists=
in the appropriate range
* the node lookup procedure is correct and takes logarithmic time
* if the closest node to the target ID has depth _h_
* if node of this node's _h_ most significant k-buckets is empty
  * the lookup procedure will find a node half as close in each step
  * distance is 1 bit shorter
  * turns up the node in _h - log k_ steps
  * if 1 of the node's k-buckets is empty - it could be the case that the
  target node resides in the range of the empty bucket
  * if the case that the target node resides in the range of the empty bucket
  * the final steps will not decrease the distance by half
  * the search will proceed exactly as though the bit in the key
    corresponding to the empty bucket had been flipped
  * the lookup algorithm will always return the closest node in
    _h - log k_ steps
  * when the closest node is found - the currency switches from _a_ to _k_
  * the number of steps to find the remaining _k - 1_ closest nodes can be
    no more than the bucket height of the closest node in the
    _k_<sup>th</sup> closest node
    * unlikely to be more than a constant plus _log k_

### prove the correctness of the invariant

* the effects of bucket refreshing if the invariant holds
* after being refreshed - a bucket will either:
  * contain _k_ valid nodes
  * or contain every node in its range if fewer than _k_ exist
* new nodes that join will also be inserted into any buckets that are not full
* only way to violate the invariant is if there exist _k + 11_ or more nodes in
  the range of a particular bucket and the _k_ actually contained in the bucket
  to fails with no intervening lookups or refreshes

### the probability of failure

* failure probability is much smaller than the probability of _k_ nodes
  leaving within an hour
  * every incoming or outgoing request updates nodes' buckets
  * this results from the symmetry of the XOR metric
    * because the IDs of the nodes (where a given node communicates during an
      incoming or outgoing requests) are distributed exactly compatibly with the
      node's bucket ranges
* if the invariant fails for a single bucket in a single node - it only affects
  running time - by adding a hop to some lookups
  * a lookup will fail if _k_ nodes on a lookup path each lose _k_ nodes in the
    same bucket with no intervening lookups or refreshes

### (key, value) pair's recovery
* when _(key, value)_ pair is published - it is populated at the _k_ nodes
  closest to the key
  * it is re-published every hour
* less reliable/new nodes have a 50% probability of lasting 1 hour
  * after 1 hour the _(key, value)_ pair will still be present on one of the
    _k_ nodes closest to the key with the probability of _1 - 2<sup> -k</sup>_
  * this property of the Kademlia system is not violated by the insertion of
    new nodes that are close to the key
    * when the node is inserted - it contacts the closest node - to fill its
      buckets and receive any nearby _(ky, value)_ pairs needed to be stored
    * if the _k_ closest nodes to a key fail and the _(key, value)_ pair has
      not been cached elsewhere - the key is not stored (it is lost)