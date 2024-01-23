# Тиждень 7, задача 3. Налаштування зв'язки Flux та kind_cluster за допомогою Terraform

1. У `main.tf` використаємо наступні модулі ментора, попередньо зробивши fork кожного:

```hcl
module "kind_cluster" {
  source = "github.com/den-vasyliev/tf-kind-cluster?ref=cert_auth"
}
```
```hcl
module "tls_private_key" {
  source  = "github.com/sbazanov/tf-hashicorp-tls-keys"
}
```
```hcl
module "github_repository" {
  source  = "github.com/sbazanov/tf-github-repository"
}
```

2. Виконаємо ініціалізацію terraform:
```sh
✗ terraform init
Terraform has been successfully initialized!
```

3. Перевіримо код 
```sh
✗ tf validate
Success! The configuration is valid.
```

4. Виконаємо початкові команду `terraform apply`.
```sh
✗ tf apply
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

5. Створені ресурси:
```sh
$ tf state list
module.flux_bootstrap.flux_bootstrap_git.this
module.github_repository.github_repository.this
module.github_repository.github_repository_deploy_key.this
module.kind_cluster.kind_cluster.this
module.tls_private_key.tls_private_key.this
```

6. Для моніторингу роботи Flux на рівні його логів встановимо [CLI Flux client](https://fluxcd.io/flux/installation/)
```sh
✗ curl -s https://fluxcd.io/install.sh | bash
✗ flux get all
```

7. Перевіримо список ns:
```sh
✗ k get ns
NAME                 STATUS   AGE
default              Active   16m
flux-system          Active   15m
kube-node-lease      Active   16m
kube-public          Active   16m
kube-system          Active   16m
local-path-storage   Active   16m

✗ k get po -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-69dbf9f968-qsgq9           1/1     Running   0          16m
kustomize-controller-796b4fbf5d-jxqdx      1/1     Running   0          16m
notification-controller-78f97c759b-c8vpr   1/1     Running   0          16m
source-controller-7bc7c48d8d-c8kxk         1/1     Running   0          16m
``` 

8. Додамо в репозиторій каталог `demo` та файл `new_ns.yaml`, що містить маніфест довільного `namespace`  
```sh
$ cat new_ns.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: demo
```
- Після зміни стану репозиторію контролер Flux їх виявить та одразу синхронізує кластер згідно змін
  
У даному випадку буде створено namespace `demo`:
```sh

- Після додавання файлу у репо Flux запутимо моніторинг його логів: 
✗ flux logs -f

2024-01-23T15:10:00.025Z info GitRepository/flux-system.flux-system - stored artifact for commit 'Create new_ns.yaml' 
2024-01-23T15:10:00.765Z info Kustomization/flux-system.flux-system - server-side apply for cluster definitions completed 
2024-01-23T15:10:00.919Z info Kustomization/flux-system.flux-system - server-side apply completed 
2024-01-23T15:10:00.952Z info Kustomization/flux-system.flux-system - Reconciliation finished in 900.71219ms, next run in 10m0s

- Перевіримо чи додався новий неймспейс:
✗ k get ns 
NAME                 STATUS   AGE
default              Active   152m
demo                 Active   25m
flux-system          Active   151m
kube-node-lease      Active   152m
kube-public          Active   152m
kube-system          Active   152m
```
Новий ns додався. Все працює.

9. Згенеруємо маніфести необхідних ресурсів нашого PET проєкту за допомогою CLI Flux:
```sh
$ git clone https://github.com/sbazanov/flux-gitops-repo.git
$ cd ../flux-gitops-repo 
$ flux create source git kbot \
    --url=https://github.com/sbazanov/kbot \
    --branch=main \
    --namespace=demo \
    --export > clusters/demo/kbot-gr.yaml
$ flux create helmrelease kbot \
    --namespace=demo \
    --source=GitRepository/kbot \
    --chart="./helm" \
    --interval=1m \
    --export > clusters/demo/kbot-hr.yaml
$ git add .
$ git commit -m"add Helm release manifest"
$ git push

$ flux logs -f
2024-01-23T15:19:00.025Z info GitRepository/flux-system.flux-system - stored artifact for commit 'add Helm release manifest' 
2024-01-23T15:19:00.765Z info Kustomization/flux-system.flux-system - server-side apply for cluster definitions completed 
2024-01-23T15:19:00.919Z info Kustomization/flux-system.flux-system - server-side apply completed 
2024-01-23T15:19:00.952Z info Kustomization/flux-system.flux-system - Reconciliation finished in 900.71219ms, next run in 10m0s
```
```sh
$ kubectl get po -n demo
No resources found in demo namespace.

$ tf destroy
