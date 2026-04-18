# Monitoramento-com-Zabbix-Grafana-e-Docker-no-Proxmox
Este projeto demonstra a implementação de uma infraestrutura de monitoramento completa, utilizando Docker Compose sobre uma instância Debian Server virtualizada em um ambiente Proxmox VE.

> **Nota:** Passo a passo completo para a criação de uma infraestrutura de monitoramento orquestrada por containers. Desenvolvido para rodar tanto em servidores Debian puros quanto em ambientes de alta disponibilidade como Proxmox VE.

## ***1. Preparação da VM no Proxmox***

Antes de configurar o software, precisamos preparar o hardware virtual. No seu painel Proxmox, clique em **Create VM** e siga as recomendações:

### ⚙️ Recursos Recomendados:
* **SO:** Debian 12 Server (instalação padrão, sem interface gráfica, apenas "SSH Server")
* **CPU:** 2 vCPUs (Tipo: `host` para melhor performance).
* **RAM:** 4GB (Mínimo recomendado para Zabbix + Grafana + Banco de Dados).
* **Disco:** 32GB (SSD preferencialmente).
* **Rede:** Modo **Bridge** (para que o servidor tenha um IP acessível na sua rede).

## ***2. Passo a Passo no Terminal do Debian***

Após logar no seu Debian via terminal ou console do Proxmox:

A. Instalar o Docker
```Bash
apt update && apt upgrade -y
apt install curl ca-certificates -y # (em caso de pedir a instalação do curl)
curl -fsSL https://get.docker.com | sh
usermod -aG docker $USER
# SAIA DO TERMINAL (EXIT) E ENTRE DE NOVO PARA AS PERMISSÕES VALEREM
```

B. Criar a estrutura de arquivos
```Bash
cd ~
mkdir zabbix-monitor && cd zabbix-monitor
```

C. Criar o arquivo de senhas (.env)
```Bash
nano .env
```

Copie e cole o conteúdo abaixo:
```Bash
# SENHAS DO BANCO DE DADOS
MYSQL_DATABASE=zabbix
MYSQL_USER=zabbix
MYSQL_PASSWORD=zabbix             # Alterado para facilitar
MYSQL_ROOT_PASSWORD=zabbix_lab    # Uma senha mestra diferente é recomendável
DB_SERVER_HOST=zabbix-db

# CONFIGURAÇÃO DO SISTEMA
PHP_TZ=America/Recife             # Coloque sua capital ou normalmente New York o padrão
```

D. Criar o arquivo do servidor (docker-compose.yml)
```Bash
nano docker-compose.yml
```

Copie e cole o código abaixo (Não precisa mudar IPs aqui, o Docker resolve nomes):
```YAML
services:
  zabbix-db:
    image: mariadb:10.11
    container_name: zabbix-db
    env_file: .env
    volumes:
      - ./zabbix_db:/var/lib/mysql
    restart: always

  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-6.4-latest
    container_name: zabbix-server
    ports:
      - "10051:10051"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    env_file: .env
    environment:
      - ZBX_DBHOST=zabbix-db # Nome do container do banco
    depends_on:
      - zabbix-db
    restart: always

  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:ubuntu-6.4-latest
    container_name: zabbix-web
    ports:
      - "80:8080" # Mudei para 80 para você acessar só pelo IP no navegador
    volumes:
      - /etc/localtime:/etc/localtime:ro
    env_file: .env
    environment:
      - ZBX_DBHOST=zabbix-db
      - ZBX_SERVER_HOST=zabbix-server
    depends_on:
      - zabbix-db
    restart: always

  zabbix-agent:
    image: zabbix/zabbix-agent2:latest
    container_name: zabbix-agent-self
    privileged: true
    network_mode: host # IMPORTANTE: permite monitorar o host real onde o Docker está
    environment:
      - ZBX_SERVER_HOST=127.0.0.1
      - ZBX_HOSTNAME=Zabbix-Server-Principal
    restart: always

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana_data:/var/lib/grafana
    restart: always
```

Para subir tudo:
```Bash
docker compose up -d
```


## Como conferir se deu certo?
Digite:

```Bash
docker ps
```
Você deverá ver 5 containers na lista: zabbix-server, zabbix-web, zabbix-db, zabbix-agent-self e grafana.

Resumo da sua estrutura de pastas:
Sua árvore de diretórios vai ficar assim:

```Plaintext
/home/seu-usuario/
└── zabbix-monitor/
    ├── .env                 (Suas senhas e configurações de Timezone)
    ├── docker-compose.yml   (O arquivo com os 5 serviços configurados)
    ├── zabbix_db/           (criada automaticamente para os dados do banco)
    └── grafana_data/        (criada automaticamente para as configs do grafana)
```

## Passo Final no Painel do Zabbix:
Depois de dar o docker ```compose up -d```, você precisa ativar o monitoramento no navegador:

No Zabbix (http://IP:8080):

Vá em Data Collection -> Hosts.

Você verá um host chamado "Zabbix server" que já vem criado por padrão.

Clique nele e mude o Interfaces -> IP address para ```127.0.0.1```.

Certifique-se que o ***Template*** ```Linux by Zabbix agent``` está selecionado.

Clique em Update.



## ***3. Integração Passo a Passo (Zabbix + Grafana)***

Agora, com tudo rodando, vamos fazer os dois conversarem:

No Zabbix (http://IP:8080):

Login: Admin | Senha: zabbix

Vá em Administration -> Users -> Create User.

Usuário: grafana_user | Grupo: No access to the frontend | Senha: sua_senha.

Em Permissions, dê a ele papel de Admin Role (para poder ler a API).

No Grafana (http://IP:3000):

Login: admin | Senha: admin (ele pedirá para trocar).

Menu lateral -> Plugins -> Procure por Zabbix -> Clique em Install e depois em Enable.

Menu lateral -> Connections -> Data Sources -> Add new data source -> Zabbix.

Configuração da URL: http://zabbix-web:8080/api_jsonrpc.php (O Grafana usa o nome do container do Docker).

Zabbix API Details: Coloque o usuário (grafana_user) e a senha que você criou no passo anterior.

Clique em Save & Test.


## ***4. Configurando o Zabbix Agent (Em outra máquina/VM)***

Para monitorar outra máquina, você cria um arquivo separado nela:
```Bash
mkdir zabbix-agent && cd zabbix-agent
nano docker-compose.yml
# Código do Agent (Atenção aos comentários):
```
Código do Agent (Atenção aos comentários):
```YAML
services:
  zabbix-agent:
    image: zabbix/zabbix-agent2:latest
    container_name: zabbix-agent
    ports:
      - "10050:10050"
    privileged: true
    environment:
      # COLOQUE O IP DO SEU DEBIAN SERVER ABAIXO
      - ZBX_SERVER_HOST=192.168.1.100  <-- MUDAR PARA O IP DO ZABBIX SERVER
      # NOME QUE VAI APARECER NO PAINEL DO ZABBIX
      - ZBX_HOSTNAME=Servidor-Proxmox-01
    restart: always
```


## ***5.Como integrar o Agent no Painel do Zabbix Server***

Após subir o container do Agent, siga estes passos no navegador:

Acesse o Zabbix Web: Vá em Data Collection -> Hosts.

Criar Novo Host: Clique no botão Create host (canto superior direito).

Configurações Principais:

Host name: Coloque exatamente o mesmo nome que você definiu no arquivo do Docker (ZBX_HOSTNAME). Ex: Servidor-Proxmox-01.

Templates: Procure e selecione Linux by Zabbix agent.

Host groups: Escolha um grupo (ex: Linux servers).

Interfaces:

Clique em Add -> Agent.

IP address: Coloque o IP da máquina onde o Agent está rodando.

Port: 10050 (padrão).

Finalizar: Clique em Add.
