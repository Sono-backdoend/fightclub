# Fightclub no Kubernetes (Minikube)

Implantação do projeto desenvolvido no Mensal 1 em um cluster Kubernetes local com Minikube. A aplicação é composta por três componentes, todos executando dentro do cluster:

- **Frontend:** Next.js, interface acessada pelo navegador.
- **Backend:** Next.js (API) com Prisma, responsável pela autenticação e pelo acesso aos dados.
- **Banco de dados:** PostgreSQL 16, com os dados persistidos em um volume.

**Integrantes:** <!-- preencher com os nomes do grupo -->

---

## Arquitetura

```
Navegador
   │  minikube service sono-frontend --url
   ▼
Service sono-frontend (NodePort, porta 80 → 3000)
   │
   ├──► Pod frontend 1 ┐
   └──► Pod frontend 2 ┘  rewrite /api/* (lado do servidor)
                │
                ▼
      Service sono-backend (ClusterIP, porta 3001)
                │
   ├──► Pod backend 1 ┐
   ├──► Pod backend 2 ├── initContainer: prisma migrate deploy
   └──► Pod backend 3 ┘
                │
                ▼
      Service postgres (ClusterIP, porta 5432)
                │
                ▼
      Pod postgres (1 réplica) ──► PersistentVolumeClaim postgres-data (1 Gi)
```

| Componente | Deployment | Réplicas | Service | Tipo | Porta |
|---|---|---|---|---|---|
| Frontend | `sono-frontend` | 2 | `sono-frontend` | NodePort | 80 → 3000 |
| Backend | `sono-backend` | 3 | `sono-backend` | ClusterIP | 3001 |
| Banco | `postgres` | 1 | `postgres` | ClusterIP | 5432 |

### Comunicação entre os componentes

1. O navegador acessa o frontend pelo Service `sono-frontend`, do tipo **NodePort**, o único exposto para fora do cluster.
2. O Service distribui as requisições entre os 2 pods do frontend.
3. As chamadas para `/api/*` são repassadas pelo servidor do Next.js (rewrite no `frontend/next.config.ts`) para `http://sono-backend:3001`. O navegador nunca acessa o backend diretamente.
4. O Service `sono-backend`, do tipo **ClusterIP**, distribui as requisições entre os 3 pods do backend.
5. O backend acessa o banco pela `DATABASE_URL`, que aponta para o Service `postgres`, também do tipo **ClusterIP**.

Dentro do cluster, os componentes se encontram pelo **nome do Service** (resolvido pelo DNS interno do Kubernetes), e não por IP ou `localhost`.

### Recursos do Kubernetes utilizados

| Recurso | Uso no projeto |
|---|---|
| **Deployment** | Gerencia os pods de cada componente e mantém o número de réplicas declarado. Se um pod é excluído, o ReplicaSet cria outro automaticamente. |
| **Service** | Dá um nome fixo a cada componente e balanceia as requisições entre os pods. |
| **ConfigMap** (`sono-config`) | Configurações não sensíveis: `POSTGRES_DB` e `NODE_ENV`. |
| **Secret** (`sono-secrets`) | Dados sensíveis: usuário e senha do banco, URLs de conexão e `AUTH_SECRET` (chave de assinatura do JWT). |
| **PersistentVolumeClaim** (`postgres-data`) | Guarda os dados do PostgreSQL fora do pod, para que sobrevivam à recriação do pod. |
| **initContainer** (`migrate`) | Roda `prisma migrate deploy` antes do backend iniciar, criando ou atualizando as tabelas. |
| **Readiness e liveness probes** | Verificam a rota `/api/health`. O pod só recebe tráfego quando está pronto e é reiniciado se parar de responder. |

O banco tem apenas 1 réplica porque várias instâncias do PostgreSQL escrevendo no mesmo volume corromperiam os dados. O Deployment do banco usa `strategy: Recreate` para garantir que o pod antigo pare antes do novo subir.

---

## Estrutura do repositório

```
fightclub/
├── frontend/                 código do frontend, Dockerfile e .dockerignore
├── backend/                  código do backend, Dockerfile, .dockerignore e Prisma
├── k8s/
│   ├── 00-configmap.yaml
│   ├── 01-secret.example.yaml
│   ├── 02-postgres.yaml
│   ├── 03-backend.yaml
│   └── 04-frontend.yaml
├── evidencias/               capturas de tela da aplicação e do cluster
└── README.md
```

---

## Pré-requisitos

- Linux ou Windows com WSL2 (Ubuntu)
- [Docker](https://docs.docker.com/engine/install/) funcionando sem `sudo` (no Windows, Docker Desktop com a integração WSL ativada)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- `openssl` (para gerar a chave do JWT)

O `kubectl` não precisa ser instalado separadamente: o Minikube fornece um compatível com o cluster (ver passo 2).

---

## Passo a passo

### 1. Clonar o repositório

```bash
git clone <url-do-repositorio>
cd fightclub
```

### 2. Iniciar o Minikube

```bash
minikube start
```

Crie um alias para usar o `kubectl` do Minikube:

```bash
echo 'alias kubectl="minikube kubectl --"' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

O node `minikube` deve aparecer com status `Ready`.

### 3. Construir e carregar as imagens

As imagens são construídas no Docker local e depois carregadas para dentro do Minikube, sem necessidade de registry:

```bash
docker build -t sono-frontend:1.0 ./frontend
docker build -t sono-backend:1.0 ./backend

minikube image load sono-frontend:1.0
minikube image load sono-backend:1.0
```

Confira se as imagens chegaram ao cluster:

```bash
minikube image ls | grep sono
```

> As versões recentes do Minikube usam o runtime **containerd**, com o qual o `eval $(minikube docker-env)` não funciona de forma confiável. Por isso usamos `minikube image load`.

### 4. Criar o Secret

Copie o arquivo de exemplo:

```bash
cp k8s/01-secret.example.yaml k8s/01-secret.yaml
```

Gere uma chave para o JWT:

```bash
openssl rand -base64 32
```

Edite `k8s/01-secret.yaml` e cole a chave gerada em `AUTH_SECRET`. Os demais valores podem ser mantidos para o ambiente local. O arquivo `01-secret.yaml` está no `.gitignore` e não deve ser enviado ao repositório.

### 5. Aplicar os manifestos

Aplique os arquivos **nesta ordem**. O backend depende do banco para rodar as migrations, então esperamos o PostgreSQL ficar pronto antes de subir o backend.

```bash
kubectl apply -f k8s/00-configmap.yaml
kubectl apply -f k8s/01-secret.yaml
kubectl apply -f k8s/02-postgres.yaml

kubectl wait --for=condition=ready pod -l app=postgres --timeout=180s

kubectl apply -f k8s/03-backend.yaml
kubectl rollout status deployment sono-backend

kubectl apply -f k8s/04-frontend.yaml
kubectl rollout status deployment sono-frontend
```

> Não use `kubectl apply -f k8s/` com a pasta inteira, porque isso aplicaria também o arquivo de exemplo do Secret.

### 6. Popular o banco (seed)

Cria o usuário administrador inicial. O script verifica se o admin já existe, então pode ser executado mais de uma vez sem duplicar dados.

```bash
kubectl exec -it deploy/sono-backend -c sono-backend -- npx tsx prisma/seed.ts
```

Saída esperada: `Admin criado: admin@projeto.com`.

### 7. Acessar a aplicação

```bash
minikube service sono-frontend --url
```

Abra a URL exibida (algo como `http://127.0.0.1:XXXXX`) no navegador e **mantenha o terminal aberto** enquanto usa a aplicação. No WSL com o driver Docker, o IP do Minikube não é acessível diretamente pelo Windows, e esse comando cria o túnel necessário.

Credenciais do administrador, na rota `/admin/login`:

- **E-mail:** `admin@projeto.com`
- **Senha:** `12345`

---

## Verificação

```bash
kubectl get pods -o wide
kubectl get services
kubectl get deployments
kubectl get configmap,secret,pvc
```

O resultado esperado são 6 pods com status `Running` e `READY 1/1`: 2 do frontend, 3 do backend e 1 do PostgreSQL.

Para ver o log das migrations de um pod do backend:

```bash
kubectl logs <nome-do-pod-do-backend> -c migrate
```

---

## Demonstrações

### Balanceamento de carga

A rota `/api/health` do backend retorna o nome do pod que respondeu. Fazendo várias requisições de dentro do cluster, as respostas se alternam entre os 3 pods:

```bash
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  sh -c 'for i in $(seq 1 10); do curl -s http://sono-backend:3001/api/health; echo; done'
```

> O teste é feito de dentro do cluster porque o `kubectl port-forward` conecta sempre ao mesmo pod, e o navegador reaproveita a conexão, o que esconderia o balanceamento.

### Recriação automática de pods

Em um terminal, acompanhe os pods:

```bash
kubectl get pods -w
```

Em outro, exclua um pod do backend:

```bash
kubectl delete pod <nome-de-um-pod-do-backend>
```

O pod excluído aparece como `Terminating`, e um novo, com outro nome, é criado imediatamente e passa por `Init:0/1`, `PodInitializing` e `Running`. O ReplicaSet criado pelo Deployment percebe que restaram menos réplicas do que as 3 declaradas e cria outra para voltar ao estado desejado.

### Persistência dos dados

Excluindo o pod do banco, ele é recriado e os dados continuam disponíveis, porque ficam no PersistentVolumeClaim e não dentro do pod:

```bash
kubectl delete pod -l app=postgres
```

Depois que o novo pod ficar pronto, o login com o mesmo administrador continua funcionando.

---

## Atualizando uma imagem

Ao alterar o código, gere a imagem com uma **nova tag**, carregue no Minikube e atualize o campo `image` no manifesto correspondente:

```bash
docker build -t sono-backend:1.1 ./backend
minikube image load sono-backend:1.1
# altere sono-backend:1.0 para sono-backend:1.1 em k8s/03-backend.yaml (initContainer e container)
kubectl apply -f k8s/03-backend.yaml
```

Recarregar uma imagem com a mesma tag nem sempre substitui a versão que já está no Minikube.

Alterações apenas no Secret ou no ConfigMap não exigem nova imagem, mas os pods precisam ser recriados para ler os novos valores:

```bash
kubectl apply -f k8s/01-secret.yaml
kubectl rollout restart deployment sono-backend
```

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| Pod em `ErrImagePull` ou `ImagePullBackOff` | A imagem não foi carregada no Minikube ou a tag não confere | Rodar `minikube image load` e conferir a tag no manifesto |
| Backend em `Init:Error` | As migrations falharam (banco fora do ar ou variável faltando) | `kubectl logs <pod> -c migrate` |
| Pod em `CrashLoopBackOff` | A aplicação quebra ao iniciar | `kubectl logs <pod>` |
| Pod `Running` mas `READY 0/1` | A probe não encontra `/api/health` | Conferir se a rota existe na imagem |
| Login retorna erro 500 | `AUTH_SECRET` ausente ou com nome errado no Secret | `kubectl describe secret sono-secrets` e conferir o nome exato da chave |
| Pods em `Unknown` após reiniciar o computador | O cluster foi desligado | `kubectl rollout restart deployment sono-backend` |

---

## Removendo tudo

```bash
kubectl delete -f k8s/04-frontend.yaml -f k8s/03-backend.yaml -f k8s/02-postgres.yaml \
  -f k8s/01-secret.yaml -f k8s/00-configmap.yaml
minikube delete
```

---

## Evidências

As capturas de tela estão na pasta [`evidencias/`](./evidencias):

- Imagens no Minikube, node, pods, services, deployments, ConfigMap, Secret e PVC
- Log das migrations, teste de balanceamento de carga e recriação automática de pod
- Aplicação acessível pelo navegador, tela de login e área logada do administrador
