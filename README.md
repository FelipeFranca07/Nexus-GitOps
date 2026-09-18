# Nexus GitOps

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Kustomize](https://img.shields.io/badge/Kustomize-326CE5?style=flat&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logo=argo&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

Repositório de configuração declarativa (padrão **GitOps**) para a
[`Nexus-Api`](https://github.com/FelipeFranca07/Nexus-Api), gerenciado pelo **ArgoCD**. Este repositório é a
**única** fonte da verdade do estado desejado do cluster — nenhum deploy é feito com `kubectl apply` manual, e
nenhum outro pipeline tem credenciais do cluster.

> Este repositório documenta um **padrão de arquitetura de referência**, construído do zero para estudo e
> portfólio — não é uma extração de um ambiente de produção real.

![Arquitetura CI/CD + GitOps](architecture.svg)

## Índice

- [Por que isso existe](#por-que-isso-existe)
- [A decisão central: sync automático em dev/staging, manual em prod](#a-decisão-central-sync-automático-em-devstaging-manual-em-prod)
- [Padrão App of Apps](#padrão-app-of-apps)
- [Estrutura](#estrutura)
- [Fluxo de promoção](#fluxo-de-promoção)
- [Bootstrap em um cluster (do zero)](#bootstrap-em-um-cluster-do-zero)
- [Comandos úteis / como testar](#comandos-úteis--como-testar)
- [Como adaptar para o seu ambiente](#como-adaptar-para-o-seu-ambiente)
- [Boas práticas de segurança](#boas-práticas-de-segurança)
- [Repositório da aplicação](#repositório-da-aplicação)

## Por que isso existe

Um cluster Kubernetes gerenciado por `kubectl apply` manual (ou por um pipeline de CI que aplica direto) tem um
problema de fundo: o estado real do cluster e o estado registrado em algum lugar **divergem com o tempo**,
silenciosamente — alguém roda um `kubectl edit` numa emergência, um `helm upgrade` é feito de uma máquina local,
e seis meses depois ninguém sabe mais qual é o estado "correto". GitOps resolve isso invertendo o modelo: em
vez de *empurrar* mudanças para o cluster, um agente (o ArgoCD) roda **dentro** do cluster e *puxa* o estado
desejado do Git continuamente, corrigindo qualquer divergência (drift) sozinho. O histórico de commits deste
repositório é, literalmente, o histórico de deploys.

## A decisão central: sync automático em dev/staging, manual em prod

GitOps não significa "tudo 100% automático até produção" — isso trocaria um risco (deploy manual sem
rastreabilidade) por outro (qualquer commit no `main` vai direto pro cluster de produção sem revisão humana).
Este repositório trata isso como uma escolha explícita por ambiente, configurada por Application do ArgoCD:

| Ambiente | `syncPolicy.automated` | O que isso significa na prática |
|---|---|---|
| `dev` | `prune: true, selfHeal: true` | Todo push no `main` da API vira deploy em segundos, sem intervenção — é pra isso que `dev` existe |
| `staging` | `prune: true, selfHeal: true` | Mesmo comportamento, mas só recebe uma tag depois que alguém promove via PR (ver [Fluxo de promoção](#fluxo-de-promoção)) |
| `prod` | **ausente** (sem bloco `automated`) | O ArgoCD detecta a diferença e mostra `OutOfSync`, mas só aplica quando alguém clica "Sync" manualmente no ArgoCD |

Essa assimetria é intencional: o `syncPolicy` de cada `Application` em `argocd/applications/` é o único lugar
onde essa política é decidida — não existe lógica condicional escondida em pipeline nenhum.

## Padrão App of Apps

Em vez de registrar cada `Application` manualmente no ArgoCD (`kubectl apply -f` uma por uma, um processo que
não escala e é fácil de esquecer), existe uma única Application raiz (`argocd/root-app.yaml`) que aponta para a
pasta `argocd/applications/`. O ArgoCD sincroniza essa pasta como se fosse mais um "deploy", e cria/gerencia as
Applications filhas (dev/staging/prod) automaticamente. Isso tem uma consequência prática importante: **agregar
um ambiente ou uma aplicação nova ao GitOps é só adicionar um arquivo YAML nessa pasta e commitar** — não é
preciso rodar nenhum comando `argocd app create` manualmente.

## Estrutura

```
apps/nexus-api/
├── base/                              # Deployment, Service e ConfigMap "puros"
│   ├── deployment.yaml                # readiness em /ready, liveness em /health
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/                           # namespace nexus-dev · 1 réplica · tag atualizada automaticamente pelo CI
    ├── staging/                       # namespace nexus-staging · 2 réplicas · promoção manual via PR
    └── prod/                          # namespace nexus-prod · 3 réplicas · limits maiores · sync manual

argocd/
├── project.yaml                       # AppProject "nexus" — restringe quais namespaces/recursos são permitidos
├── root-app.yaml                      # App of Apps — aponta para argocd/applications
└── applications/
    ├── nexus-api-dev.yaml             # sync automático (auto-sync + self-heal + prune)
    ├── nexus-api-staging.yaml         # sync automático
    └── nexus-api-prod.yaml            # sync manual (aprovação humana no ArgoCD)
```

Cada overlay usa `kustomize` para herdar o `base/` e só sobrescrever o que muda por ambiente (namespace,
réplicas, tag da imagem, limits) — o `Deployment`/`Service`/`ConfigMap` em si é definido uma única vez.

## Fluxo de promoção

- **dev**: o pipeline de CI da [`Nexus-Api`](https://github.com/FelipeFranca07/Nexus-Api) atualiza a tag da
  imagem no overlay `dev` a cada push no `main` (via `kustomize edit set image`). O ArgoCD sincroniza sozinho
  em segundos.
- **staging/prod**: promoção é manual — copia-se a tag já validada em `dev` para o `kustomization.yaml` do
  overlay correspondente, via Pull Request (o PR em si já é a auditoria: quem promoveu, quando, o quê). Staging
  sincroniza sozinho após o merge; prod exige clicar "Sync" no ArgoCD mesmo depois do merge.

## Bootstrap em um cluster (do zero)

### Passo 1 — Instalar o ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Passo 2 — Criar o AppProject e a Application raiz (App of Apps)

```bash
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/root-app.yaml
```

A partir daqui, o ArgoCD cria as três Applications filhas (`nexus-api-dev/staging/prod`) e sincroniza cada
overlay no seu respectivo namespace — nenhum outro `kubectl apply` é necessário.

### Passo 3 — Acessar a UI/CLI do ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# usuário: admin / senha inicial:
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

```bash
argocd login localhost:8080
argocd app list
```

### Passo 4 — Confirmar que o cluster convergiu

```bash
argocd app get nexus-api-dev
kubectl get pods -n nexus-dev
```

### Checklist final

| Item | Onde vive |
|---|---|
| ArgoCD instalado e com o pod `argocd-server` `Available` | Namespace `argocd` |
| `AppProject nexus` aplicado | `argocd/project.yaml` |
| `Application nexus-root` aplicada (App of Apps) | `argocd/root-app.yaml` |
| `GITOPS_PAT` configurado no repo da aplicação | Ver README do [`Nexus-Api`](https://github.com/FelipeFranca07/Nexus-Api#setup-passo-a-passo-do-zero) |

## Comandos úteis / como testar

**Renderizar um overlay localmente, sem aplicar nada (útil pra revisar um PR de promoção):**
```bash
kubectl kustomize apps/nexus-api/overlays/staging
```

**Ver o diff entre o que está no Git e o que está rodando no cluster:**
```bash
argocd app diff nexus-api-prod
```

**Forçar uma sincronização manual (ex.: promover produção depois de revisar o diff):**
```bash
argocd app sync nexus-api-prod
```

**Ver o histórico de syncs de uma Application (e fazer rollback, se precisar):**
```bash
argocd app history nexus-api-prod
argocd app rollback nexus-api-prod <ID>
```

**Checar se a `Application` está `Synced`/`Healthy` sem abrir a UI:**
```bash
kubectl get applications -n argocd
```

**Validar a sintaxe de todos os overlays antes de commitar (útil como pre-commit/CI check):**
```bash
for env in dev staging prod; do kubectl kustomize apps/nexus-api/overlays/$env > /dev/null || echo "FALHOU: $env"; done
```

## Como adaptar para o seu ambiente

- **Mais serviços além do `nexus-api`?** Crie `apps/<novo-serviço>/base` e `overlays/`, e adicione as
  Applications correspondentes em `argocd/applications/` — o App of Apps pega automaticamente, sem tocar em
  `root-app.yaml`.
- **Quer `staging` também com sync manual?** Remova o bloco `syncPolicy.automated` de
  `argocd/applications/nexus-api-staging.yaml`, igual já é feito em `prod`.
- **Sem Kustomize, prefere Helm?** A estrutura de pastas (`base` + `overlays` por ambiente) mapeia direto para
  um `Chart.yaml` + `values-<env>.yaml` — só o `source` de cada `Application` muda (`path` vira `chart` +
  `helm.valueFiles`).
- **Múltiplos clusters (não namespaces) por ambiente?** Troque `destination.server` de cada `Application` pelo
  endpoint do cluster correspondente, em vez de variar só o `namespace`.

## Boas práticas de segurança

- **Este repositório nunca contém segredos.** ConfigMaps aqui carregam só configuração não-sensível
  (`APP_VERSION`); credenciais reais (ex.: string de conexão de banco) devem vir de um `Secret` do Kubernetes
  gerenciado fora do Git (Sealed Secrets, External Secrets Operator, Vault) — nunca committado em texto claro,
  nem mesmo "só pra homologação".
- **O `AppProject` restringe o blast radius.** `destinations` limita as Applications deste projeto a criar
  recursos só em namespaces `nexus-*`; um `Application` mal configurada não consegue, por exemplo, escrever em
  `kube-system`.
- **`prod` nunca tem `syncPolicy.automated`.** Mesmo com o histórico de commits como auditoria, uma mudança em
  produção passa por uma confirmação humana explícita antes de ser aplicada — self-heal automático corrige
  drift, mas não promove revisões novas sozinho.
- **O token que escreve neste repositório (`GITOPS_PAT`, configurado no repo da aplicação) é fine-grained e
  restrito só a este repositório** — nunca um token com acesso a todos os repositórios da conta. Ver
  [Boas práticas de segurança](https://github.com/FelipeFranca07/Nexus-Api#boas-práticas-de-segurança) no
  repositório da aplicação.

## Repositório da aplicação

Código-fonte, testes e pipeline de CI/CD ficam em [`Nexus-Api`](https://github.com/FelipeFranca07/Nexus-Api).

---

*Este documento descreve um padrão de arquitetura de referência, construído do zero para estudo e portfólio —
não uma extração de ambiente de produção real.*
