---
title : Peeking into p2p world
---
We are familar with the the client server model. A single server listens for requests and serves the content corresponding to the request.
It could be a http server, ftp server etc.These servers are powerful and can handle a large amount of requests. 

<img src="/img/Client-Server Architecture.png" alt="Client-Servers" width="400px" height="300px">

In Peer to Peer(P2P) systems there is no central server serving the data. The peers provide each other with the data. The p2p system architecture can be partially decentralized or completely decentralized. When we look for p2p the first words that we come across are Bittorrent, Gnutella, Bitcoin. 
so let's have a look at Bittorrent protocol which is most widely used for file sharing.

## Bittorrent
The detailed Bittorrent specification can be found here[0]. I don't want to create another same copy but I will provide some key visualizations to understand better.

Some terms to know beforehand<br/>
**peer** : A machine that participates in file sharing can upload and download.<br/>
**client** : A user agent that acts as a peer on behalf of a user.<br/>
**tracker** : Holds information about peers in a swarm. It is a HTTP server responds to get requests.<br/>
**swarm** : A group of peers involved with a particular torrent.

   I am considering that the readers are familiar with steps in downloading torrent.Let's start with the torrent file. I had a torrent file of Big Bang theory episode and I opened it in a text editor.The file was partially readable with some text like '_announce_' and _links_ , remaining part was just weird ASCII characters.You can also have a look at any torrent file you have.The file is encoded using **_bencoding_**.

Bencoding has four datatypes string,integer and two compound types list and dicitionary.<br/>
length:string &emsp;example 4:bird<br/>
**i**integer**e** &emsp;example i7e<br/>
**l**&lt;any datatype&gt;**e** &emsp;example l3:foo2:ati91ee<br/>
This list is ["foo", "at", 91]<br/>
A list can have elements integer,string,dict,list.<br/>
**d**&lt;string which is key&gt;&lt;any datatype which is value&gt;**e**<br/> example d7:Algebrai45e3:Engli25ei67eee<br/>
This dict is { "Algebra" : 45 , "Eng" : [25, 67] }<br/>
dictionary must be sorted by keys.

The torrent file is bencode dictionary with the keys as follows

* announce
* announce-list
* comment
* created by
* creation date
* info 

info's value is a dictionary with keys

* for single file
  * name
  * piece length
  * pieces
  * length
* for multiple files
  * name
  * piece length
  * pieces
  * files  *list of dictionaries*
    * length
    * path

A file is divided into number of pieces where each piece is of length:piece length except for last piece.pieces is concatenation 20byte SHA1 hash value of all pieces as a single string and not list.For multiple files piece boundary may overlap files.Here is the big bang theory torrent printed using libtorrent library.![Torrent metainfo raw](/img/Screenshot_1.png)![Torrent metainfo](/img/Screenshot_2.png)
The second image is torrent file info with some infered information like number of pieces and info hash which is hash of the info value in the dictionary.

### Trackers 
tracker is an HTTP service which enables peer to join a swarm and locate other peers.It does not provide the data. It accepts a GET request with following parameters

#### Request

* info_hash  
  20-byte SHA1 hash value of the "info" key in metainfo file(.torrent)
* peer_id  
  20-byte peer ID.
* port  
  port number used by the torrent client
* uploaded  
  Total amount of bytes peer has uploaded
* downloaded  
  Total amount of bytes peer has downloaded
* left  
  Amount of bytes needed to complete
* ip
* numwant  
  Number of peers it wants.
* event  
  started,stopped,completed

Last three are optional. info_hash and peer_id have to be url encoded.

#### Response

* interval  
  A peer must send regular GET requests to the tracker.time in seconds
* leechers  
  number of peers downloading and uploading i.e incomplete files
* seeders  
  number of peers with entire file
* peers  
  List of dictionaries with
  * peer id
  * peer ip
  * peer port

At this point I opened the torrent in transmission app and also started wireshark to sniff packets with filter ```http.request.method == "GET" ``` but there was no activity.Later searching on Internet I found that the udp tracker protocol is different.The abbove image shows that the trackers are udp service. According to the specification[2]

#### UDP trackers
There are four messages

1. connect request
2. connect response
3. announce request
4. announce response

Timeout's are handled with resend of request after waiting for incremental amount of seconds.As it is possible to spoof the source ip in udp a connection id is used by tracker and given to client and for further request it is checked to verify.

*connect request*

|offset	|size	  |Name	     | value	|	
|-------|---------|----------|----------|	
|  0	| 64-bit integer	| connection_id	|	|
|  8	|32-bit integer	|action	    |	   0 //connect |
|  12	|32-bit integer	|transaction_id	|	|
|  16	|	|	|	|

<br/>
*connect response*

|offset	|size	  	|Name	     | value	|	
|-------|---------------|----------|----------|	
|  0	| 32-bit integer	| action | 0 //connect	|
|  4	|32-bit integer	|transaction_id    |	   |
|  8	|64-bit integer	|	connection_id|	|
|  16	|	|	|	|

<br/>
*announce request*

|offset	|size	  	|Name	     | value	|	
|-------|:-------------:|---------:|:--------:|	
|  0	| 64-bit integer	| connection_id	|	|
|  8	|32-bit integer	|action	    |	   1 //announce |
|  12	|32-bit integer	|transaction_id	|	|
|  16	|20-byte string	|info_hash	|	|
|  36	|20-byte string	|peer_id	|	|
|  56	|64-bit integer	|downloaded	|	|
|  64	|64-bit integer	|left		|	|
|  72	|64-bit integer	|uploaded	| 	|
|  80	|32-bit integer	|event	|0// 0:None 1:completed 2:started 3:stopped	|
|  84	|32-bit integer	|IP address	|0 //default	|
|  88	|32-bit integer	|key	|
|  92	|32-bit integer	|num_want	| -1 //default|
|  96	|16-bit integer	|port	| 	|
|98	|		|	|	|	

<br/>
*announce response:*

|offset	|size	  	|Name	     | value	|	
|-------|:---------------:|----------:|----------:|	
|  0	|32-bit integer	|action	    |	   1 //announce |
|  4	|32-bit integer	     |   transaction_id |	  |
|  8	|32-bit integer	|interval    |	  |
|  12	|32-bit integer	|leechers	    |	  |
|  16	|32-bit integer	|seeders	    |	  |
|  20 + 6 * n	|32-bit integer	|Ip address    |	  |
|  20 + 6 * n	|16-bit integer	|port    |	  |


Here is the video to demostrate.
<iframe width="560" height="315" src="https://www.youtube.com/embed/WxX0AjqQ28g" frameborder="0" allowfullscreen></iframe>







 

