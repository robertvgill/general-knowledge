# TCP Server and Client Connection Setup

# Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Setting Up the TCP Server](#setting-up-the-tcp-server)
    - [Choose a Programming Language](#choose-a-programming-language)
    - [Write the Server Code](#write-the-server-code)
    - [Run the Server](#3-run-the-server)
4. [Connecting Clients to the Server](#connecting-clients-to-the-server)
    - [Open Multiple Terminals](#open-multiple-terminals)
    - [Connect Clients to the Server](#connect-clients-to-the-server)
    - [Verify Connections](#verify-connections)
5. [Monitoring Connections](#monitoring-connections)
    - [Run `netstat`](#run-netstat)
    - [Analyzing Output](#analyzing-output)
6. [Conclusion](#conclusion)


## Introduction
This guide demonstrates how to create a TCP server and connect multiple clients to it using `nc` (netcat). It also explains how to monitor the connections to the server and identify the remote clients' IP addresses and ports.

## Prerequisites
- **Python**: You need Python installed on your system to run the server code. You can download and install Python from the [official Python website](https://www.python.org/downloads/).

- **`nc` (Netcat)**: Netcat is a utility for reading from and writing to network connections using TCP or UDP. It should be pre-installed on most Unix-like systems. If not installed, you can typically install it using your package manager. For example, on Debian-based systems, you can install it using `apt`:

```sh
sudo apt-get update
sudo apt-get install netcat
```

## Setting Up the TCP Server

### Choose a Programming Language

You can create a TCP server using any programming language that supports socket programming. In this example, we'll use Python.

### Write the Server Code

Create a file named `tcp_server.py` and add the following code:

```python
import socket
import threading

HOST = '0.0.0.0'  # Listen on all network interfaces
PORT = 12345      # Port to listen on

def handle_client(client_socket, address):
    print(f"Connection from {address} established.")
    while True:
        try:
            data = client_socket.recv(1024)
            if not data:
                break
            print(f"Received from {address}: {data.decode()}")
        except ConnectionResetError:
            print(f"Connection from {address} closed.")
            break
        except Exception as e:
            if e.errno == 104:
                print(f"EOF received from {address}. Connection closed.")
                break
            else:
                print(f"Error receiving data from {address}: {e}")
                break
    client_socket.close()

def main():
    # Create a socket object
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Bind the socket to the host and port
    server_socket.bind((HOST, PORT))

    # Listen for incoming connections
    server_socket.listen(5)
    print(f"Listening on {HOST}:{PORT}...")

    try:
        while True:
            # Accept a connection
            client_socket, address = server_socket.accept()

            # Start a new thread to handle the client
            client_thread = threading.Thread(target=handle_client, args=(client_socket, address))
            client_thread.start()
    except KeyboardInterrupt:
        print("Server interrupted. Closing...")
    finally:
        server_socket.close()

if __name__ == "__main__":
    main()
```

This code creates a TCP server that listens on all available network interfaces (`0.0.0.0`) and on port `12345`.

### Run the Server

Execute the following command in your terminal to run the server:

```bash
python3 tcp_server.py
```

## Connecting Clients to the Server

### Open Multiple Terminals

Open two or three separate terminal windows or tabs.

### Connect Clients to the Server

In each terminal, use `nc` (netcat) to connect to the server:

```bash
nc <localhost> 12345  # if testing locally
nc <server_ip> 12345  # if testing remotely
```

Replace `<server_ip>` with the IP address of the machine running the server.

### Verify Connections

After connecting clients to the server, you should see corresponding messages on the server terminal indicating the connections.

Example:
```
# python3 tcp_server.py
Listening on 0.0.0.0:12345...
Connection from ('192.168.243.206', 60106) established.
Connection from ('127.0.0.1', 33406) established.
```

## Monitoring Connections

To monitor connections to the server and identify the remote clients' IP addresses and ports, you can use utilities like `netstat`.

### Run `netstat`

Execute the following command in your terminal:

```bash
$ netstat -ntp | grep 12345
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 192.168.243.206:12345   192.168.243.206:41228   ESTABLISHED 53140/python3
tcp        0      0 127.0.0.1:49666         127.0.0.1:12345         ESTABLISHED 53859/nc
tcp        0      0 192.168.243.206:41228   192.168.243.206:12345   ESTABLISHED -
tcp        0      0 127.0.0.1:12345         127.0.0.1:49666         ESTABLISHED 53140/python3
```

This command will display all connections on port `12345`, including the IP addresses and ports of the remote clients.

### Analyzing Output

Look for entries with the `ESTABLISHED` status to identify active connections. The IP addresses and ports listed alongside these entries represent the remote clients connected to your server. `Recv-Q` represents the incoming data waiting to be processed by the application, while `Send-Q` represents the outgoing data waiting to be transmitted over the network. 

## Conclusion

You have now set up a TCP server and connected multiple clients to it using `nc`. Additionally, you have learned how to monitor connections to the server and identify the remote clients' IP addresses and ports.