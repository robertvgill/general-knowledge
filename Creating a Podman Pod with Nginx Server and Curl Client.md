# Creating a Podman Pod with Nginx Server and Curl Client

# Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Steps](#steps)
    - [Create the Pod](#create-the-pod)
    - [Run the Nginx Server Container](#run-the-nginx-server-container)
    - [Test Connectivity](#test-connectivity)
    - [Stop and Remove the Pod](#stop-and-remove-the-pod)
4. [Conclusion](#conclusion)


## Introduction
This guide demonstrates how to set up a Podman pod containing Nginx server and Curl client, enabling seamless testing and development environments.

## Prerequisites
- podman

## Steps

### Create the Pod

To create the `nginx-curl` pod, run the following command:
```sh
podman pod create --name nginx-curl
```

### Run the Nginx Server Container

Start the Nginx server container within the pod:
```sh
podman run -d --pod nginx-curl --name nginx-server nginx
```

## Test Connectivity

Test connectivity between Nginx server and Curl client:
```sh
podman run --rm --pod nginx-curl --name curl-client curlimages/curl curl http://localhost
```

Example Output:
```sh
$ podman run --rm --pod nginx-curl --name curl-client curlimages/curl curl http://localhost
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
100   615  100   615    0     0  19919      0 --:--:-- --:--:-- --:--:-- 20500
```

This command runs the curl client container named curl-client within the nginx-curl pod and makes a curl request to http://localhost, interacting with the nginx server container.

### Stop and Remove the Pod

To clean up resources created by this task, run the following commands:
```sh
podman pod stop nginx-curl
podman pod rm nginx-curl
```

The first command stops the pod, and the second command removes it entirely from your system.

## Conclusion
You have successfully set up a Podman pod containing `Nginx` server and `Curl` client. This setup provides a versatile environment for testing and development purposes, allowing you to easily interact with both server and client applications within a containerized environment.