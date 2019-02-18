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

- `client` is a JS-only site that runs a IPFS node, talks to
  the `servers` and displays the comments. This site can be
  hosted on `github.io` since it doesn't have any server-side
  code.
- `server` is a VPS with a static IP address that runs a IPFS
  node and stores comments posted by the `clients`. The `server`
  publishes its IP address and port to the DHT using the
  hardcoded `network-id` DHT key.
- `network-id` is a random 192/256-bit value used as the DHT key
  to make `servers` discoverable by `clients` and other `servers`.

1. The extension grabs the tab URL and gives it to the client.
   The URL is given in the fragment part because there is no need
   to send it to the server. The URL itself isn't used; only
   its hash is used.
    ```
    client5.github.io/#<URL>
    ```
2. The client uses the DHT (WebTorrent DHT or IPFS DHT) to find
   servers in the network. It uses the hardcoded network id as the
   DHT key:
    ```bash
    for prov in $(ipfs dht findprovs $networkid); do
      timeout 3 ipfs dht findpeer $prov;
    done
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
   all the comments for all the URLs it knows.


```
  +--------+
  | client |           Web
  +--------+
      |
--- 1.| -------------------
      v
+-------------+
| http server |        VPS
+-------------+
      |    |
    2.|    +--------+
      |           4.|
      v             v
+----------+ +-----------+
| HTTP GET | | HTTP POST |
+----------+ +-----------+
      |
      |      +---------+
    3.|      | watcher |
      |      +---------+
      v
+-------------+  
| ipfs daemon | 
+-------------+
```

The `server` or VPS consists of 4 processes:

- The `http server` process listens on a certain port and accepts
  the `GET` and `POST` requests from the `clients`. It doesn't do
  the work, but rather forwards requests to the designated handlers.
- The `HTTP GET` handler is a process that works on the `GET`
  requests. It runs the `ipfs add` command that can be slow sometimes.
- The `HTTP POST` handler is a process that works on the `POST`
  requests. It creates files with new comments and doesn't need to
  use IPFS commands.
- `ipfs daemon` runs in its own process.
- The `watcher` process keeps an eye on all the other processes and
  restarts them if needed. This especially applies to the IPFS
  daemon that tends to leak memory.

1. The `client` sends a `GET` request to fetch comments for a URL.
2. The `http server` forwards the `GET` to the `HTTP GET` handler.
3. The `HTTP GET` handler runs `ipfs add` to generate a IPFS folder
   id for the comments.
4. If the `client` sends a `POST` request to add a comment, the
   request is processed by the `HTTP POST` handler that creates a
   file with this comment. It doesn't need access to the IPFS daemon.
