# Bootstrap the Talos linux cluster

```
talosctl get disks --insecure --nodes $CONTROL_PLANE_IP
```

Generate the configurations files:

```
talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 --install-disk /dev/$DISK_NAME
```

Apply the configurations files to the controlplane:

```
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file controlplane.yaml
```

Apply the workers configurations:

```
for ip in "${WORKER_IP[@]}"; do
    echo "Applying config to worker node: $ip"
    talosctl apply-config --insecure --nodes "$ip" --file worker.yaml
done
```

Set the endpoint:

```
talosctl --talosconfig=./talosconfig config endpoints $CONTROL_PLANE_IP
```

Bootstrap etcd cluster:

```
talosctl bootstrap --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
```

Merge the new cluster kubeconfig file to you default configuration:

```
talosctl kubeconfig --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
```

Merge the new cluster kubeconfig file on the local machine:

```
talosctl kubeconfig alternative-kubeconfig --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
export KUBECONFIG=./alternative-kubeconfig
```

Check cluster health:

```
talosctl --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig health
```
