# Angular E-commerce — Deploy na AWS EC2 com K3s

Projeto de e-commerce desenvolvido com **Angular, Node.js, Express.js e MySQL**, preparado para execução local e implantação em uma instância **AWS EC2 utilizando Docker e K3s/Kubernetes**.

Repositório:

[RaybrendoT/angular-ecommerce-app](https://github.com/RaybrendoT/angular-ecommerce-app?utm_source=chatgpt.com)

O projeto original utiliza Angular no frontend, Node.js/Express.js no backend e MySQL como banco de dados.

---

# 1. Arquitetura utilizada

A implantação utiliza a seguinte estrutura:

```text
                        INTERNET
                           │
                           │
                  AWS EC2 - Ubuntu
                           │
                    ┌──────┴──────┐
                    │     K3s     │
                    │ Kubernetes  │
                    └──────┬──────┘
                           │
              Namespace: app-demo
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐       ┌───────────┐       ┌─────────┐
   │ Frontend│       │  Backend  │       │  MySQL  │
   │ Angular │       │ Node.js   │       │         │
   └────┬────┘       └─────┬─────┘       └────┬────┘
        │                  │                  │
        │                  │                  │
     Port 80           Port 3000           Port 3306
        │                  │                  │
        ▼                  ▼                  │
   NodePort             NodePort              │
    30080                30020                │
        │                  │                  │
        └───────────┬──────┘                  │
                    │                         │
                    └─────────────────────────┘
```

### Portas

| Serviço  | Porta interna |  NodePort | Finalidade                     |
| -------- | ------------: | --------: | ------------------------------ |
| Frontend |            80 | **30080** | Acesso à aplicação             |
| Backend  |          3000 | **30020** | API                            |
| MySQL    |          3306 |         — | Comunicação interna do cluster |

O MySQL **não deve ser exposto diretamente à internet**.

---

# 2. Pré-requisitos

## Máquina local

É recomendado possuir:

* Windows 10/11
* Git
* Docker Desktop
* PowerShell
* Chave `.pem` da AWS
* Repositório clonado localmente

Exemplo de diretório:

```text
C:\workspace\Tópicos Especiais ( Prof. Pedro Paulo )\
└── angular-ecommerce-app\
    └── angular-ecommerce-app\
        ├── backend\
        ├── client\
        ├── k8s\
        └── docker-compose.yml
```

---

# 3. Criar e acessar a instância EC2

A instância utilizada deve possuir Ubuntu.

Após criar a instância, existem duas formas principais de conexão:

## Opção A — SSH pelo PowerShell

No Windows:

```powershell
ssh -i "C:\Users\SEU_USUARIO\OneDrive\Documentos\webmarket.pem" ubuntu@IP_PUBLICO_DA_EC2
```

Exemplo:

```powershell
ssh -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" ubuntu@100.48.82.234
```

Após conectar, deverá aparecer algo semelhante a:

```text
ubuntu@ip-172-31-90-96:~$
```

---

## Opção B — AWS EC2 Instance Connect / Browser

Também é possível acessar a instância diretamente pelo navegador através do console da AWS.

Fluxo:

```text
AWS Console
   ↓
EC2
   ↓
Instances
   ↓
Selecionar instância
   ↓
Connect
   ↓
EC2 Instance Connect
   ↓
Connect
```

Depois da conexão, os comandos abaixo devem ser executados **dentro da instância Ubuntu**.

---

# 4. Preparação do ambiente

> Os comandos desta seção devem ser executados **na EC2**, depois de conectar via SSH ou Browser.

---

# 5. Instalação do Docker

## 5.1 Pré-requisitos

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg apt-transport-https conntrack
```

---

## 5.2 Adicionar repositório oficial do Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Adicionar o repositório:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

# 6. Instalar Docker Engine

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

# 7. Iniciar Docker

```bash
sudo systemctl enable --now docker
```

Verificar:

```bash
sudo systemctl status docker --no-pager
```

Também é possível testar:

```bash
sudo docker version
```

---

# 8. Utilizar Docker sem sudo

Opcionalmente:

```bash
sudo usermod -aG docker $USER
```

Ativar o grupo na sessão atual:

```bash
newgrp docker
```

Testar:

```bash
docker ps
```

> Caso não funcione imediatamente, saia da sessão SSH e conecte novamente.

---

# 9. Instalação do K3s

O K3s será utilizado como distribuição Kubernetes leve para executar a aplicação na EC2.

Execute:

```bash
#!/bin/bash
set -e

echo "[1/6] Atualizando pacotes..."
sudo apt update -y
sudo apt install -y curl conntrack

echo "[2/6] Instalando o K3s..."
curl -sfL https://get.k3s.io | sh -

echo "[3/6] Configurando kubectl para o usuário..."
mkdir -p ~/.kube

sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER":"$USER" ~/.kube/config
chmod 600 ~/.kube/config

unset KUBECONFIG
export KUBECONFIG="$HOME/.kube/config"

grep -q 'export KUBECONFIG=' ~/.bashrc || \
    echo 'export KUBECONFIG="$HOME/.kube/config"' >> ~/.bashrc

which kubectl
kubectl version --client
```

Depois:

```bash
source ~/.bashrc
```

Verificar o cluster:

```bash
kubectl get nodes
```

Resultado esperado:

```text
NAME                STATUS   ROLES
ip-xxx-xxx-xxx      Ready    control-plane,master
```

---

# 10. Verificar os componentes do K3s

```bash
kubectl get pods -A
```

Deve aparecerem componentes como:

```text
coredns
local-path-provisioner
metrics-server
traefik
```

Todos devem estar em estado:

```text
Running
```

---

# 11. K9s — opcional

O K9s fornece uma interface de terminal para gerenciamento do Kubernetes.

Instalação:

```bash
curl -sS https://webinstall.dev/k9s | bash
```

Atualizar o ambiente:

```bash
source ~/.bashrc
```

Configurar novamente o kubeconfig:

```bash
mkdir -p ~/.kube
```

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

```bash
sudo chown $USER:$USER ~/.kube/config
```

Executar:

```bash
k9s
```

---

# 12. Estrutura Kubernetes do projeto

O projeto utiliza uma pasta específica para os manifests:

```text
angular-ecommerce-app/
│
├── backend/
│
├── client/
│
├── k8s/
│   ├── namespace.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── mysql-statefulset.yaml
│   ├── mysql-service.yaml
│   ├── mysql-pvc.yaml
│   ├── mysql-import-job.yaml
│   └── kustomization.yaml
│
└── docker-compose.yml
```

> Os nomes dos arquivos podem variar conforme a versão atual do projeto.

---

# 13. Copiar os manifests para a EC2

No Windows PowerShell:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
-r "C:\workspace\Tópicos Especiais ( Prof. Pedro Paulo )\angular-ecommerce-app\angular-ecommerce-app\k8s" `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

Depois, na EC2:

```bash
ls -la ~/k8s
```

Verificar os YAML:

```bash
find ~/k8s -type f \( -name "*.yml" -o -name "*.yaml" \)
```

---

# 14. Aplicar os manifests com Kustomize

Como o projeto possui:

```text
kustomization.yaml
```

deve ser utilizado:

```bash
kubectl apply -k ~/k8s/
```

Não utilizar:

```bash
kubectl apply -f ~/k8s/
```

quando a intenção for processar o `kustomization.yaml`.

Antes de aplicar, é possível visualizar o resultado:

```bash
kubectl kustomize ~/k8s/
```

---

# 15. Verificar os recursos

Como a aplicação está no namespace:

```text
app-demo
```

utilizar:

```bash
kubectl get all -n app-demo
```

Pods:

```bash
kubectl get pods -n app-demo
```

Services:

```bash
kubectl get svc -n app-demo
```

---

# 16. Imagens Docker da aplicação

A aplicação utiliza imagens locais:

```text
backend:dev
frontend:dev
```

Como o K3s possui seu próprio containerd, uma imagem criada no Docker Desktop do Windows **não aparece automaticamente dentro do K3s da EC2**.

É necessário:

```text
Docker local
     ↓
docker build
     ↓
docker save
     ↓
SCP
     ↓
EC2
     ↓
k3s ctr images import
     ↓
K3s
```

---

# 17. Build do Backend

No Windows, dentro da raiz do projeto:

```powershell
docker build -t backend:dev .\backend
```

Verificar:

```powershell
docker images
```

---

# 18. Build do Frontend

```powershell
docker build -t frontend:dev .\client
```

Verificar:

```powershell
docker images
```

---

# 19. Exportar as imagens

Backend:

```powershell
docker save -o backend-dev.tar backend:dev
```

Frontend:

```powershell
docker save -o frontend-dev.tar frontend:dev
```

---

# 20. Enviar as imagens para a EC2

Backend:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
backend-dev.tar `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

Frontend:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
frontend-dev.tar `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

---

# 21. Importar imagens no K3s

Na EC2:

```bash
sudo k3s ctr images import ~/backend-dev.tar
```

```bash
sudo k3s ctr images import ~/frontend-dev.tar
```

Verificar:

```bash
sudo k3s ctr images list | grep backend
```

```bash
sudo k3s ctr images list | grep frontend
```

---

# 22. Banco de dados MySQL

O MySQL é executado dentro do Kubernetes.

Verificar:

```bash
kubectl get pods -n app-demo
```

Exemplo:

```text
mysql-0    1/1    Running
```

Verificar o Service:

```bash
kubectl get svc mysql -n app-demo
```

O MySQL utiliza:

```text
Porta: 3306
Tipo: ClusterIP
```

Portanto, ele fica disponível apenas internamente no cluster.

---

# 23. Variáveis do MySQL

No cenário utilizado:

```text
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=loja
MYSQL_USER=pedro
```

Banco:

```text
loja
```

---

# 24. Importar o banco de dados

Caso exista um arquivo:

```text
sql_dump.sql
```

enviá-lo para a EC2.

No Windows:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
"C:\workspace\Tópicos Especiais ( Prof. Pedro Paulo )\angular-ecommerce-app\angular-ecommerce-app\backend\sql_dump.sql" `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

---

# 25. Copiar o dump para o Pod MySQL

Na EC2:

```bash
kubectl cp ~/sql_dump.sql app-demo/mysql-0:/tmp/sql_dump.sql
```

Verificar:

```bash
kubectl exec -n app-demo mysql-0 -- ls -lh /tmp/sql_dump.sql
```

---

# 26. Importar o dump

```bash
kubectl exec -n app-demo mysql-0 -- \
sh -c 'mysql -u root -proot loja < /tmp/sql_dump.sql'
```

O MySQL pode apresentar:

```text
mysql: [Warning] Using a password on the command line interface can be insecure.
```

Isso é apenas um **warning**, não significa que a importação falhou.

---

# 27. Verificar as tabelas

Entrar no MySQL:

```bash
kubectl exec -it -n app-demo mysql-0 -- mysql -u root -p loja
```

Senha:

```text
root
```

Depois:

```sql
SHOW TABLES;
```

Para sair:

```sql
exit;
```

---

# 28. Verificar o Backend

Ver os Pods:

```bash
kubectl get pods -n app-demo
```

Ver logs:

```bash
kubectl logs -n app-demo deployment/backend
```

Resultado esperado:

```text
Server is running on port 3000 using production env.
MySQL is connected...
```

Também é possível acompanhar os logs:

```bash
kubectl logs -f -n app-demo deployment/backend
```

---

# 29. Verificar o Frontend

```bash
kubectl get pods -n app-demo
```

Exemplo:

```text
frontend-xxxxx    1/1    Running
frontend-yyyyy    1/1    Running
```

Ver logs:

```bash
kubectl logs -n app-demo deployment/frontend
```

---

# 30. Services da aplicação

Verificar:

```bash
kubectl get svc -n app-demo
```

No cenário atual:

```text
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
backend    NodePort    10.43.24.246   <none>        3000:30020/TCP
frontend   NodePort    10.43.27.169   <none>        80:30080/TCP
mysql      ClusterIP   None           <none>        3306/TCP
```

---

# 31. Acessar a aplicação

O frontend está disponível através da porta:

```text
30080
```

Acessar no navegador:

```text
http://IP_PUBLICO_DA_EC2:30080
```

Exemplo:

```text
http://100.48.82.234:30080
```

O backend está disponível através da porta:

```text
30020
```

Exemplo:

```text
http://100.48.82.234:30020
```

---

# 32. Configuração do Security Group da AWS

No Security Group da EC2, liberar:

```text
TCP 30080
```

para acesso ao frontend.

Caso seja necessário acessar diretamente a API:

```text
TCP 30020
```

### Não liberar:

```text
TCP 3306
```

O MySQL deve permanecer interno ao Kubernetes.

---

# 33. Atualização da aplicação

Uma alteração no código local **não atualiza automaticamente o Pod da EC2**.

A aplicação está dentro de uma imagem Docker.

Portanto:

```text
Código alterado
      ↓
Docker Build
      ↓
Nova imagem
      ↓
docker save
      ↓
SCP
      ↓
k3s ctr images import
      ↓
Rollout Restart
```

---

# 34. Alteração do API_URL do Frontend

Esse ponto é especialmente importante para Angular.

Se o `API_URL` estiver dentro do `environment` do Angular, ele é incorporado durante o processo de build.

Por exemplo:

```typescript
apiUrl: 'http://IP_DA_EC2:30020'
```

Depois de alterar o valor:

```text
environment
      ↓
npm build
      ↓
arquivos estáticos
      ↓
imagem Docker
      ↓
K3s
```

Portanto, **alterar o `environment` exige um novo build da imagem do frontend**.

Não é suficiente apenas reiniciar o Pod.

---

# 35. Rebuild do Frontend após alterar API_URL

No Windows:

```powershell
docker build -t frontend:dev .\client
```

Exportar:

```powershell
docker save -o frontend-dev.tar frontend:dev
```

Enviar:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
frontend-dev.tar `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

Na EC2:

```bash
sudo k3s ctr images import ~/frontend-dev.tar
```

Reiniciar o Deployment:

```bash
kubectl rollout restart deployment frontend -n app-demo
```

Acompanhar:

```bash
kubectl rollout status deployment frontend -n app-demo
```

Verificar:

```bash
kubectl get pods -n app-demo
```

---

# 36. Atualização do Backend

Caso o código do backend seja alterado:

```powershell
docker build -t backend:dev .\backend
```

Exportar:

```powershell
docker save -o backend-dev.tar backend:dev
```

Enviar:

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
backend-dev.tar `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

Na EC2:

```bash
sudo k3s ctr images import ~/backend-dev.tar
```

Reiniciar:

```bash
kubectl rollout restart deployment backend -n app-demo
```

Verificar:

```bash
kubectl rollout status deployment backend -n app-demo
```

---

# 37. Atualizar Frontend e Backend juntos

Quando os dois forem alterados:

### Windows

```powershell
docker build -t backend:dev .\backend
docker build -t frontend:dev .\client
```

```powershell
docker save -o backend-dev.tar backend:dev
docker save -o frontend-dev.tar frontend:dev
```

```powershell
scp -i "C:\Users\raybr\OneDrive\Documentos\webmarket.pem" `
backend-dev.tar `
frontend-dev.tar `
ubuntu@IP_PUBLICO_DA_EC2:~/
```

### EC2

```bash
sudo k3s ctr images import ~/backend-dev.tar
sudo k3s ctr images import ~/frontend-dev.tar
```

Depois:

```bash
kubectl rollout restart deployment backend -n app-demo
kubectl rollout restart deployment frontend -n app-demo
```

Acompanhar:

```bash
kubectl rollout status deployment backend -n app-demo
```

```bash
kubectl rollout status deployment frontend -n app-demo
```

---

# 38. Verificação final

Executar:

```bash
kubectl get pods -n app-demo
```

Todos os Pods da aplicação devem estar:

```text
Running
```

Depois:

```bash
kubectl get svc -n app-demo
```

Esperado:

```text
backend     NodePort
frontend    NodePort
mysql       ClusterIP
```

Verificar backend:

```bash
kubectl logs -n app-demo deployment/backend
```

Verificar imagens:

```bash
sudo k3s ctr images list | grep -E 'backend|frontend'
```

---

# 39. Comandos úteis

## Ver todos os Pods

```bash
kubectl get pods -A
```

## Ver Pods da aplicação

```bash
kubectl get pods -n app-demo
```

## Ver Services

```bash
kubectl get svc -n app-demo
```

## Ver Deployments

```bash
kubectl get deployments -n app-demo
```

## Ver StatefulSets

```bash
kubectl get statefulsets -n app-demo
```

## Ver eventos

```bash
kubectl get events -n app-demo --sort-by=.lastTimestamp
```

## Descrever um Pod

```bash
kubectl describe pod NOME_DO_POD -n app-demo
```

## Logs do Backend

```bash
kubectl logs -n app-demo deployment/backend
```

## Logs do Frontend

```bash
kubectl logs -n app-demo deployment/frontend
```

## Reiniciar Backend

```bash
kubectl rollout restart deployment backend -n app-demo
```

## Reiniciar Frontend

```bash
kubectl rollout restart deployment frontend -n app-demo
```

---

# 40. Importante sobre `kubectl`

Se aparecer:

```text
permission denied
/etc/rancher/k3s/k3s.yaml
```

utilizar temporariamente:

```bash
sudo kubectl get pods -n app-demo
```

Porém, quando o kubeconfig estiver configurado corretamente para o usuário:

```bash
kubectl get pods -n app-demo
```

deve funcionar sem `sudo`.

---

# 41. Fluxo completo de implantação

O processo completo utilizado neste projeto pode ser resumido em:

```text
1. Criar EC2
        ↓
2. Conectar via SSH ou Browser
        ↓
3. Instalar Docker
        ↓
4. Instalar K3s
        ↓
5. Configurar kubectl
        ↓
6. Copiar manifests k8s
        ↓
7. Build das imagens Docker no Windows
        ↓
8. docker save
        ↓
9. SCP das imagens para EC2
        ↓
10. k3s ctr images import
        ↓
11. kubectl apply -k
        ↓
12. Criar/validar MySQL
        ↓
13. Importar sql_dump.sql
        ↓
14. Verificar Backend
        ↓
15. Verificar Frontend
        ↓
16. Liberar NodePorts no Security Group
        ↓
17. Acessar:
    http://IP_EC2:30080
```

---

# 42. Fluxo para uma nova versão

Depois que a aplicação já estiver funcionando:

```text
Alterar código
     ↓
Commit / Push GitHub
     ↓
Atualizar código local
     ↓
Alterar environment se necessário
     ↓
Docker Build
     ↓
Docker Save
     ↓
SCP para EC2
     ↓
Importar no K3s
     ↓
Rollout Restart
     ↓
Testar aplicação
```

Para o frontend:

```text
Alterou API_URL?
      ↓
SIM
      ↓
REBUILD obrigatório
      ↓
Nova imagem frontend:dev
      ↓
Importar no K3s
      ↓
Restart do frontend
```

---

# 43. Comandos rápidos — manutenção

### Frontend

```bash
kubectl rollout restart deployment frontend -n app-demo
```

### Backend

```bash
kubectl rollout restart deployment backend -n app-demo
```

### Ver status

```bash
kubectl get pods -n app-demo
```

### Ver serviços

```bash
kubectl get svc -n app-demo
```

### Ver logs do backend

```bash
kubectl logs -f -n app-demo deployment/backend
```

### Ver eventos

```bash
kubectl get events -n app-demo --sort-by=.lastTimestamp
```

---

# 44. Resultado esperado

Ao final, a aplicação estará executando na seguinte arquitetura:

```text
                 AWS EC2
                    │
                    ▼
                  K3s
                    │
             namespace app-demo
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
   Frontend       Backend       MySQL
   Angular        Node.js       Database
       │            │            │
     :80          :3000        :3306
       │            │            │
     :30080       :30020       ClusterIP
       │            │
       └──────┬─────┘
              │
              ▼
       Internet / Browser
```

Acesso principal:

```text
http://IP_PUBLICO_DA_EC2:30080
```

Backend:

```text
http://IP_PUBLICO_DA_EC2:30020
```

MySQL:

```text
Somente dentro do cluster
```

---

# 45. Observação sobre produção

Esta configuração foi criada para **ambiente de estudo/teste**.

Não é recomendado utilizar em produção mantendo:

```text
root
root
```

como credencial do MySQL, nem deixar credenciais diretamente nos manifests.

Em um ambiente de produção, recomenda-se utilizar:

* Kubernetes Secrets
* AWS Secrets Manager
* HTTPS/TLS
* domínio
* Ingress
* Load Balancer
* imagens armazenadas em registry
* CI/CD
* backups automatizados
* políticas de segurança do Security Group
* Persistent Volumes adequadamente dimensionados

---

# 46. Repositório

Projeto utilizado:

[GitHub — RaybrendoT/angular-ecommerce-app](https://github.com/RaybrendoT/angular-ecommerce-app?utm_source=chatgpt.com)

Fork baseado no projeto original:

[Pbrantis/angular-ecommerce-app](https://github.com/Pbrantis/angular-ecommerce-app?utm_source=chatgpt.com)
