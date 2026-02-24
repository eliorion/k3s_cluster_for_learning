# Architecture

Explainasion of the Flux structure folder.

## Tree

```bash
|-- apps
|   |-- base
|   `-- staging
|-- clusters
|   `-- staging
|       |-- apps.yaml
|       |-- monitoring.yaml
|       |-- flux-system
|-- monitoring
|   |-- configs
|   |   |-- base
|   |   `-- staging
|   `-- controllers
|       |-- base
|       `-- staging
```

This tree structure represent the fondamental structure of a single Flux repository. The Flux configuration is store on `clusters/staging/flux-system`, so when you bootstrap Flux on an empty repository, it will create this structure.

### **clusters**

The first important folder is `clusters` who contain the flux configuration who can find the `apps` and `monitoring` folders and deploy ressources on the cluster based on the containe of this folders.
This folder containe also the Flux system folder who is managed only by Flux.

#### **Files.yaml**

All the files containe the configuration to give the information to Flux where to find ressources.
For exemple, the file `apps.yaml` indiquet Flux to watch on the folder `apps/staging/`. So Flux will see if there is any `kustomization.yaml` file on this directory. The convention require to have an application folder with kustomization on it, not just the file on the `staging` directory.

#### **flux-sysem**

This directoy is own by Flux, no need to touch.

### apps

On this folder you have all your applications you when to have on the cluster. But the entrypoint of Flux is the `staging/` folder.

#### **Staging**

On this exemple there is just a staging step, but you can add `production`, `dev` part who the configuration on eatch deployment is differents. Also, this folder, `staging` containes all applications sub-folders. On eatch applications sub-folders there is a `kustomization.yaml` who indiquet what is the ressource to deploy. This kustomization file containe the namespace who the ressources will be deploy on.
This is a typicaly exemple:
```bash

#### Base

The base folder represent the part of the application who doesn't change when you are on stagin or production cluster for exemple. It containe directory for eatch application. For exemple, if you when to deploy a linkding application:

```bash
|-- apps
|   |-- base
|   |   `-- linkding
|   |       |-- deployment.yaml
|   |       |-- kustomization.yaml
|   |       |-- namespace.yaml
|   |       |-- service.yaml
|   |       `-- storage.yaml
```


