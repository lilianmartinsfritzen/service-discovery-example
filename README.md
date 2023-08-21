# Service Discovery and Consul

## Introduction

Discovering services in a distributed environment can be challenging. With multiple machines being created and removed, load balancers in play, questions arise on how to determine which service is up, where to direct traffic, and ensuring health. The [Consul](https://www.consul.io/) tool will be used to understand the subject practically, it offers Service Discovery along with various other functionalities.

### Common Scenario in Distributed Applications

In scenarios where applications communicate, like App A calling App B which in turn calls App C, knowing the machine being called becomes crucial. This involves knowing the IP, port, or subdomain linked to an IP through DNS.

![Scenario 1](https://github.com/lilianmartinsfritzen/service-discovery-example/assets/83084256/e71fcbf6-7505-44a7-ab99-377ad6123f40)

As the usage of App B increases, new instances are created to manage load. This raises questions about which instance of App B App A should call, selection criteria, and readiness of new instances.

![Scenario 2](https://github.com/lilianmartinsfritzen/service-discovery-example/assets/83084256/fc2e9dbc-f366-437d-8be6-79bff6b24f9d)

Challenges emerge:

- Which machine to call?
- Which port to use?
- Do I need instance IPs?
- How to ensure an instance's health?
- How to confirm permission to access?

## Key Concepts of Service Discovery

- Discover available machines for access
- Segment machines for security
- Resolve via DNS
- Active health checks
- Verify access permission

## [HashiCorp](https://www.hashicorp.com/) [Consul](https://developer.hashicorp.com/consul/docs)

Consul is often associated with Service Discovery due to its mature and lightweight approach. It offers a wide array of features, including:

- **Service Segmentation:** Specify services inaccessible to certain services.
- **Layer 7 Load Balancing:** Balancing internet traffic outside the network.
- **Key / Value Configuration (Storage):** Store configuration variables, secrets, etc.
- **Open Source / Enterprise**

### IMPORTANT - SECURITY

To enable service communication, Mutual TLS is utilized, ensuring certificates and incorporating [Service Mesh](https://en.wikipedia.org/wiki/Service_mesh) and [Proxies](https://en.wikipedia.org/wiki/Proxy_server).

- Mutual Authentication: [Learn more](https://en.wikipedia.org/wiki/Mutual_authentication)
- TLS (Transport Layer Security): [Learn more](https://en.wikipedia.org/wiki/Transport_Layer_Security)

<aside>
💡 When working with Kubernetes, Service Discovery is essential. Consul can be run in Kubernetes environments.

</aside>

### Consul & Centralized Service Registry

The Service Registry contains all services, along with Health Check information.

Imagine three instances of Application B, all of them have Consul agents, continuously reporting to the server, maintaining a consolidated service registry. Health Checks are conducted locally; if an issue arises, Consul removes the machine from the registry.

This communication uses the [Gossip Protocol](https://en.wikipedia.org/wiki/Gossip_protocol). Consul features its DNS server, allowing DNS queries for available machines.

It also offers an API for querying and obtaining information from the Service Registry.

### Health Check

Consul's Agent runs on each server, configured to access addresses (URLs, Endpoints, TCP Ports). If the agent stops responding, it notifies the Consul Server, which then removes the service from the Registry.

The Consul Server doesn't continually check the Client for service availability. Instead, the Client Agent verifies service health, updating the Server.

### Multi-region & Multi-cloud

- Multi-language
- Multi-operating system
- Consul is not just multi-cloud; it allows communication between Consul instances in different clouds, ensuring consistency.
- Consul's "[Datacenter](https://en.wikipedia.org/wiki/Data_center)" concept facilitates managing multiple clusters to meet specific needs.

### Agent, Client, and Server

Main functions summarized:

- **Agent:** A daemon process executed on all cluster nodes, running in Client or Server Mode.
- **Client:** Locally registers services, performs Health Checks, forwards service information to Server.
- **Server:** Maintains cluster state, registers services, handles client/server membership, and more.

### Dev Mode

- Simulate Consul servers
- **NEVER** use in production
- For testing features, examples, and proof of concepts
- Runs as a server
- Does not scale
- Registers everything in memory

### Starting a Consul Agent as a Study Example

The example provided in this repository does not cover security aspects, therefore, for production environments, it's necessary to encrypt communication between Consul clusters using TLS certificates and encryption among members.

```bash
❯ docker compose up
```

### Getting Started

1. Clone this repository and navigate to the project directory.
2. Run `docker-compose up` to start the demonstration environment.
3. Run each service in root mode using the command `docker-compose exec <service-name> sh`

### Starting a Consul Agent for Study

Consul has three modes: client, server, and dev.

```bash
# command example
❯ consul agent -dev
```

You can see the cluster members and their information using the following command:

```bash
❯ consul members
```

### Core Concepts

- Always start at least 3 servers for stability and leader election.
- Consul replicates information automatically from the leader to other servers.
- Consul offers a REST API accessible via HTTP and includes a built-in DNS server.
- Consul provides SDKs to simplify service catalog interaction and caching.

### Accessing the Catalog

Use the following command to access the catalog of the cluster:

```bash
❯ curl localhost:8500/v1/catalog/nodes
```

### Building a Cluster with 3 Servers

1. Configure three Consul instances in the `docker-compose.yaml`.
2. Start each agent in server mode with the appropriate parameters, including `-server`, `-bootstrap-expect`, `-node`, `-bind`, `-data-dir`, and `-config-dir`.

### Bringing Up Servers

To start a server agent:

1. Determine your machine's local IP using `ifconfig`.
2. Use the following command to launch the agent (replace placeholders as needed):

```bash
❯ consul agent -server -bootstrap-expect=3 -node=<node-name> -bind=<local-ip> -data-dir=/var/lib/consul -config-dir=/etc/consul.d 
```

### Bringing Up Clients

1. Create directories `/etc/consul.d` and `/var/lib/consul` in the client's container.
2. Start the client agent using the following command:

```bash
❯ consul agent -bind=<client-ip> -data-dir=/var/lib/consul -config-dir=/etc/consul.d 
```

## Registering the First Service with Consul

### Creating a Configuration File

1. Create a `.json` file, you can name it anything, for example, let's use `services.json`.

In the file, you'll describe:

- `"id"`: A unique identifier for the service.
- `"name"`: The name of the service.
- `"tags"`: Tags that help with service discovery (e.g., "production", "development", "web").
- `"port"`: The port on which the service is running.
- Other parameters available in the Consul documentation.

```json
{
  "service": {
    "id": "nginx",
    "name": "nginx",
    "tags": [
      "web"
    ],
    "port": 80
  }
}
```

Registering a service differs from starting a service. When you register the service, you're not starting it yet. Registration and service deployment are independent. Always run an agent on the same machine as the service.

After configuring, use `consul reload` to update Consul with registered services.

```bash
❯ consul reload
```

### Accessing the Service via DNS

```bash
❯ dig @localhost -p 8600 nginx.service.consul

# expected output
;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.18.0.3
```

You can see the service `nginx.service.consul` running on IP `172.18.0.3`. Any machine can see the service by using the same command.

### API Service Search
You can also use the Consul API to fetch information about the registered services. Use the following command to list the available services:

```bash
❯ curl localhost:8500/v1/catalog/services

# expected output
{"consul":[],"nginx":["web"]}

```

### Adding a Second Nginx Service

To register another `nginx` service, create a JSON configuration similar to the first one but with a unique `id`.

```json
{
  "service": {
    "id": "nginx2",
    "name": "nginx",
    "tags": [
      "web"
    ],
    "port": 80
  }
}
```

Bring up a new client, and using `-retry-join`, the client automatically discovers the entire cluster.

```bash
❯ consul agent -bind=172.19.0.6 -data-dir=/var/lib/consul -config-dir=/etc/consul.d -retry-join=172.19.0.2 -retry-join=172.19.0.3
```

The second `nginx2` service synchronizes, and the join to the cluster is successful.

### Implementing Health Check

If we don't implement this check service, Consul will always show the registered services regardless of whether they are available or not. This is why adding the Health Check feature becomes so useful.

Consul offers various ways to implement Checks, as documented in [Define health checks](https://developer.hashicorp.com/consul/docs/services/usage/checks).

We'll add the check only to the `consul01` service, and to apply the changes, we need to run the command `consul reload`.

First, let's verify the services:

```bash
❯ dig @localhost -p 8600 nginx.service.consul

# expected output
;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.18.0.6
nginx.service.consul.   0       IN      A       172.18.0.3
```

We run `consul reload` on the machine where the check was added, i.e., `consulclient01`:

```bash
❯ consul reload
Configuration reload triggered

# expected output
2023-08-19T19:28:55.199Z [WARN]  agent: Check is now critical: check=nginx
```

After running the command `dig @localhost -p 8600 nginx.service.consul` again, we only see the service from `client02`:

```bash
❯ dig @localhost -p 8600 nginx.service.consul

# expected output
;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.18.0.6
```

If we try to check via the API, we'll get the following response since we don't have anything installed. It's important to note that registering a service is one thing, and actually running the service is another. Up to this point, we haven't even installed Nginx yet.

```bash
❯ curl localhost
curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused
```

Installing and starting Nginx:

```bash
❯ apk add nginx
❯ nginx
❯ ps
PID   USER     TIME  COMMAND
  213 root      0:00 nginx: master process nginx
  214 nginx     0:00 nginx: worker process
  215 nginx     0:00 nginx: worker process
  216 nginx     0:00 nginx: worker process
  217 nginx     0:00 nginx: worker process
  218 nginx     0:00 nginx: worker process
  219 nginx     0:00 nginx: worker process
  220 nginx     0:00 nginx: worker process

❯ curl localhost
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

But when we look at the terminal of `client01` where Consul is running, the service is still critical:

```bash
2023-08-19T19:35:25.284Z [WARN]  agent: Check is now critical: check=nginx
```

We need to configure Nginx:

```bash
# Adding a text editor
❯ apk add vim

# Creating a directory using -p to create recursively if any of the folders don't exist
❯ mkdir /usr/share/nginx/html -p

#

 Opening the default Nginx configuration file
❯ vim /etc/nginx/http.d/default.conf

Replace:
      # Everything is a 404
      location / {
              return 404;
      }

# with:
      root /usr/share/nginx/html;

# Now, edit the content of the file that will be displayed
❯ vim /usr/share/nginx/html/index.html

Hellooo... finally!!!

# Restart Nginx to apply the changes
❯ nginx -s reload
```

After reloading Nginx, the service status should change in the terminal:

```bash
# expected output
2023-08-19T22:22:13.955Z [INFO]  agent: Synced check: check=nginx
```

Synchronized Check. Running `curl [localhost](<http://localhost>)` should display the content we added to the HTML:

```bash
❯ curl localhost
# expected output
Hellooo... finally!!!
```

Now we can verify via DNS if the `Client01` service is up and running:

```bash
❯ dig @localhost -p 8600 nginx.service.consul

# expected output
;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.19.0.6
nginx.service.consul.   0       IN      A       172.19.0.5
```

We can now kill the Nginx process to see if the Check actually works and if the service becomes unavailable after being online:

```bash
# List processes
❯ ps
PID   USER     TIME  COMMAND
   87 root      0:00 nginx: master process nginx

# Kill the process and then query via DNS
❯ kill 87
❯ dig @localhost -p 8600 nginx.service.consul

;; ANSWER SECTION:
# expected output
nginx.service.consul.   0       IN      A       172.19.0.6

# Automatically, Consul on Client01 indicates that the service is critical
2023-08-19T22:29:14.071Z [WARN]  agent: Check is now critical: check=nginx
```

After restarting Nginx, wait for 10 seconds, and the service should be visible again:

```bash
❯ nginx
❯ dig @localhost -p 8600 nginx.service.consul

# expected output
;; ANSWER SECTION:
nginx.service.consul.   0       IN      A       172.19.0.5
nginx.service.consul.   0       IN      A       172.19.0.6

# Also in the terminal of Consul Client01
2023-08-19T22:30:24.137Z [INFO]  agent: Synced check: check=nginx
```

## Syncing Servers via Configuration File

In this example, we synchronize servers by using a JSON configuration file placed in the `server` directory. This file contains all the necessary parameters to start the servers within it.

```json
// In the case below, the "server" parameter indicates the type of agent being started
// TRUE = SERVER and FALSE = CLIENT
{
  "server": true,
  "bind_addr": "172.19.0.2",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "retry_join": [
    "172.19.0.3",
    "172.19.0.4"
  ]
}
```

After configuring the above, you can restart the container and then run the simplified Consul command:

```bash
❯ docker compose up -d
❯ docker exec -it consulserver01 sh
❯ consul agent -config-dir=/etc/consul.d
```

Replicate these configurations for the other servers, rerun the Docker command, and start the remaining servers that are missing.

## Working with Encryption

Throughout this example, every time we use the `consul join XXX.XX.X.X` command, we enter the network, and it's advisable to add more security. We have two types of encryption here: one refers to membership-related encryption, involving joining the Cluster, synchronizing information, and recognizing all members.

The other type of encryption uses [Mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication), where we generate a certificate through a certificate authority, possibly one of the servers. This server generates and distributes client certificates, enabling secure catalog exchange and API access. This ensures secure information exchange.

We made changes to `server.json` for all services:

```json
// For the first server, we removed retry_join
{
  "server": true,
  "bind_addr": "172.19.0.2",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "node_name": "consulserver01"
}

// For the second server, we kept only the address of server01
{
  "server": true,
  "bind_addr": "172.19.0.3",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "retry_join": [
    "172.19.0.2"
  ],
  "node_name": "consulserver02"
}

// We kept server03 the same
{
  "server": true,
  "bind_addr": "172.19.0.4",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "retry_join": [
    "172.19.0.2",
    "172.19.0.3"
  ],
  "node_name": "consulserver03"
}

// We added node_name to all servers
```

Regarding encryption, let's focus on encrypting among members. We need to generate a Consul key and pass it as the "encrypt" parameter with the same key in all `server.json` files. However, for testing purposes, we left the key out of server03.

First, we run the `rm -rf /tmp/*` command on all terminals to prevent errors related to previously created data. Then, we run `consul keygen` and add the generated key to the first two servers:

```json
{
  "server": true,
  "bind_addr": "172.19.0.2",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "node_name": "consulserver01",
  "encrypt": "run the ```consul keygen``` command and add the key here"
}

{
  "server": true,
  "bind_addr": "172.19.0.3",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "retry_join": [
    "172.19.0.2"
  ],
  "node_name": "consulserver02",
  "encrypt": "run the ```consul keygen``` command and add the key here"
}

// For server03, we removed the "encrypt" parameter
{
  "server": true,
  "bind_addr": "172.19.0.4",
  "bootstrap_expect": 3,
  "data_dir": "/tmp",
  "retry_join": [
    "172.19.0.2",
    "172.19.0.3"
  ],
  "node_name": "consulserver03"
}
```

Remember to run `rm -rf /tmp/*` on all terminals to clean up any previous data.

After applying all the changes, we start the first server and, at the end of its terminal, we see that all services have started. We repeat the same steps for server02 and then server03. 

Since, in this example, we deliberately didn't provide the encryption key to server03, the output we see in the terminal is as follows:

```bash
❯ consul agent -config-dir=/etc/consul.d

# expected output
==> Starting Consul agent...
           Version: '1.10.12'
         Node name: 'consulserver03'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: -1, DNS: 8600)
      Cluster Addr: 172.19.0.4 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false, Auto-Encrypt-TLS: false

==> Log data will now stream in as it occurs:
2023-08-19T23:22:49.457Z [INFO]  agent: Joining cluster...: cluster=LAN
2023-08-19T23:22:49.457Z [INFO]  agent: (LAN) joining: lan_addresses=[172.19.0.2, " 172.19.0.3"]
2023-08-19T23:22:49.457Z [INFO]  agent: started state syncer
2023-08-19T23:22:49.457Z [INFO]  agent: Consul agent running!
2023-08-19T23:22:51.460Z [WARN]  agent.server.memberlist.lan: memberlist: Failed to resolve  172.19.0.3: lookup  172.19.0.3: no such host
2023-08-19T23:22:51.460Z [WARN]  agent: (LAN) couldn't join: number_of_nodes=0 error="2 errors occurred:
        * Failed to join 172.19.0.2:8301: Remote state is encrypted and encryption is not configured
        * Failed to resolve  172.19.0.3: lookup  172.19.0.3: no such host
```

We can then see that it cannot access the Consul cluster.

And when we run the ```command consul``` members in the server01 terminal, we notice that only server01 and server02 have been added to the Cluster.

```bash
❯ consul members

# expected output
Node            Address          Status  Type    Build    Protocol  DC   Segment
consulserver01  172.19.0.2:8301  alive   server  1.10.12  2         dc1  <all>
consulserver02  172.19.0.3:8301  alive   server  1.10.12  2         dc1  <all>
```

We can also attempt to listen on a port on the TCP network interface to verify if the data is indeed being encrypted, attempting to observe the information being transmitted.

First, let's install tcpdump on the container.
```bash
❯ apk add tcpdump

# We verified using `ifconfig` that our TCP interface is `eth0`.
❯ ifconfig

# With the command below, we attempt to read the information being transmitted.
❯ tcpdump -i eth0 -an port 8301 -A

# expected output
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23:35:08.963680 IP 172.19.0.3.8301 > 172.19.0.2.8301: UDP, length 116
E.....@.@.Tq........ m m.|X......j!@...  /......s..@e"iu.$..,xy..g7.y!.y......L~x.B.....@...{:..Qh.A..........*....M..<.J..@.C.^ ...C.p....0.yK.
23:35:08.964058 IP 172.19.0.2.8301 > 172.19.0.3.8301: UDP, length 181
E...".@.@........... m m..X..n..?X*.:
.ss.&t...@......v"G..6j.k}.qp:.Sb.\.....5.4../..`gv../.r.       l(...$.....F&.......z..o....=c...........{.Gt.k.k...]6\@/|...v........../...~N.../f...  e.t......BT.Hv...i...A....
23:35:09.292942 IP 172.19.0.2.8301 > 172.19.0.3.8301: UDP, length 116
E...#.@.@..,........ m m.|X..9. .._..ZY0...>".....b.i.+..9.Tb....z....y.....u........M....,._7....M.4[.z..q.zc.....zl........f...L.......T....k.
23:35:09.293476 IP 172.19.0.3.8301 > 172.19.0.2.8301: UDP, length 181
E.....@.@.T......... m m..X...........$%....&..^.gBT4..zJY....`.+-.'.......a.34.....5......x.\..!z..y..,.<H_}(........lM..\C0k.W.!
!e..........zB.H}......t.r?9..E..5z....Z.........m,...].x....x=..&......Y]....
23:35:09.964173 IP 172.19.0.3.8301 > 172.19.0.2.8301: UDP, length 116
E.....@.@.TE........ m m.|X....4....Q.
@zO..B.c....0e.$.........I...6..P..x.6..S%.h...P_......_.?..8.(S.95.w....Q.@..."...*...B.F...5.9........R
23:35:09.965185 IP 172.19.0.2.8301 > 172.19.0.3.8301: UDP, length 181
E...#.@.@........... m m..X......v.J.+..)...t...N.....uq.@...M..].*[...@.%M.......B.).^...iZ>.$2^.R..5.47.........V.j....+=l+5..k...-.i.`.4..>....1.i...&.....{.Y........O....;+.<!.3.....u...z(..n'.akx.Z#.p..D.
23:35:10.843093 IP 172.19.0.2.43660 > 172.19.0.3.8301: Flags [S], seq 1265554281, win 64240, options [mss 1460,sackOK,TS val 2170266511 ecr 0,nop,wscale 7], length 0
E..<..@.@............. mKn.i........XZ.........

```

The security change is in the data exchange between the servers. Previously, anyone who knew the IP address could join the cluster. However, with this new security model, only machines that have the key are allowed to join the network. This is the main security improvement, encrypting all the network traffic exchanged by servers. Without the correct key, it's impossible to access the network.

Please note that this example is aimed at understanding how to run a cluster and isn't necessarily secure for production environments. In a production environment, you would handle the keys and certificates more securely, preferably using a secure key management system.

## Consul User Interface and Production Tips
The Consul User Interface is accessible via a web browser, and to use it, we need to enable it when bringing up the Cluster. There are two ways to do this:

→ Include the ```-ui``` tag when starting the client: ```consul agent -config-dir=/etc/consul.d -ui```

→ In the case of running within Docker, since the UI is designed to run on localhost and attempting to access localhost from outside Docker by sharing a port, we need to make this UI accessible from anywhere. Therefore, we could pass 0.0.0.0 to the client, but we will configure this directly in the configuration file.

Since the UI runs on port 8500, we also had to share this port in the docker-compose.yaml.

We then ran the docker-compose up and the servers again.

![vscode](https://github.com/lilianmartinsfritzen/service-discovery-example/assets/83084256/8ca7ca06-15d5-44cc-bcd0-2693cfc9bb32)


So we have access to the Consul User Interface → http://localhost:8500/ui/dc1/services

![ui-consul](https://github.com/lilianmartinsfritzen/service-discovery-example/assets/83084256/00bc68bf-d5af-403e-bc3e-a74895e95096)

<h2 id="developer">Developer</h2>
  <img src="https://user-images.githubusercontent.com/83084256/180618959-7691ab72-29fd-413f-a489-d3206831231b.jpeg" width="110" height="110" style="border-radius: 65px" /> <br>
  <a href="https://www.linkedin.com/in/lilian-martins-fritzen/" target="blank">
    <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" />
  </a>
