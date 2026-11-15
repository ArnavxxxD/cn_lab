# Computer Networks Lab: Execution Steps

Step-by-step, copy-paste execution guide for Experiments 1–10. Every command sits in a code block, so use GitHub's copy button on the top-right of each block.

## Setup

**Ubuntu (Exp 2, 3, 4, 5, 8, 9, 10), one-time:**

```bash
sudo apt update && sudo apt install gcc gedit -y
mkdir cn_lab && cd cn_lab
```

**Pattern for every C program:**

```bash
gedit file.c &
gcc file.c -o name
./name
```

> In every C step: paste the code in gedit, save with `Ctrl+S`, close gedit. Open a second terminal with `Ctrl+Shift+T` where two terminals are needed.

## Contents

| Exp | Title | Platform |
|:-:|---|---|
| 1A | [Network topologies, media, devices and RJ-45](#experiment-1a-network-topologies-media-devices-and-rj-45) | Study |
| 1B | [File sharing within an organisation using IP address](#experiment-1b-file-sharing-within-an-organisation-using-ip-address) | Windows |
| 2 | [TCP Server](#experiment-2-tcp-server) | Ubuntu, Terminal 1 |
| 3 | [TCP Client](#experiment-3-tcp-client) | Ubuntu, Terminal 2 |
| 4 | [UDP Server](#experiment-4-udp-server) | Ubuntu, Terminal 1 |
| 5 | [UDP Client](#experiment-5-udp-client) | Ubuntu, Terminal 2 |
| 6 | [DNS analysis using nslookup and Wireshark](#experiment-6-dns-analysis-using-nslookup-and-wireshark) | Windows |
| 7 | [DNS and HTTP analysis using Wireshark](#experiment-7-dns-and-http-analysis-using-wireshark) | Windows |
| 8 | [Sliding Window Protocol (Go-Back-N)](#experiment-8-sliding-window-protocol-go-back-n) | Ubuntu |
| 9A | [Leaky Bucket](#experiment-9a-leaky-bucket) | Ubuntu |
| 9B | [Token Bucket](#experiment-9b-token-bucket) | Ubuntu |
| 10 | [Distance Vector Routing](#experiment-10-distance-vector-routing) | Ubuntu |

---

## Experiment 1A: Network topologies, media, devices and RJ-45

*Platform: Study*

> Theory-only experiment: there are no commands to run. Write topologies, transmission media, networking devices and RJ-45 colour codes in your record.

**RJ-45 colour coding**

| Pin | T568A | T568B |
|:-:|---|---|
| 1 | White/Green | White/Orange |
| 2 | Green | Orange |
| 3 | White/Orange | White/Green |
| 4 | Blue | Blue |
| 5 | White/Blue | White/Blue |
| 6 | Orange | Green |
| 7 | White/Brown | White/Brown |
| 8 | Brown | Brown |

---

## Experiment 1B: File sharing within an organisation using IP address

*Platform: Windows*

**Step 1: Open Command Prompt (`Win+R`, type `cmd`, press Enter) and find this PC's IP**

```bash
ipconfig
```

**Step 2: Check connectivity with the other PC (replace with its IPv4 address)**

```bash
ping 10.17.60.44
```

**Step 3: Share a folder on the host PC (GUI)**

1. Create a folder, e.g. `Shared_Folder`
2. Right-click the folder → **Properties** → **Sharing** tab → **Advanced Sharing**
3. Tick **Share this folder** → **Permissions** → allow Full Control / Read → **OK** → **OK**


**Step 4: Or share from an Administrator Command Prompt instead of the GUI**

```cmd
mkdir C:\Shared_Folder
net share Shared_Folder=C:\Shared_Folder /GRANT:Everyone,FULL
```

**Step 5: On the client PC press `Win+R` and paste (replace with the host IP)**

```text
\\10.17.60.44\Shared_Folder
```

**Step 6: Or map it as a drive from Command Prompt on the client PC**

```cmd
net use Z: \\10.17.60.44\Shared_Folder
```

**Expected output**

```text
Reply from 10.17.60.44: bytes=32 time<1ms TTL=128
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

The shared folder opens and files can be read/copied.
```

---

## Experiment 2: TCP Server

*Platform: Ubuntu, Terminal 1*

**Step 1: Create the file**

```bash
gedit server_1290.c &
```

Paste this into `server_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>

#define PORT 10000+298
#define BUF_SIZE 1024

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    char buffer[BUF_SIZE];
    int addrlen = sizeof(address);
    
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    
    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, 3);
    printf("TCP Echo Server listening on port %d...\n", PORT);
    
    new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen);
    while (1) {
        memset(buffer, 0, BUF_SIZE);
        int bytes = read(new_socket, buffer, BUF_SIZE);
        if (bytes <= 0) break;
        printf("Received: %s", buffer);
        send(new_socket, buffer, bytes, 0); // echo back
    }
    
    return 0;
}
```

**Step 2: Compile**

```bash
gcc server_1290.c -o server
```

**Step 3: Run (keep this terminal open and go to Experiment 3 in Terminal 2)**

```bash
./server
```

**Expected output**

```text
TCP Echo Server listening on port 10298...
Received: Hello Server
Received: Second message
```

> The `Received` lines appear after the client in Experiment 3 sends messages. The server stops by itself when the client disconnects.

---

## Experiment 3: TCP Client

*Platform: Ubuntu, Terminal 2*

> The server from Experiment 2 must already be running in Terminal 1.

**Step 1: Open a new terminal (`Ctrl+Shift+T`) and go to the lab folder**

```bash
cd ~/cn_lab
```

**Step 2: Create the file**

```bash
gedit client_1290.c &
```

Paste this into `client_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 10298
#define BUF_SIZE 1024

int main(int argc, char *argv[]) {
    int sock = 0;
    struct sockaddr_in serv_addr;
    char buffer[BUF_SIZE] = {0};
    char message[BUF_SIZE];

    if (argc != 2) {
        printf("Usage: %s <Server IP>\n", argv[0]);
        return -1;
    }

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        printf("\n Socket creation error \n");
        return -1;
    }

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);

    if (inet_pton(AF_INET, argv[1], &serv_addr.sin_addr) <= 0) {
        printf("\nInvalid address/ Address not supported \n");
        return -1;
    }

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        printf("\nConnection Failed \n");
        return -1;
    }
    printf("Connected to server. Enter message: ");

    while (fgets(message, BUF_SIZE, stdin) != NULL) {
        send(sock, message, strlen(message), 0);
        int valread = read(sock, buffer, BUF_SIZE);
        printf("Echo from server: %s", buffer);
        memset(buffer, 0, BUF_SIZE);
        printf("Enter message: ");
    }

    close(sock);
    return 0;
}
```

**Step 3: Compile**

```bash
gcc client_1290.c -o client
```

**Step 4: Run (`127.0.0.1` = same computer)**

```bash
./client 127.0.0.1
```

**Step 5: Type these messages, pressing Enter after each, then press `Ctrl+D` to quit**

```text
Hello Server
Second message
```

**Expected output**

```text
Connected to server. Enter message: Hello Server
Echo from server: Hello Server
Enter message: Second message
Echo from server: Second message
Enter message:
```

---

## Experiment 4: UDP Server

*Platform: Ubuntu, Terminal 1*

**Step 1: Create the file**

```bash
gedit udp_server_1290.c &
```

Paste this into `udp_server_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 10290
#define BUF_SIZE 1024

int main() {
    int server_fd;
    struct sockaddr_in servaddr, cliaddr;
    char buffer[BUF_SIZE];
    socklen_t len;
    int n;

    // 1. Socket Creation
    if ((server_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("socket creation failed");
        exit(EXIT_FAILURE);
    }

    memset(&servaddr, 0, sizeof(servaddr));
    memset(&cliaddr, 0, sizeof(cliaddr));

    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = INADDR_ANY;
    servaddr.sin_port = htons(PORT);

    // 2. Binding
    if (bind(server_fd, (const struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    printf("UDP Echo Server listening on port %d...\n", PORT);

    len = sizeof(cliaddr);

    while (1) {
        memset(buffer, 0, BUF_SIZE);
        // 3. Receiving Data
        n = recvfrom(server_fd, (char *)buffer, BUF_SIZE, MSG_WAITALL, (struct sockaddr *)&cliaddr, &len);
        buffer[n] = '\0';
        printf("Received from client: %s", buffer);

        // 4. Sending Data (Echo back)
        sendto(server_fd, (const char *)buffer, strlen(buffer), MSG_CONFIRM, (const struct sockaddr *)&cliaddr, len);
        printf("Sent back to client: %s", buffer);
    }

    // 5. Closing Socket
    close(server_fd);
    return 0;
}
```

**Step 2: Compile**

```bash
gcc udp_server_1290.c -o udp_server
```

**Step 3: Run (keep it open and go to Experiment 5 in Terminal 2)**

```bash
./udp_server
```

**Expected output**

```text
UDP Echo Server listening on port 10290...
Received from client: Hello UDP Server
Sent back to client: Hello UDP Server
```

> Stop the server with `Ctrl+C` after the client has finished.

---

## Experiment 5: UDP Client

*Platform: Ubuntu, Terminal 2*

> The server from Experiment 4 must already be running in Terminal 1.

**Step 1: Open a new terminal (`Ctrl+Shift+T`) and go to the lab folder**

```bash
cd ~/cn_lab
```

**Step 2: Create the file**

```bash
gedit udp_client_1290.c &
```

Paste this into `udp_client_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 10290
#define BUF_SIZE 1024

int main(int argc, char *argv[]) {
    int sock;
    struct sockaddr_in servaddr;
    char buffer[BUF_SIZE];
    char message[BUF_SIZE];
    socklen_t len;
    int n;

    if (argc != 2) {
        printf("Usage: %s <Server IP>\n", argv[0]);
        return -1;
    }

    // 1. Socket Creation
    if ((sock = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("socket creation failed");
        exit(EXIT_FAILURE);
    }

    memset(&servaddr, 0, sizeof(servaddr));

    // 2. Server Address Configuration
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(PORT);
    if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0) {
        printf("\nInvalid address/ Address not supported \n");
        return -1;
    }

    printf("Enter message to send: ");
    while (fgets(message, BUF_SIZE, stdin) != NULL) {
        // 3. Data Transmission
        sendto(sock, (const char *)message, strlen(message), MSG_CONFIRM, (const struct sockaddr *)&servaddr, sizeof(servaddr));
        
        len = sizeof(servaddr);
        memset(buffer, 0, BUF_SIZE);
        
        // 4. Receiving Response
        n = recvfrom(sock, (char *)buffer, BUF_SIZE, MSG_WAITALL, (struct sockaddr *)&servaddr, &len);
        buffer[n] = '\0';
        printf("Echo from server: %s", buffer);
        
        printf("Enter message to send: ");
    }

    // 5. Socket Closure
    close(sock);
    return 0;
}
```

**Step 3: Compile**

```bash
gcc udp_client_1290.c -o udp_client
```

**Step 4: Run**

```bash
./udp_client 127.0.0.1
```

**Step 5: Type this message and press Enter, then press `Ctrl+D` to quit**

```text
Hello UDP Server
```

**Expected output**

```text
Enter message to send: Hello UDP Server
Echo from server: Hello UDP Server
Enter message to send:
```

---

## Experiment 6: DNS analysis using nslookup and Wireshark

*Platform: Windows*

**Step 1: Open Command Prompt (`Win+R`, type `cmd`, Enter) and clear the DNS cache so queries go on the network**

```cmd
ipconfig /flushdns
```

**Step 2: Start Wireshark**

1. Open Wireshark and double-click the active interface (Wi-Fi / Ethernet) to start the capture
2. Type the display filter below in the filter bar and press Enter


**Step 3: Wireshark display filter**

```text
dns
```

**Step 4: In Command Prompt: forward lookup (A record)**

```cmd
nslookup www.example.com
```

**Step 5: Reverse lookup (PTR record)**

```cmd
nslookup 104.20.23.154
```

**Step 6: Mail exchanger lookup (MX record)**

```cmd
nslookup -query=MX example.com
```

**Step 7: Stop and analyse**

1. Click the red **Stop** button in Wireshark
2. Click a DNS packet and expand **Domain Name System** to see Transaction ID, Flags, Queries and Answers
3. Note: protocol UDP, port 53 (query and response)
4. Take a screenshot for the record


**Expected output**

```text
Server:  SIT-CDC.SIT-CDC.SYMBINGP.SOC
Address:  10.17.28.42

Non-authoritative answer:
Name:    www.example.com
Addresses:  2606:4700:10::6814:179a
          2606:4700:10::ac42:93f3
          104.20.23.154
          172.66.147.243
```

> IP addresses and the server name will differ on your system.

---

## Experiment 7: DNS and HTTP analysis using Wireshark

*Platform: Windows*

**Step 1: Open Command Prompt (`Win+R`, type `cmd`, Enter) and clear the DNS cache**

```cmd
ipconfig /flushdns
```

**Step 2: Start Wireshark: double-click the active interface (Wi-Fi / Ethernet) to start the capture**

**Step 3: In Command Prompt: DNS lookup**

```cmd
nslookup www.google.com
```

**Step 4: Generate HTTP traffic (plain HTTP on port 80)**

```cmd
curl -v http://example.com
```

**Step 5: Wireshark filter 1: DNS packets**

```text
dns
```

**Step 6: Wireshark filter 2: HTTP packets**

```text
http
```

**Step 7: Stop and analyse**

1. Click the red **Stop** button in Wireshark
2. DNS packet: UDP port 53 → Transaction ID, Flags, Queries, Answers
3. HTTP packet: TCP port 80 → `GET` request, Host, User-Agent, then the `200 OK` response
4. Take screenshots for the record


**Expected output**

```text
Server:  SIT-CDC.SIT-CDC.SYMBINGP.SOC
Address:  10.17.28.42

Non-authoritative answer:
Name:    www.google.com
Addresses:  2001:4860:482c:7700::
          142.251.153.119
          142.251.150.119
```

> IP addresses will differ on your system.

---

## Experiment 8: Sliding Window Protocol (Go-Back-N)

*Platform: Ubuntu*

**Step 1: Create the file**

```bash
gedit sliding_window_1290.c &
```

Paste this into `sliding_window_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define MAX_FRAMES 50

void sender(int frames[], int size, int window_size);
void receiver(int frames[], int start, int window_size);

int main() {
    int n, window_size;
    int frames[MAX_FRAMES];

    printf("Sliding Window Protocol Simulation (Go-Back-N)\n");
    printf("Enter number of frames to send (max %d): ", MAX_FRAMES);
    scanf("%d", &n);

    if (n > MAX_FRAMES || n <= 0) {
        printf("Invalid number of frames \n");
        return 0;
    }

    printf("Enter window size: ");
    scanf("%d", &window_size);

    for (int i = 0; i < n; i++) {
        frames[i] = i;
    }

    int i = 0;
    char choice;

    while (i < n) {
        printf("\nSender window: ");
        for (int j = i; j < i + window_size && j < n; j++) {
            printf("[%d] ", frames[j]);
        }
        printf("\n");

        printf("Sending frames...\n");
        int sent_count = 0;
        for (int j = i; j < i + window_size && j < n; j++) {
            printf("Sent frame: %d\n", frames[j]);
            sent_count++;
        }

        printf("\nSimulating receiver...\n");
        int ack = i;
        for (int j = i; j < i + window_size && j < n; j++) {
            printf("Received frame: %d, sending ACK: %d\n", frames[j], frames[j] + 1);
            ack = j + 1;
        }

        printf("\nDid all frames receive correctly? (y/n): ");
        scanf(" %c", &choice);

        if (choice == 'y' || choice == 'Y') {
            i += sent_count;
        } else {
            printf("Error detected! Resending window (Go-Back-N)...\n");
        }
    }

    printf("All frames sent successfully!\n");
    return 0;
}
```

**Step 2: Compile**

```bash
gcc sliding_window_1290.c -o sliding_window
```

**Step 3: Run**

```bash
./sliding_window
```

**Step 4: Type these inputs, one per line, when asked**

```text
6
3
y
y
```

**Step 5: Or run it in one command**

```bash
printf '6\n3\ny\ny\n' | ./sliding_window
```

**Expected output**

```text
Sliding Window Protocol Simulation (Go-Back-N)
Enter number of frames to send (max 50): 6
Enter window size: 3

Sender window: [0] [1] [2]
Sending frames...
Sent frame: 0
Sent frame: 1
Sent frame: 2

Simulating receiver...
Received frame: 0, sending ACK: 1
Received frame: 1, sending ACK: 2
Received frame: 2, sending ACK: 3

Did all frames receive correctly? (y/n): y

Sender window: [3] [4] [5]
Sending frames...
Sent frame: 3
Sent frame: 4
Sent frame: 5

Simulating receiver...
Received frame: 3, sending ACK: 4
Received frame: 4, sending ACK: 5
Received frame: 5, sending ACK: 6

Did all frames receive correctly? (y/n): y
All frames sent successfully!
```

---

## Experiment 9A: Leaky Bucket

*Platform: Ubuntu*

**Step 1: Create the file**

```bash
gedit leaky_bucket_1290.c &
```

Paste this into `leaky_bucket_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>

int main() {
    int bucket_size, output_rate, n, packet_size;
    int current_bucket = 0;

    printf("Enter bucket size: ");
    scanf("%d", &bucket_size);

    printf("Enter output rate (leak rate): ");
    scanf("%d", &output_rate);

    printf("Enter number of packets to incoming queue: ");
    scanf("%d", &n);

    printf("\n--- Simulation ---\n");
    for (int i = 1; i <= n; i++) {
        printf("Enter packet size for packet %d: ", i);
        scanf("%d", &packet_size);

        if (current_bucket + packet_size <= bucket_size) {
            current_bucket += packet_size;
            printf("Incoming packet of size %d. Bucket status: %d/%d\n", packet_size, current_bucket, bucket_size);
        } else {
            printf("Incoming packet of size %d. Bucket overflow! Dropped packet of size %d.\n", packet_size, packet_size);
        }
    }

    printf("\nLeaking packets...\n");
    while (current_bucket > 0) {
        if (current_bucket >= output_rate) {
            current_bucket -= output_rate;
        } else {
            current_bucket = 0;
        }
        printf("Leaked %d packets. Remaining in bucket: %d\n", output_rate, current_bucket);
    }

    return 0;
}
```

**Step 2: Compile**

```bash
gcc leaky_bucket_1290.c -o leaky_bucket
```

**Step 3: Run**

```bash
./leaky_bucket
```

**Step 4: Type these inputs, one per line, when asked**

```text
10
3
4
4
5
2
3
```

**Step 5: Or run it in one command**

```bash
printf '10\n3\n4\n4\n5\n2\n3\n' | ./leaky_bucket
```

**Expected output**

```text
Enter bucket size: 10
Enter output rate (leak rate): 3
Enter number of packets to incoming queue: 4

--- Simulation ---
Enter packet size for packet 1: 4
Incoming packet of size 4. Bucket status: 4/10
Enter packet size for packet 2: 5
Incoming packet of size 5. Bucket status: 9/10
Enter packet size for packet 3: 2
Incoming packet of size 2. Bucket overflow! Dropped packet of size 2.
Enter packet size for packet 4: 3
Incoming packet of size 3. Bucket overflow! Dropped packet of size 3.

Leaking packets...
Leaked 3 packets. Remaining in bucket: 6
Leaked 3 packets. Remaining in bucket: 3
Leaked 3 packets. Remaining in bucket: 0
```

---

## Experiment 9B: Token Bucket

*Platform: Ubuntu*

**Step 1: Create the file**

```bash
gedit token_bucket_1290.c &
```

Paste this into `token_bucket_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>

int main() {
    int max_capacity, tokens, token_rate, n, packet_tokens;

    printf("Enter maximum capacity of token bucket: ");
    scanf("%d", &max_capacity);

    printf("Enter initial tokens in bucket: ");
    scanf("%d", &tokens);

    printf("Enter token generation rate per interval: ");
    scanf("%d", &token_rate);

    printf("Enter number of incoming packets: ");
    scanf("%d", &n);

    printf("\n");
    for (int i = 1; i <= n; i++) {
        printf("Packet %d requires tokens: ", i);
        scanf("%d", &packet_tokens);

        if (tokens >= packet_tokens) {
            tokens -= packet_tokens;
            printf("Tokens available: %d + generated (%d). Packet transmitted successfully! Remaining tokens: %d\n\n", 
                   tokens + packet_tokens, 0, tokens);
        } else {
            printf("Not enough tokens! Adding generated tokens...\n");
            tokens += token_rate;
            if (tokens > max_capacity) tokens = max_capacity;

            if (tokens >= packet_tokens) {
                tokens -= packet_tokens;
                printf("Current tokens after refill: %d. Packet transmitted successfully! Remaining tokens: %d\n\n", 
                       tokens + packet_tokens, tokens);
            } else {
                printf("Current tokens: %d. Still insufficient tokens. Packet delayed/dropped.\n\n", tokens);
            }
        }
    }

    return 0;
}
```

**Step 2: Compile**

```bash
gcc token_bucket_1290.c -o token_bucket
```

**Step 3: Run**

```bash
./token_bucket
```

**Step 4: Type these inputs, one per line, when asked**

```text
10
5
2
3
4
6
2
```

**Step 5: Or run it in one command**

```bash
printf '10\n5\n2\n3\n4\n6\n2\n' | ./token_bucket
```

**Expected output**

```text
Enter maximum capacity of token bucket: 10
Enter initial tokens in bucket: 5
Enter token generation rate per interval: 2
Enter number of incoming packets: 3

Packet 1 requires tokens: 4
Tokens available: 5 + generated (0). Packet transmitted successfully! Remaining tokens: 1

Packet 2 requires tokens: 6
Not enough tokens! Adding generated tokens...
Current tokens: 3. Still insufficient tokens. Packet delayed/dropped.

Packet 3 requires tokens: 2
Tokens available: 3 + generated (0). Packet transmitted successfully! Remaining tokens: 1
```

---

## Experiment 10: Distance Vector Routing

*Platform: Ubuntu*

**Step 1: Create the file**

```bash
gedit distance_vector_1290.c &
```

Paste this into `distance_vector_1290.c`, save (`Ctrl+S`) and close gedit:

```c
#include <stdio.h>

struct node {
    unsigned dist[20];
    unsigned from[20];
} rt[10];

int main() {
    int costmat[20][20];
    int nodes, i, j, k, count = 0;

    printf("Enter the number of routers: ");
    scanf("%d", &nodes);

    printf("Enter the cost matrix (use 999 for infinity/no direct link):\n");
    for (i = 0; i < nodes; i++) {
        for (j = 0; j < nodes; j++) {
            scanf("%d", &costmat[i][j]);
            rt[i].dist[j] = costmat[i][j];
            rt[i].from[j] = j;
        }
    }

    do {
        count = 0;
        for (i = 0; i < nodes; i++) {
            for (j = 0; j < nodes; j++) {
                for (k = 0; k < nodes; k++) {
                    if (rt[i].dist[j] > costmat[i][k] + rt[k].dist[j]) {
                        rt[i].dist[j] = rt[i].dist[k] + rt[k].dist[j];
                        rt[i].from[j] = k;
                        count++;
                    }
                }
            }
        }
    } while (count != 0);

    for (i = 0; i < nodes; i++) {
        printf("\n\nRouter info for router: %d\n", i + 1);
        printf("Dest Node\tNext Hop\tDistance\n");
        for (j = 0; j < nodes; j++) {
            printf("%d\t\t%d\t\t%d\n", j + 1, rt[i].from[j] + 1, rt[i].dist[j]);
        }
    }

    return 0;
}
```

**Step 2: Compile**

```bash
gcc distance_vector_1290.c -o distance_vector
```

**Step 3: Run**

```bash
./distance_vector
```

**Step 4: Type these inputs, one per line, when asked**

```text
3
0 2 5
2 0 1
5 1 0
```

**Step 5: Or run it in one command**

```bash
printf '3\n0 2 5\n2 0 1\n5 1 0\n' | ./distance_vector
```

**Expected output**

```text
Enter the number of routers: 3
Enter the cost matrix (use 999 for infinity/no direct link):
0 2 5
2 0 1
5 1 0

Router info for router: 1
Dest Node	Next Hop	Distance
1		1		0
2		2		2
3		2		3

Router info for router: 2
Dest Node	Next Hop	Distance
1		1		2
2		2		0
3		3		1

Router info for router: 3
Dest Node	Next Hop	Distance
1		2		3
2		2		1
3		3		0
```

---

<sub>All C programs were compiled with `gcc` and tested on Ubuntu. Wireshark/Windows steps use standard commands.</sub>
