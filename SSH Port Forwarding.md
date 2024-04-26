# SSH Port Forwarding

# Table of Contents

1. [Introduction](#introduction)
2. [Forward Tunnel (-L option)](#forward-tunnel--l-option)
    - [Listen for Connections](#listen-for-connections)
    - [Forward Local Port to Remote Service](#forward-local-port-to-remote-service)
    - [Test and Send Data](#test-and-send-data)
3. [Reverse Tunnel (-R option)](#reverse-tunnel--r-option)
    - [Listen for Connections](#listen-for-connections-1)
    - [Forward Remote Port to Local Service](#forward-remote-port-to-local-service)
    - [Test and Send Data](#test-and-send-data-1)
4. [Bidirectional SSH Tunnel for Proxying Traffic](#bidirectional-ssh-tunnel-for-proxying-traffic)
4. [Explanation of SSH Command Parameters](#explanation-of-ssh-command-parameters)
5. [Conclusion](#conclusion)


## Introduction
This step explains how to use SSH port forwarding, also known as SSH tunneling, which creates encrypted tunnels between local and remote ports to secure connections or bypass network restrictions.

SSH port forwarding allows you to establish secure communication between a local port on your machine and a remote port on another machine. This technique is commonly used to secure connections to services that are otherwise insecure or to circumvent network restrictions.

In this guide, you'll learn how to establish SSH port forwarding between two machines, enabling secure communication and access to services running on both machines.

## Forward Tunnel (-L option)

### Listen for Connections

From your *virtual machine*, open a new terminal and run the following command:
```sh
nc -l 8888
```
This command listens for connections on port `8888` on your virtual machine.

### Forward Local Port to Remote Service

From your *local*, open a new terminal and run the following command:
```sh
ssh -L [local_port]:[remote_destination]:[remote_port] [username]@[ssh_server] -p [ssh_port] -N
```
Example:
```sh
ssh -L 1234:127.0.0.1:8888 rgill@localhost -p 2222 -N
```
This command forwards local port `1234` to the remote service on port `8888` on the virtual machine.

### Test and Send Data

From your *local*, open a new terminal and run the following command:
```sh
nc localhost 1234
Hello world, QEMU OL9!
```
In this scenario, you're forwarding connections from a local port `1234` and sends the message to a remote host. Even though you're initiating the SSH connection from your local machine, you're connecting to the SSH port `2222` on the remote host. The `-p` option specifies the port number of the SSH server on the remote host that you want to connect to.

## Reverse Tunnel (-R option)

### Listen for Connections

From your *local*, open a new terminal and run the following command:
```sh
nc -l 8000
```
This command listens for connections on port `8000` on your virtual machine.

### Forward Remote Port to Local Service

From your *local*, open a new terminal and run the following command:
```sh
ssh -R [tunnel_port]:[local_host]:[local_port] [username]@[remote_host] -p [remote_port] -N
```
Example:
```sh
ssh -R 4321:localhost:8000 rgill@127.0.0.1 -p 2222 -N
```
This command forwards local port `4321` to the remote service on port `8000` on the virtual machine.

### Test and Send Data

From your *virtual machine*, open a new terminal and run the following command:
```sh
nc localhost 4321
Hello world, WSL Ubuntu!
```
In this scenario, you're setting up a reverse tunnel, where connections made to a port `4321` on the remote host are forwarded to a port `8000` on your local machine. Again, the `-p` option specifies the port number `2222` of the SSH server on the remote host that you want to connect to, while the port `22` on your local machine that receives the forwarded connections is determined by the parameters you set up in the tunnel.

## Bidirectional SSH Tunnel for Proxying Traffic

This guide will walk you through setting up a bidirectional SSH tunnel for proxying traffic from your local machine. This allows you to securely tunnel traffic between your local machine and a remote server, forwarding traffic in both directions.

### Listen for Connections

From your *remote*, open a new terminal and run the following command:
```sh
nc -l 8888
```
This command listens for connections on port `8888` on your virtual machine.

### Bridging Local and Remote Networks

From your *local*, open a new terminal and run the following command:
```sh
ssh -L [local_port]:[remote_host]:[remote_port] -R [remote_port]:<[ocal_host]:[local_port] [username]@[remote_host] -p [ssh_port]
```

Example:
```sh
ssh -L 1234:192.168.200.95:8888 -R 4321:localhost:8000 rgill@192.168.200.95
```
### Test and Send Data

From your *local*, open a new terminal and run the following command:
```sh
nc localhost 1234
Hello world, QEMU OL9!
```
Once the command executes successfully, the SSH tunnel will be established bidirectionally between your local machine and the remote host, proxying traffic as specified.

This sets up a bidirectional SSH tunnel. It forwards traffic from port `1234` on your local machine to port `8888` on the remote host `192.168.200.95`. Additionally, it forwards traffic from port `4321` on the remote host back to port `8000` on your local machine. The SSH connection is established with a username to the remote host `192.168.200.95`.

## Explanation of SSH Command Parameters

`-L [local_port]:[remote_host]:[remote_port]`
This parameter sets up a local port forwarding. It creates a tunnel between a port on your local machine `local_port` and a port on a remote host `remote_port` via the SSH server `remote_host`.

`-R [remote_port]:[local_host]:[local_port]`
This parameter sets up a remote port forwarding. It creates a tunnel between a port on a remote host `remote_port` and a port on your local machine `local_port` via the SSH server, with traffic forwarded to `local_host`.

`[username]@[remote_host]`
Specifies the username and hostname (or IP address) of the remote SSH server to connect to.

`-p [remote_ssh_port]`
Specifies the port number on which the SSH server on the remote host is listening. This allows you to connect to the remote host using a non-default SSH port.

## Conclusion
SSH port forwarding provides a secure and versatile solution for accessing resources on remote networks and exposing local services to remote hosts. By leveraging local and remote port forwarding.

By following the steps outlined in this guide, you can set up SSH port forwarding between two machines, enabling secure access to services running on both machines. This technique is invaluable for maintaining security and flexibility in network configurations.