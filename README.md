
# k8s-configs-multitenancy
Some k8s YAML files to explore multitenancy through namespaces in k8s

# local set up
These YAML files were tested on a local K8s cluster set up with 4 VMS on Debian 12 using KVM. Cluster was set up with `kubeadm` and Flannel CNI was used to handle networking across the nodes.

# k8s-configs

This whole thing is for **testing Kubernetes multitenancy (MT)** without the use of third-party security tools. Purpose is to see how tenants are isolated, what works, and where stuff breaks.

### access-control/ → Roles, RoleBindings, ServiceAccounts
When we set up a local K8s cluster, **it’s a pain to create users**. Instead of dealing with that, we just create **ServiceAccounts (SAs)** for each namespace (tenant). This lets us **simulate different users with different permissions**.

We apply the **Roles, RoleBindings, and ServiceAccounts**, then test permissions with:
```bash
kubectl auth can-i list pods -n tenant1 --as=system:serviceaccount:tenant2:tenant2-sa
```

```bash
kubectl auth can-i list pods -n tenant2 --as=system:serviceaccount:tenant1:tenant1-sa
```

If we try to access a namespace we **shouldn’t have access to**, we get **"no"** since the RoleBinding we created prevents us from accessing a namespace that the ServiceAccount in question should not.
If we run the same command **as ourselves**, we still have access because we're cluster admins.

**ClusterRoles:** are risky if too broad. They apply **cluster-wide** and should be restricted so we avoid using them unless necessary.

### multitenancy/ → Namespaces, Quotas, Pod Security, Networking
This folder holds **everything that enforces tenant separation**.  
- **Namespaces:** Each tenant gets its own isolated space.  
- **Resource Quotas:** We limit CPU/memory per tenant so no one hogs everything.  
- **Pod Security:** Restrict privileged containers and enforce security policies.  
- **Networking:** Set up policies so tenants **can’t talk to each other** unless allowed.

### autoscaling/ → HPA configs
There's 2 HPAs for the 2 Noisy deployments in order to see more pods get created. We should enforce some resource quotas or limits to avoid the noisy neighbor situation.
For it to work, Metrics Server should be installed in the cluster: kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
You may need to disable TLS certification for it to run properly, depending on what type of K8s you run

### tests/ → Privileged Pod tests, Noisy Neighbor tests, Network Isolation tests
We run **MT-related attack scenarios** here to see what breaks.  
- **Privileged Pod Tests:** Check if we can escalate privileges where we shouldn’t.  
- **Noisy Neighbor Tests:** Simulate one tenant consuming too many resources to see if quotas hold.  
- **Network Isolation Tests:** Verify if tenants can or can’t reach each other.
