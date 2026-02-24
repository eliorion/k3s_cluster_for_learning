# Bootstrap the cluster

## **00-Choose your hardware**

The project use one `K3S` node. You can add to any type of node (VM, container, Raspberry Pi, PC, ...) as long as you can install and have access to a K3S cluster. This bootstrap will be execut on a VM with Debian 13 install on a Proxmox server.

## **01-DevPod initialisation**

To setup the devcontainer with DevPod:

```bash
devpod up .
```

I configure DevPod to haven't define any code editor because I use `vim` and `tmux`. So I need to connect manualy on my devcontainer and use my vim directly on the devcontaine. But if you have configured your DevPod with a text editor, you can ignore this command.

```bash
devpod ssh
```

You need to select the right devcontainer now. When you are on the right devcontainer (with ssh or with your own text editor) you can follow the next step.

## **02-K3S installation**

It's assume that at this step you have install a linux distribution on your hardware, and you can connect this server with SSH. The simple way is to be able to SSH directly on your devcontainer.

Flux use it own helm controller, and K3S install by default a helm controller too. So, you need to disable the K3S helm controller when you install K3S on the machine. You can use this command: (this need to be done on your server, not in your devcontainer)

```bash
sudo su -

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=helm-controller" sh

sudo systemctl status k3s.service
```

You need to be the `root` user when you install K3S. You can check with `systemctl` if K3S running on your server.

Next, you need to copy the kubeconfig file on your devcontainer to be able to communicate with your cluster directly on your devcontainer.
On your server, copy the file on your home directory:

```bash
sudo cp /etc/rancher/k3s/k3s.yamll $HOME
```

Back on your devcontainer, you can copy the kubeconfig file using `scp`:
```bash
scp {NAME_OF_THE_SERVER}@{IP_OF_THE_SEVER}:/home/{NAME_OF_THE_SERVER}/k3s.yaml ./kubeconfig
```

With the kubeconfig file on your devcontainer, you can change the owner of the file to be able to execute.

```bash
chown ($USER:$USER) kubeconfig
``
Next, change your `KUBECONFIG` environment variable to point on the `kubeconfig` file. I prefer use a `.envrc` file because I use `direnv` to load and unload environment variable.

```bash
echo "KUBECONFIG=/workspace/$DEVPOD_WORKSPACE_ID/kubeconfig" > .envrc
```

Now you can check if you are connected to the cluster:

```bash
kubectl get pods -A
```

If you can see some ressources, your good to go.

If you have trouble using this file with `kubectl` you can copy the file on your home directory and execute there with the right `KUBECONFIG` environment variable.

## **03-set repository**

I assume at this point you have already created your own repository of this project with full access. You need to push code on this repository. 

### **03.01-create token for flux**

Flux need to push and pull your distant repository. I use GitHub so I created a Token to give access to my repository to flux.

You can add this Token on your `.envrc` variable but it not very secure, so you can just add temporary before bootstrap flux.

#### Creating a GitHub Personal Access Token

- Go to GitHub Settings > Developer Settings > Personal Access Tokens
- Generate a new classic token with 'repo' permissions
- Load the next environment variable on your bash session:

```bash
export GITHUB_TOKEN=<your-token>
export export GITHUB_USER=<your-username>
```

## **04-

## **05-Bootstrap flux**

You just need to bootstrap flux on your cluster with the next command

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=k3s_cluster_for_learning \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

It will took some time to reconciliate, pull the images, start the pods, enable services.
