# Projeto: GitOps na Prática com Kubernetes e ArgoCD

## Objetivo

Executar um conjunto de microserviços (Online Boutique) em Kubernetes local usando Rancher Desktop, controlado por GitOps com ArgoCD, a partir de um repositório público no GitHub.

## Ferramentas Utilizadas

* **Rancher Desktop:** Para a criação de um cluster Kubernetes local.
* **kubectl:** A ferramenta de linha de comando para interagir com o cluster.
* **ArgoCD:** A ferramenta de GitOps para automação do deploy contínuo.
* **GitHub:** Para hospedar o repositório Git que serve como referência para o ArgoCD.
* **Git:** Para o versionamento do código e manifestos.
* **Docker:** Como o container runtime gerenciado pelo Rancher Desktop.

## Pré-requisitos

Antes de começar, garanta que você tenha os seguintes softwares instalados e configurados:

- **Rancher Desktop:** Instalado e com um cluster Kubernetes em execução.
- **kubectl:** Configurado para se conectar ao cluster do Rancher Desktop.
- **Git:** Instalado e configurado com suas credenciais do GitHub.
- **Conta no GitHub:** Necessária para criar o repositório do projeto.
- **Docker:** Instalado e funcionando, gerenciado pelo Rancher Desktop.
- **ArgoCD:** (Opcional) A instalação será descrita no passo a passo, mas é bom já ter familiaridade.

## Passo a Passo da Execução

### Etapa 1: Preparação do Repositório Git

O primeiro passo é criar um repositório Git que servirá como a "fonte da verdade" para o estado da nossa aplicação.

1.  **Fork do Repositório Original:** Faça um fork do repositório [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) da Google para a sua conta no GitHub. Isso cria uma cópia pessoal do projeto.

2.  **Criação do Repositório de Manifestos:** Crie um novo repositório público no seu GitHub (ex: `gitops-microservices`). Dentro deste novo repositório, crie a seguinte estrutura de arquivos:
    ```
    /k8s
    └── online-boutique.yaml
    ```
    O conteúdo do arquivo `online-boutique.yaml` deve ser copiado do arquivo `release/kubernetes-manifests.yaml` que está no repositório que você forcou.

### Etapa 2: Instalação do ArgoCD no Cluster

Com o repositório pronto, o próximo passo é instalar o ArgoCD, que irá monitorá-lo.

1.  **Crie um Namespace:**
    ```bash
    kubectl create namespace argocd
    ```

2.  **Aplique o Manifesto de Instalação:**
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
    
### Etapa 3: Acessando a Interface do ArgoCD

Após a instalação, o ArgoCD está rodando dentro do cluster. Para acessá-lo, seguimos dois passos:

1.  **Criar um Túnel com `port-forward`:** Execute o comando abaixo em um terminal para criar uma conexão entre a sua máquina local (na porta 8080) e o serviço do ArgoCD. **Mantenha este terminal rodando.**
    ```bash
    kubectl port-forward svc/argocd-server 8080:80 -n argocd
    ```
    Após executar, acesse [`http://localhost:8080`](http://localhost:8080) no seu navegador.

2.  **Obter a Senha Inicial:** O usuário padrão é `admin`. Para obter a senha, abra um **novo terminal** e execute o comando para buscar a senha do secret do Kubernetes.
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
    ```
    O resultado será uma string codificada em Base64. Copie essa string e use uma ferramenta online de "Base64 Decode" para obter a senha real e fazer o login.
    
### Etapa 4: Configurando a Aplicação no ArgoCD

Com o ArgoCD instalado e acessível, o próximo passo é configurá-lo para monitorar nosso repositório Git.

1.  **Crie a Aplicação:** Na interface do ArgoCD, clique em **+ NEW APP**.

2.  **Preencha as Informações:** Preencha o formulário com as seguintes informações:
    - **Application Name:** `online-boutique`
    - **Project Name:** `default`
    - **Sync Policy:** `Manual`

3.  **Configure a Origem (Source):**
    - **Repository URL:** A URL do seu repositório Git criado na Etapa 1 (ex: `https://github.com/seu-usuario/gitops-microservices`).
    - **Path:** `k8s` (O diretório que contém o arquivo de manifesto).

4.  **Configure o Destino (Destination):**
    - **Cluster URL:** `https://kubernetes.default.svc` (o cluster local).
    - **Namespace:** `default`

Após preencher, clique em **CREATE** para finalizar a criação da aplicação.

### Etapa 5: Sincronização e Verificação do Deploy

Após a criação, a aplicação aparecerá no dashboard do ArgoCD com o status `Missing` e `OutOfSync`, indicando que os recursos definidos no Git ainda não existem no cluster.

1.  **Sincronize a Aplicação:** Clique no botão **SYNC** para instruir o ArgoCD a aplicar os manifestos do seu repositório no cluster.

2.  **Verifique o Status:** Após a sincronização, o status da aplicação mudará para `Healthy` e `Synced`. Isso confirma que o ArgoCD criou todos os recursos (Deployments, Services, etc.) e que os pods da aplicação estão rodando corretamente.

### Etapa 6: Acessando a Aplicação

Para visualizar a loja online, que está rodando dentro do cluster, é necessário criar um novo `port-forward` para o serviço do `frontend`.

1.  **Liste os Serviços:** Primeiro, liste os serviços no namespace `default` para encontrar o nome do frontend.
    ```bash
    kubectl get svc
    ```

2.  **Crie a Conexão:** Crie a conexão para a porta 80 do serviço `frontend`, expondo-a na porta `7070` da sua máquina. Abra um novo terminal para este comando.
    ```bash
    kubectl port-forward svc/frontend 7070:80
    ```

3.  **Acesse a Loja:** Com o comando acima rodando, acesse [`http://localhost:7070`](http://localhost:7070) no seu navegador para ver o site da Online Boutique.
