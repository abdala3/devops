# DevOps

## Visão Geral
1. Gerar chave SSH para deploy
2. Preparar os servidores (ou ambientes remotos)
3. Configurar repositório no GitHub
4. Criar workflows no GitHub Actions
5. Testar o deploy para cada ambiente

### Geração de chaves SSH
1. Ambiente de desenvolvimento:
  - ``` ssh-keygen -t ed25519 -C "github-actions@dev" ```
  - salvar em **/home/seu_usuario/.ssh/id_dev_env**:
  - **IMPORTANTE: NÃO DIGITAR SENHA QUANDO SOLICITADO!**
  - será gerado o par de chaves:
    - privada: **id_dev_env** (vai para o GitHub)
    - pública: **id_dev_env.pub** (vai para a instância/servidor) 
2. Ambiente de homologação:
  - ``` ssh-keygen -t ed25519 -C "github-actions@homolog" ```
  - salvar em **/home/seu_usuario/.ssh/id_homolog_env**
  - será gerado o par de chaves:
    - privada: **id_homolog_env** (vai para o GitHub)
    - pública: **id_homolog_env.pub** (vai para a instância/servidor) 
3. Ambiente de produção:
  - ``` ssh-keygen -t ed25519 -C "github-actions@prod" ```
  - salvar em **/home/seu_usuario/.ssh/id_prod_env**
  - será gerado o par de chaves:
    - privada: **id_prod_env** (vai para o GitHub)
    - pública: **id_prod_env.pub** (vai para a instância/servidor) 

### Criação da instância na AWS
1. Acessar a plataforma AWS: ```https://us-east-2.console.aws.amazon.com/console/home?region=us-east-2```
2. Acessar o produto **EC2 > Instances > Launch an instance > Application and OS Images (Amazon Machine Image) > Quick Start > Amazon Linux (AWS) - Amazon Machine Image (AMI)**
3. Launch Instance
4. Instância criada:
<img width="1307" height="278" alt="image" src="https://github.com/user-attachments/assets/98c6cd99-e383-426d-a5c4-f0f72c6d602a" />

5. Clicar no **ID da Instância** para consultar o **IP Público ou o DNS Público** da instância:
<img width="1304" height="499" alt="image" src="https://github.com/user-attachments/assets/071a9601-2fd8-4e2d-9271-0467299d24fc" />

6. Teste de conexão ao servidor remoto via **SSH** (sem necessidade de autenticação com senha: ```ssh -i ~/.ssh/id_dev_env ec2-user@ec2-18-222-56-125.us-east-2.compute.amazonaws.com```

### Configurar seus servidores
1. Conexão à instância criada (usuário padrão é o **ec2-user**): ``` ssh -i ~/.ssh/id_dev_env [ec2-user]@[IP Público da Instância] ```
   - Se for recebida a mensagem: ***The authenticity of host '18.220.175.103 (18.220.175.103)' can't be established.***, digitar ***yes***
   - Caso não funcione, utilize a chave **.pem** gerada no momento da criação da instância: ``` ssh -i [caminho/da/chave].pem ec2-user@18.220.175.103 ```
     - Se for recebida a mensagem: ***The authenticity of host '18.220.175.103 (18.220.175.103)' can't be established.***
       - ***Are you sure you want to continue connecting (yes/no/[fingerprint])?***, escolher ***yes***
         <img width="819" height="362" alt="image" src="https://github.com/user-attachments/assets/015ea923-5908-4aa0-ae99-9e5b6493919b" />
   - Copiar a chave pública - gerada na máquina local - para o servidor remoto: ``` ssh-copy-id -i ~/.ssh/id_dev_env.pub [ec2-user]@[IP Público] ```
     - Caso não exista o comando ***ssh-copy-id*** na máquina:
       - copie a chave pública via ***cat*** e
       - cole na instância: ```nano ~/.ssh/authorized_keys```
     - Altere as permissões do diretório e do arquivo de chaves autorizadas no servidor:
       - chmod 700 ~/.ssh
       - chmod 600 ~/.ssh/authorized_keys
2. Criar o diretório do projeto web: ```mkdir -p /var/www/dev-site```
3. Alterar o owner do diretório e arquivo:  ```sudo chown ec2-user:ec2-user /var/www/dev-site```

### Adicionar a chave privada ao GitHub
1. Consultar a chave privada no computador local: ``` cat ~/.ssh/id_dev_env ```
2. Copiar todo o conteúdo apresentado

##### Criar o segredo no GitHub
1. Acessar o repositório no Github
2. Clicar em **Settings**
3. No menu lateral, acesse **Secrets and Variables > Actions**:
   - **Nome**: [DEV_SSH_KEY]
   - **Secret**: [colar o conteúdo da chave privada]
4. Repita o processo para **Homologação** e **Produção**

### Gerar o workflow para o Github Actions (arquivo .yaml)
1. Criação de um projeto simples: **HTML** e **CSS**
   - um projeto só com **HTML** e **CSS** não precisa de build (npm run build, Webpack, etc.), então o **CI/CD** com **GitHub Actions** é direto e simplificado
     
#### Passos
1. Criar projeto **HTML**
2. Subir para o **GitHub**
3. **Workflow** no GitHub
4. **Deploy** automático

##### Criação do projeto
1. Criar o diretório do projeto: ```mkdir meu-site```
2. Crie o arquivo **index.html** dentro do diretório:
   ```
    <!DOCTYPE html>
    <html lang="pt-br">
    <head>
      <meta charset="UTF-8">
      <title>Meu Site</title>
      <link rel="stylesheet" href="style.css">
    </head>
    <body>
      <h1>Olá, mundo!</h1>
      <p>Este é meu primeiro site com deploy automático.</p>
    </body>
    </html>
   ```
3. Crie o arquivo **style.css**:
   ```
    body {
      font-family: sans-serif;
      background-color: #f9f9f9;
      color: #333;
      text-align: center;
      padding-top: 50px;
    }
   ```
###### Criação do repositório no GitHub
```
git init
git remote add origin git@github.com:seu-usuario/seu-repositorio.git
git add .
git commit -m "Primeira versão do site"
git branch -M dev
git push -u origin dev
```
2. Criar o diretório do **workflow de deploy** com **GitHub Actions**: ``` mkdir -p .github/workflows ```
3. Crie o arquivo: **.github/workflows/deploy-dev.yml**
```
name: Deploy HTML/CSS to EC2 - Development

on:
  push:
    branches:
      - dev

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEV_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H 54.123.45.67 >> ~/.ssh/known_hosts

      - name: Deploy via SCP
        run: |
          echo "Deploying HTML/CSS files..."
          scp -i ~/.ssh/id_rsa -r ./index.html ./style.css ec2-user@54.123.45.67:/var/www/dev-site
```
4. Finalizada a etapa anterior, faça um ***push*** forçado no projeto para testar as etapas de **CI/CD**.
   - Commitar sem a necessidade de alteração em algum arquivo: ```git commit --allow-empty -m "força execução do pipeline após correção de segredo"``
   - Ativar o deploy: ```git push```

###### Passos configurados (Actions)
1. Checkout repo
2. Set up SSH
3. Deploy via SCP



