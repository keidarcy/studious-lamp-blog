+++
title = 'Introducing E1s the K9s Inspired Cli for Aws Ecs'
description = 'E1S - Easily Manage AWS ECS Resources in Terminal(~k9s for ECS)'
date = 2024-04-19T09:21:51+09:00
+++

![](https://storage.googleapis.com/zenn-user-upload/84798ff12916-20240531.png)


If you're working with AWS ECS like me, no matter whether with Fargate or EC2, managing resources can sometimes be a challenge to using aws-cli. You might look for something that brings the ease of k9s to k8s, thee1s aims to become the tool you've been waiting for.
If you're working with AWS ECS, managing resources with aws-cli can sometimes be a challenge, whether you're using Fargate or EC2. For those seeking the ease of k9s for Kubernetes, e1s is the tool you've been waiting for.

## What is e1s?

https://github.com/keidarcy/e1s

`e1s` is a command-line interface tool designed to simplify managing AWS ECS resources. It's inspired by the Kubernetes CLI tool k9s and brings a similar level of convenience and efficiency to the world of AWS ECS like this.

![](https://storage.googleapis.com/zenn-user-upload/e04a1aac2fce-20240419.png)

With `e1s`, you can manage your ECS clusters, services, tasks, and containers directly from your terminal. It's the ideal tool for developers who prefer a terminal-based workflow and require a robust solution to streamline their ECS operations.

## Key Features of e1s

 - **Read-Only Mode**: Browse your ECS resources without the fear of making unintended changes.
 - **Auto Refresh**: Keep your view up to date with the latest changes in your ECS environment.
 - **Resource Descriptions**: Get detailed information about your clusters, services, tasks, and more.
 - **CloudWatch Logs Integration**: View and stream logs directly from your terminal.
 - **Exec towards Containers**: Access your containers directly for quick debugging and troubleshooting via ECS Exec API.
 - **Edit Services**: Quickly register a new task definition JSON and update the service with the registered revision and desired count on the fly.
 - **Port Forwarding Sessions**: Easily set up port forwarding to access your services locally in a private subnet.
 - **File Transfers**: Move files to and from your local machine and ECS containers.
 - **Customize color theme**: highly customizable color theme.


## How e1s solve real-world ECS operation challenges

### Exec towards containers

Experience the convenience of immediate access to your ECS containers with `e1s`. The ecs-exec feature allows you to interactively exec to a running container, providing a powerful way to perform real-time debugging and execute commands work like SSH.

![e1s-ssh-demo](https://storage.googleapis.com/zenn-user-upload/8b64c3645673-20240419.gif)

 This is particularly useful for developers and operators who need to quickly diagnose and resolve issues within their ECS environment.

### Remote Host Port Forwarding

Unlock the potential of your ECS environment with remote host port forwarding capability of `e1s`. Create secure connections from your local machine to a remote service via an ECS container, all without exposing your services to the public internet. This feature is invaluable for developers needing to interact with RDS, ElastiCache, APIs, or other services that are securely tucked away in private subnets.

![e1s-remote-host-port-forwarding-session-demo](https://storage.googleapis.com/zenn-user-upload/5f0247819011-20240419.gif)

Leverage the power of `e1s` to bring remote services to your local port, simplifying the complexities of network access and security. It's the bridge that connects your local development environment to the cloud with ease and security.

### File Transfer via S3 Buckets

Simplify your file transfer workflow with `e1s`. Whether you're deploying new application configurations or extracting logs for analysis, e1s makes transferring files between your local system and ECS containers a breeze.

![e1s-file-transfer](https://storage.googleapis.com/zenn-user-upload/db55df6f2b28-20240419.png)

By utilizing S3 buckets, `e1s` ensures that your transfers are not only quick but also secure and reliable.

 ### On-the-Fly Task Definition Registration and Service Updates

Revolutionize your deployment strategy with `e1s`. Quickly register new task definitions from a JSON file and update your ECS services with the latest revision and desired count in real time.

![e1s-register-task-definition-demo](https://storage.googleapis.com/zenn-user-upload/266f3aec4aaf-20240419.gif)

Embrace the agility that e1s offers for your ECS deployments. Also monitor the deployment, container health, and logs, all in your terminal.

### Customize color theme

You can choose various themes with `--theme`, and if you specify colors with `--config-file`, you can select any color you want.

  - command `e1s --theme dracula`
  - screenshot

![theme-dracula](https://storage.googleapis.com/zenn-user-upload/d6c65a3d422f-20240506.png)

  - config file

```yml
colors:
  BgColor: "#272822"
  FgColor: "#f8f8f2"
  BorderColor: "#a1efe4"
  Black: "#272822"
  Red: "#f92672"
  Green: "#a6e22e"
  Yellow: "#f4bf75"
  Blue: "#66d9ef"
  Magenta: "#ae81ff"
  Cyan: "#a1efe4"
  Gray: "#808080"
```

  - screenshot

![theme-hex](https://storage.googleapis.com/zenn-user-upload/ada31dfbbd5a-20240506.png)

  - config file

```yml
colors:
  BgColor: black
  FgColor: cadeblue
  BorderColor: dodgerblue
  Black: black
  Red: orangered
  Green: palegreen
  Yellow: greenyellow
  Blue: darkslateblue
  Magenta: mediumpurple
  Cyan: lightskyblue
  Gray: lightslategray
```

  - screenshot

  ![theme-w3c](https://storage.googleapis.com/zenn-user-upload/7eedd9145db6-20240506.png)


 ## A Step Forward with e1s for ECS management

`e1s` is designed to make ECS management straightforward and efficient, offering a user-friendly alternative to k9s for the ECS community. While `e1s` may not yet offer the same level of power as k9s, it is continuously evolving, and your feedback is crucial for its growth.

I hope `e1s` become an indispensable part of your ECS management toolkit, helping to simplify your operations and save you valuable time as it helps me!

Please check more details at https://github.com/keidarcy/e1s.
