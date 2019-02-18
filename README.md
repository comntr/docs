```
               +------------------+
               | Chrome Extension |
               +------------------+
                         |
                       1.|
                         |
                         v
      +-----+   2   +--------+  5  +------+ 
   +->| DHT |<------| client |---->| IPFS |
   |  +-----+       +--------+     +------+
 6.|                 |     |
   |               3.|   4.|
   |  +----------+   |     |
   |  | server 1 |<--+     |
   |  +----------+         |
   |       ^               |
   |     7.|               |
   |       |               |        
   |  +----------+         |
   +--| server 2 |<--------+
      +----------+
```

Terminology:

- `client` is a JS-only site that runs a IPFS node, talks to
  the `servers` and displays the comments. This site can be
  hosted on `github.io` since it doesn't have any server-side
  code.
- `server` is a VPS with a static IP address that runs a IPFS
  node and stores comments posted by the `clients`. The `server`
  publishes its IP address and port to the DHT using the
  hardcoded `network-id` DHT key.
- `network-id` is a random 256-bit value used as the DHT key
  to make `servers` discoverable by `clients` and other `servers`.

Call flow:

1. The extension grabs the tab URL and gives it to the client.
   The URL is given in the fragment part because there is no need
   to send it to the server:
    ```
    client5.github.io/#<URL>
    ```
2. The client uses the DHT (WebTorrent DHT or IPFS DHT) to find
   servers in the network. It uses the hardcoded network id as the
   DHT key:
    ```
    ipfs dht findprovs <network-id>
    ```
   The DHT finds 2 servers and returns their IP addresses and ports:
    ```
    [server 1] 11.22.33.44:12345
    [server 2] 22.33.44.55:54321
    ```
3. The client sends a `GET /hash(URL)` to server 1 to get comments
   for this URL. The server uses `ipfs add` to publish the folder
   with comments to IPFS and returns the `<folder-id>`.
4. The client sends a `POST /hash(URL)` to add a comment for the URL.
5. The client gets files for `<folder-id>` from IPFS.
6. Server 2 uses the DHT to find other servers.
7. Server 2 sends a `GET /` to server 1 to get a IPFS folder id for
   all the comments for all the URLs it knows. This is how servers
   sync comments between each other.

