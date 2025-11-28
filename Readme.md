# Instruções gerais para usar o Chatbot

Esse documento contém as instruções gerais para rodar o sistema completo do chatbot.

## Backend

O backend que atua como um middleware entre o aplicativo mobile e o servidor LLM foi feito em Go usando o framework Gin para a API.

Ele está no repositório abaixo:
<https://github.com/vitorpereira26r/chat-bot-backend>

No readme desse repositório tem as instruções para rodar o backend localmente.

### Fazer deploy do backend (Digital ocean)

Esse passo a passo funciona com algumas modificações em quase todos os servidores.

Primeiro, acessar o painel do Digital Ocean.

Criar um Droplet: Ubuntu 22.04 (Sugestão). 1 GB RAM já é o suficiente (plano mais barato).

Após criar o Droplet, usando o endereço IP desse Droplet, acessa-lo via SSH:

``` bash
ssh root@IP_DROPLET
```

Ao acessar, criar um usuário não `root`:

``` bash
adduser vitor
usermod -aG sudo vitor
```

(Ou outro nome)

Instalar o git:

``` bash
sudo apt update
sudo apt install git -y
```

Clonar o repositório do backend:

``` bash
git clone https://github.com/vitorpereira26r/chat-bot-backend.git
```

(Ou link do fork).

Acessar pasta do backend:

``` bash
cd chat-bot-backend
```

#### PostgreSQL

Criar uma pasta para o banco de dados:

``` bash
sudo mkdir postgresql
cd postgresql
```

Criar um arquivo `.env` com alguns dados para o banco de dados:

``` env
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_DB=postgres
```

Criar um `docker-compose.yml`:

``` yml
services:
  postgres:
    image: postgres:16
    container_name: postgres16
    restart: unless-stopped
    env_file: ./.env
    volumes:
      - pgdata:/var/lib/postgresql/data
    command:
      - "postgres"
      - "-c"
      - "listen_addresses=*"
    networks:
      pgnet:
        ipv4_address: 172.31.0.10

volumes:
  pgdata:

networks:
  pgnet:
    external: true
```

Instalar Docker e docker-compose:

``` bash
sudo apt install docker.io
sudo apt install docker-compose
```

Subir container do postgres:

``` bash
sudo docker-compose up -d
sudo docker-compose logs -f postgres
sudo docker ps
```

#### Rodar backend

Acessar pasta do backend:

```bash
cd chat-bot-backend
```

Criar um `.env`:

``` bash
sudo nano .env
```

Conteúdo do `.env` (exemplo), não mudar o `CORS_ORIGIN`:

``` env
PORT=8080
API_KEY=API_KEY_EXEMPLE
JWT_SECRET=JWT_SECRET_EXEMPLE
DATABASE_DSN=postgres://user:password@localhost:5432/postgres?sslmode=disable&TimeZone=America/Sao_Paulo
CORS_ORIGINS=http://localhost:8081,http://vprcloud.com.br
```

Usar no `DATABASE_DSN`, o user, a senha e a database setados no banco postgres. O `API_KEY` e `JWT_SECRET`, podem ser gerados ou usar qualquer coisa.

Instalar o Go e dependências:

``` bash
sudo apt install golang-go -y
go mod download
```

Executar:

``` bash
go run .
```

#### Criar serviço para o backend (Opcional)

Seguir os passos abaixo

```bash
sudo nano /etc/systemd/system/chatbot-backend@backend.service
```

Conteúdo do arquivo:

``` bash
[Unit]
Description=Chatbot Backend - %i
After=network.target

[Service]
Type=simple
User=%i
WorkingDirectory=/home/%i/chat-bot-backend
EnvironmentFile=/home/%i/chat-bot-backend/.env
ExecStart=/usr/bin/go run .
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Ativar serviço:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable chatbot-backend@vitor
sudo systemctl start chatbot-backend@vitor
sudo systemctl status chatbot-backend@vitor
```

##### Atualizar backend no futuro usando o serviço

``` bash
cd ~/chat-bot-backend
git pull
sudo systemctl restart chatbot-backend@vitor
```

#### Acessar backend

Acessar o backend pelo URL:

``` url
http://IP_DROPLET:8080/
```

## Deploy de modelo AWS (ou outro servidor)

Para fazer o deploy de um modelo Ollama em um servidor (usei AWS como exemplo), o passo a passo está abaixo.

Documentação Ollama:
<https://ollama.com/>

### Deploy

Open AWS.

Open EC2 console.

Click "Launch instance"

Put the name one "Ollama server".

``` text
Ollama server
```

Click "Browse more AMIs".

Search for "Deep learning".

``` text
Deep learning
```

Select "Deep learning OSS Nvidia Driver AMI GPU  TensorFlow 2.18 (Ubuntu 22.04)"

Select instance type "g4ad.xlarge"

Select key pair "ollama-model"

Allow "HTTPS" and "HTTP" traffic.

Click to "Launch instance".

(Wait instance to be launched)

Click "Connect to instance".

Open in my PC the directory "C:/Users/USER/ChatBotApp/deploy-aws" in the terminal.

``` bash
C:/Users/USER/ChatBotApp/deploy-aws
```

Copy the ssh command to open the instance in the AWS page.

Run the command to update server:

``` bash
sudo apt update && sudo apt upgrade -y
```

Run the command to install go:

``` bash
sudo apt install -y golang-go
```

Run the command to install ollama:

``` bash
curl -fsSL https://ollama.com/install.sh | sh
```

Run the command to clone the repository:

``` bash
https://github.com/vitorpereira26r/aws-ec2-cuda-ollama
```

``` bash
cd aws-ec2-cuda-ollama
```

Run the command:

``` bash
ollama serve
```

Run the command to run the LLM:

``` bash
ollama pull gemma2:2b
```

Run the command to run the go server:

``` bash
go run main.go
```

Go to the AWS instance.

Click the "Security" tab.

Click the security group in it.

Click "Edit inbound rules".

Click "Add rule".

Leave "Custom TCP", Port range "8080", and "0.0.0.0/0".

Save rules.

Get the instance url.

### Registrar modelo no sistema

Para registrar o modelo no sistema e usar no app, usar o endpoint:

``` bash
curl --location 'https://apigo.vprcloud.com.br/api/models' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjaGF0LWJvdC1iYWNrZW5kIiwic3ViIjoiY2E2YTZlMTItYjg1Yi00YjY2LWE2YmItNDgzYjNmNjQ0NDQyIiwiZXhwIjoxNzYzNTg0Mjc0LCJpYXQiOjE3NjM0OTc4NzR9.W5TN_JlrmbrTSrnHC6KJFrIOmk0Iby8BxQSLwyCPJbo' \
--data '{
  "name": "gemma2:2b",
  "api_url": "http://ec2-3-87-245-195.compute-1.amazonaws.com:8080/v1/chat/completions"
}
'
```

(Usar a url do backend)

No campo `api_url`, colocar a url com o ip ou endereço de onde está o servidor Ollama e a porta que ele está usando (o servidor de forward). O path do url é `/v1/chat/completions` (deixar assim).

#### Editar modelo no sistema

Se quiser, editar o modelo no endpoint:

``` bash
curl --location --request PATCH 'https://apigo.vprcloud.com.br/api/models/gemma' \
--data '{
  "name": "gemma2:2b",
  "api_url": "http://ec2-3-87-245-195.compute-1.amazonaws.com:8080/v1/chat/completions"
}
'
```

## Aplicativo mobile

Para rodar o aplicativo, seguir as instruções do Readme.md do repositório abaixo:
<https://github.com/vitorpereira26r/chat-bot-mobile-app>
