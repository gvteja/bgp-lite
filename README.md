# bgp-lite
Application layer routing protocol that implements distance vector routing uses Bellman-Ford algorithm.  

The basic algorithm for this is as follows:  
input: a file with the link information for each neighbour and a timeout in sec  
 - link information includes source and destination interface, destination name, and the link weight/cost.  
 - timeout is the max amount of time to periodically read the link information file and/or max amount of time a node can go without updating its neighbours of its current dv.  

Algo:  
Read link info file and construct the cost vector, and interface mappings  
If there are n neighbours, we create n+1 threads.  
Each link of the current node has exactly one corresponding neighbour node.  
So, the first n threads will listen on the current node endpoint interface ip for a connection from their corresponding neighbour endpoint. The port to listen on is 9999.  
The last thread will send a connect request to all the neighbours on port 9999.  
  
We then join on all the above n+1 threads to complete.  

Once they complete, we now have 2 tcp connections for each neighbour. We use them in a half-duplex fashion and use one of them as only for reading and the other only for listening.  

Every node now locally maintains in memory:  
* distance vector (dv) of current node - vector containing estimate of shortest distance to every node, from the current node  
* next-hope nodes - next hop node (neighbour) on the shortest path for each destination node  
* dv for all neighbours  
* time since last update of dv to neighbours  
* time since the link information file was last read  

We now need to listen on the read sockets for any updates from neighbours. Since any neighbour can send dv at any time, we cannot do this synchronously. Hence, we use the select function in a while true loop, to asynchronously check for any sockets which have received an update from any of the neighbours or until a timeout value.  
Once select returns, we now have either have an update from the neighbour, or the call timed out.  
We initially set the update flag to false at the start of every loop.  
In case we have an update from a neighbour, we first read the data. The data is a json encoded dictionary thats contains the distance vector of the neighbour.  
Using the neighbours dv, we then do bellman ford algorithm to check if a better path exists for any of the destination through the neighbours new dv. If it does, we then update the current dv for the destinations with better path and change the corresponding next hop map to the neighbour through which we found the best path. If we found a better path, we also set the update flag to true, indicating the need to update neighbours of the new dv.  
We also check if time since last update to neighbours is more than the timeout value and set the update to true in case it is.  
Similarly, we check if time since the link information file was last read is greater than timeout period. In case it is, we read the link file to check for any link changes. In case there are any, we recompute the dv of current node to see if there are any changes. We set the update flag to True if there are any changes. We also set the time since the link information file was last read to the current time.  

We now check for the update flag. If its set, we then convert our current dv into json data and send that data to each of the write socket (to each neighbour). We update the last update time to the current time. We also dump the current dv, next hop nodes, and the dvs of all neighbours to files at this point.  

We set the select timeout to: timeout - later of (last update time, last link info read time). This is to make sure we do not wait for more than timeout seconds without update or read.

