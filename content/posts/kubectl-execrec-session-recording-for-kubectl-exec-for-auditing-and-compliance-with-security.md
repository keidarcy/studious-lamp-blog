+++
title = 'kubectl-execrec: Session Recording for kubectl exec for auditing and compliance with security'
description = 'The plugin you need just record `kubectl exec` sessions with minimal changes to existing workflows. No complex setup, no additional infrastructure, just session recording.'
date = 2025-08-11T18:23:00+09:00
+++

![intro-image-created-by-chatgpt.png](https://miro.medium.com/v2/resize:fit:700/1*xpNw34UIvHmnt-4OLl-G3Q.png)

## Why I Created This Plugin

[https://github.com/keidarc/kubectl-execrec](https://github.com/keidarc/kubectl-execrec)

Kubernetes has built-in audit logs, but they only capture API calls, not the actual session content. When you use `kubectl exec`, you get audit entries showing who accessed which pod, but not what commands were run or what output was produced.

For comprehensive security monitoring, tools like [Falco](https://falco.org/) provide real-time threat detection across containers and Kubernetes. However, these are heavy solutions designed for enterprise security teams.

I wanted something simpler: just record `kubectl exec` sessions with minimal changes to existing workflows. No complex setup, no additional infrastructure, just session recording when you need it for security and compliance.

## Solution: [kubectl-execrec](https://github.com/keidarc/kubectl-execrec)

**execrec** = **exec** + **rec**order

A transparent wrapper around `kubectl exec` that automatically logs all sessions to timestamped files for security auditing and compliance. This is a simple and lightweight solution that can be used by anyone to record their `kubectl exec` sessions and logs are saved to the local filesystem with optional to storage services like S3.

## How It Works

The plugin acts as a transparent wrapper around `kubectl exec`. When you run `kubectl execrec`, it:

1. Creates a log file with session metadata
2. Starts the actual `kubectl exec` command with a pseudo-terminal
3. Captures all input/output and writes to both your terminal and the log file
4. Handles signals like Ctrl+C properly
5. Closes the session and finalizes the log file
6. Uploads the log file to the storage service if configured

The key is that it's completely transparent. You use it exactly like `kubectl exec`, but everything gets recorded automatically.

## Features

### Transparent Wrapper

```bash
# Instead of:
kubectl exec -n default my-pod -it -- bash

# Use:
kubectl execrec -n default my-pod -it -- bash
```

All flags, arguments, and behavior remain identical while providing complete session security logging.

### Log Format

```
[command] kubectl execrec -n default my-pod -it -- bash
[session] start=2025-08-10T14:33:32+09:00 user=username
================================================================================
root@my-pod:/app# ls -la
total 1234
drwxr-xr-x 1 root root 4096 Aug 10 14:33 .
-rw-r--r-- 1 root root  123 Aug 10 14:33 file.txt
root@my-pod:/app# cat file.txt
Hello, World!
root@my-pod:/app# exit
================================================================================
[session] end=2025-08-10T14:35:12+09:00
```

### Storage Service Integration(S3 compatible storage is only supported at the moment)

```bash
export KUBECTL_EXECREC_S3_BUCKET=my-logs-bucket
export KUBECTL_EXECREC_S3_ENDPOINT=http://localhost:9000  # MinIO as an example
kubectl execrec -n default my-pod -it -- bash
```
