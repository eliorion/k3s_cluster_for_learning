# Single node K3S VM homelab

This project is the result of the KubeCraft homelab OS cours.

## Requirements

- [] Knows linux fondamentals
- [] Knows container fondamentals (container, docker)
- [] Knows Kubernetes fondamentals (pods, deployment, service, ingress, ...)
- [] Knows `kubectl` (how to check ressource, generet chart, logging, deboging, ...)
- [] Knows `flux` and the GitOps fondamentals

Bonus:
- [] Knows devcontainer
- [] Knows DevPod (devcontainer manager)

## Documentation summary

0. [[documentations/00-.md]]
1. [[documentations/01-.md]]
2. [[documentations/02-.md]]
3. [[documentations/03-.md]]
4. [[documentations/04-.md]]
5. [[documentations/05-.md]]

## Tech stack

### Devcontainer

This project use a devcontainer to developpe the project with DevPod as devenvironnement manager. With the `.devcontainer/` Â`scripts/setup`and `mise.tomlÂ` the project is good to run.You can use your personnal dotfiles repository with DevPod, so your environnements and your tools will automaticaly be set on your devcontainer.

### Tools required

All the tools use on this project is find on the `mise.yaml` file.

### GitOps project

This project is base on a GitOps architecture. The folders architecture use the standard flux single repository architecture. More details on why this specific architecture and the meaning of all folders [[documentations/00-gitops-architecture]].
Flux is the GitOps implementation choose.

