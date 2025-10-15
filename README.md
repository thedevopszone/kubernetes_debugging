# Kubernetes Debugging

The industry trend is moving away from SSH in containers toward ephemeral debugging containers and better observability tools. 
This aligns with immutable infrastructure principles and improves security posture.


# Practical kubectl debug Examples

## 1. Debug a Running Pod (Ephemeral Container)

Most common use case - attach a debug container to an existing pod:

```bash
#Add a debug container with common tools to a running pod
kubectl debug -it my-pod --image=busybox --target=my-container

# Use a more feature-rich debug image
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-container

# Debug with Ubuntu/Debian tools
kubectl debug -it my-pod --image=ubuntu --target=my-container
```

The --target flag shares the process namespace, so you can see processes from the target container.


## 2. Debug a Crashed/CrashLooping Pod

Create a copy of the pod with a different command:

```bash
# Copy the pod but override the command to keep it running
kubectl debug my-crashing-pod -it --copy-to=my-crashing-pod-debug --container=app -- sh

# Or just change the command but keep everything else
kubectl debug my-pod -it --copy-to=debug-copy --container=mycontainer -- /bin/bash
```

This is invaluable when your container crashes immediately and you need to investigate.

##3. Debug a Node

Access a node with a privileged container:

```bash
# Debug a specific node
kubectl debug node/my-node -it --image=ubuntu

# The node's filesystem is mounted at /host
# Once inside, you can:
# - Check node logs: chroot /host
# - Inspect networking: ip addr, ss -tulpn
# - Check disk: df -h /host
```

## 4. Debug with Specific Namespace/Process Sharing

```bash
# Share ALL namespaces with target container (network, PID, IPC)
kubectl debug -it my-pod --image=nicolaka/netshoot \
  --target=my-container \
  --share-processes

# Profile with access to host network
kubectl debug -it my-pod --image=nicolaka/netshoot \
  --target=my-container \
  --profile=netadmin
```

## 5. Useful Debug Images

Different images for different scenarios:

```bash
# Network debugging (tcpdump, nslookup, curl, etc.)
kubectl debug -it my-pod --image=nicolaka/netshoot

# Lightweight shell
kubectl debug -it my-pod --image=busybox

# Full Linux utilities
kubectl debug -it my-pod --image=ubuntu

# Alpine with common tools
kubectl debug -it my-pod --image=alpine
```

## 6. Common Debugging Workflows

Network issues:

```bash
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-container

# Inside the debug container:
# - Test connectivity: curl http://service:port
# - DNS lookup: nslookup service-name
# - Capture packets: tcpdump -i any port 8080
# - Check routes: ip route
```

Application crashes:
```bash

kubectl debug my-pod --copy-to=debug --container=app -- tail -f /dev/null

# Then exec into it
kubectl exec -it debug -- bash

# Now you can:
# - Inspect filesystem
# - Check configurations
# - Manually run the application with different parameters
```

File system inspection:
```bash
kubectl debug -it my-pod --image=busybox --target=my-container

# Inside:
ls -la /var/log
cat /etc/config/app.conf
df -h
```

## 7. Profile-based Debugging (Kubernetes 1.23+)

```bash
# Restricted profile (default)
kubectl debug -it my-pod --image=busybox --profile=restricted

# General debugging (some privileges)
kubectl debug -it my-pod --image=busybox --profile=general

# Baseline profile
kubectl debug -it my-pod --image=busybox --profile=baseline

# Network admin capabilities
kubectl debug -it my-pod --image=nicolaka/netshoot --profile=netadmin

# Full sysadmin access (use carefully)
kubectl debug -it my-pod --image=ubuntu --profile=sysadmin
```

## 8. Clean Up Debug Resources

```bash
# Debug containers are ephemeral, but copied pods need cleanup
kubectl delete pod debug-copy

# List all debug pods
kubectl get pods | grep debug
```

## Pro Tips

Create an alias for common debug commands:
```bash
alias kdebug='kubectl debug -it --image=nicolaka/netshoot'
```

Check if ephemeral containers are enabled (required for kubectl debug):
```bash
kubectl explain pod.spec.ephemeralContainers
```

For production, consider using read-only debug containers when possible

Always specify a namespace in production:

```bash
kubectl debug -it my-pod -n production --image=busybox
```
