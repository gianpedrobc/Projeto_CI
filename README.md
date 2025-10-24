# Pipeline GitOps de CI/CD com GitHub Actions, Docker e ArgoCD

Este projeto implementa um **pipeline completo de CI/CD** para uma aplicação FastAPI simples, baseado em práticas de GitOps. Ele automatiza o ciclo de vida da aplicação desde o commit do código até a implantação em um cluster Kubernetes com Rancher Desktop.

---

## 1. Visão Geral

O objetivo é automatizar todo o ciclo de vida da aplicação, garantindo que **cada mudança de código** seja automaticamente testada, containerizada e implantada.

### Arquitetura da Solução

Para seguir as melhores práticas de GitOps, utilizamos dois repositórios:

#### Repositório de Aplicação (Projeto-CI)
- Contém o código-fonte da aplicação FastAPI e o **Dockerfile**.
- CI: Workflow GitHub Actions testa, constrói a imagem Docker e publica no Docker Hub.
- Gatilho de CD: Atualiza automaticamente o repositório de manifestos (`projeto-cd`) com a nova tag da imagem.

#### Repositório de Manifestos (Projeto-CD)
- Contém os manifestos YAML do Kubernetes (Fonte Única da Verdade).
- CD: ArgoCD monitora continuamente este repositório e sincroniza automaticamente o cluster Kubernetes.

### Tecnologias Utilizadas
- **CI**: GitHub Actions  
- **Registro de Imagem**: Docker Hub  
- **CD**: ArgoCD  
- **Aplicação**: FastAPI  
- **Orquestrador**: Kubernetes (via Rancher Desktop)

---

## Pré-requisitos

Antes de iniciar, você precisa ter:
- Conta no **GitHub**  
- Conta no **Docker Hub**  
- **Rancher Desktop** instalado e Kubernetes habilitado  
- **Git** instalado e configurado  

---

## Configuração Passo a Passo

### Configurar Repositórios
- Crie dois repositórios no GitHub:
  - `Projeto-CI` (Aplicação)
  - `Projeto-CD` (Manifestos)
- Clone ambos na sua máquina local.

---

### Repositório de Aplicação (`Projeto-CI`)

Estrutura de arquivos:

projeto-ci/
├── .github/
│ └── workflows/
│    └──GitActions.yml
├── app/
│ ├── main.py
│ └── Dockerfile
└── README.md


#### app/main.py
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "teste CI/CD"}
```

# app/Dockerfile
```
FROM python:3.9-slim

WORKDIR /code

RUN pip install "fastapi[standard]"

COPY ./main.py /code/

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

# .github/workflows/GitActions.yml
```
name: CI - build a image e atualiza o repository de CD

on:
  push:
    branches: [ "main" ]

jobs:
  build_and_update:
    runs-on: ubuntu-latest
    steps:
  
      - name: Checkout App Repo
        uses: actions/checkout@v4

      
      - name: Login do Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

```

# Configurar Credenciais e Secrets

# Personal Access Token (GitHub PAT)
- Escopo: repo
- Usado para o CI atualizar o projeto-cd

# Docker Hub Token
- Permissões: Read & Write
- Usado para push de imagens

# Secrets no projeto-ci
- DOCKERHUB_USERNAME
- DOCKERHUB_TOKEN
- PAT_TOKEN

<p align="center">
  <img src="midia/secret.png" alt="Secrets" width="700">
</p>

## Enviar o Código Inicial

No seu terminal, vá para a pasta projeto-ci, adicione os arquivos e faça o push:
```
git add .
git commit -m "Adiciona aplicação FastAPI e workflow de CI"
git push origin main
```

Faça o mesmo para o projeto-cd (com os arquivos YAML):
```
git add .
git commit -m "Adiciona manifestos iniciais do Kubernetes"
git push origin main
```
## Configurar o Ambiente Local (Kubernetes e ArgoCD)

# Iniciar o Kubernetes:

Abra o Rancher Desktop e garanta que o Kubernetes esteja rodando.

Verifique a conexão: `kubectl get nodes` (deve mostrar rancher-desktop).

# Instalar o ArgoCD:

Crie o namespace dedicado:

```
 kubectl create namespace argocd
```

Aplique o manifesto de instalação oficial:

```
 kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

```

Aguarde os pods iniciarem: `kubectl get pods -n argocd -w`


## Acessar o painel web do ArgoCD

### Porta-forward 

No terminal (Diferente) :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

* O comando cria um túnel: `localhost:8080` → `argocd-server:443`.
* Mensagens como `Handling connection for 8080` indicam que o túnel está funcionando e atendendo conexões.

Abra no navegador:

```
https://localhost:8080
```

* O navegador exibirá um aviso de certificado (ERR_CERT_AUTHORITY_INVALID). Isso é normal pois o certificado é autoassinado. Clique em “Avançado” → “Continuar para localhost (não seguro)”.

## Fazer login no ArgoCD

A senha inicial do usuário `admin` está armazenada num Secret. No PowerShell (Windows) decodifique assim:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

Login no ArgoCD:

* Usuário: `admin`
* Senha: (saída do comando acima)

<p align="center">
  <img src="midia/Captura de Tela (84).png" alt="Foto da página admin ArgoCD" width="700">
</p>


## Conectar o ArgoCD ao seu repositório

### ArgoCD Application Web

1. No painel do ArgoCD clique **NEW APP**.
2. Preencha:
   * Application Name: `fastapi-app` (tudo em minúsculas)
   * Project: `default`
   * Repository URL: `https://github.com/gianpedrobc/Projeto_CD.git`
   * Revision: `HEAD`
   * Path: `k8s/`
   * Cluster: `https://kubernetes.default.svc`
   * Destination Namespace: `fastapi`
   * Sync Policy: selecionar **Automatic** 
3. Clique em **Create**.

---

# Imagens: 
<h3 align="center">Configuração do ArgoCD</h3>

<p align="center">
  <img src="midia/argocd 1.jpg" alt="Configuração da ArgoCD - Etapa 1" width="700">
</p>
<p align="center">
  <img src="midia/argocd 2.jpg" alt="Configuração da ArgoCD - Etapa 2" width="700">
</p>
<p align="center">
  <img src="midia/argocd 3.jpg" alt="Configuração da ArgoCD - Etapa 3" width="700">
</p>
<p align="center">
  <img src="midia/argocd 4.jpg" alt="Configuração da ArgoCD - Etapa 4" width="700">
</p>
<p align="center">
  <img src="midia/argocd 5.jpg" alt="Configuração da ArgoCD - Etapa 5" width="700">
</p>



## Verificar o Deploy

O ArgoCD irá clonar o projeto-cd, ler a pasta k8s e aplicar os manifestos. O card da fastapi-app deve mudar para Healthy e Synced.

No seu terminal, verifique se os pods estão rodando:

`kubectl get pods`

 Você deve ver 2 pods 'fastapi-deployment-...' em estado 'Running'


---

## 🎥 Demonstração em Vídeo

Veja o funcionamento completo do pipeline CI/CD automatizado com GitHub Actions, Docker Hub e ArgoCD:

[![Assista no YouTube](https://img.youtube.com/vi/92X2tRUg6Qk/0.jpg)](https://www.youtube.com/watch?v=92X2tRUg6Qk)


