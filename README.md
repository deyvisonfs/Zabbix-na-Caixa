# Monitoramento-com-Zabbix-Grafana-e-Docker-no-Proxmox
Este projeto demonstra a implementação de uma infraestrutura de monitoramento completa, utilizando Docker Compose sobre uma instância Debian Server virtualizada em um ambiente Proxmox VE.

Zabbix de Pregioçoso!

1. Preparação da VM no Proxmox
Antes do Docker, você precisa da "casa" (a VM).

No seu Proxmox, clique em Create VM.

Recursos Recomendados:

CPU: 2 vCPUs (Tipo: 'host' para melhor performance).

RAM: 4GB (Zabbix + Grafana + DB consomem memória).

Disco: 32GB (SSD preferencialmente).

Rede: Modo Bridge (para a VM ter um IP próprio na sua rede).

Instale o Debian 12 Server (instalação padrão, sem interface gráfica, apenas "SSH Server" e "Standard System Utilities").

2. Passo a Passo no Terminal do Debian
Após logar no seu Debian via terminal ou console do Proxmox:

A. Instalar o Docker
```Bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
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
MYSQL_PASSWORD=senha_do_usuario_zabbix
MYSQL_ROOT_PASSWORD=senha_mestra_banco

# CONFIGURAÇÃO DO SISTEMA
PHP_TZ=America/Recife
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
    env_file: .env
    depends_on:
      - zabbix-db
    restart: always

  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:ubuntu-6.4-latest
    container_name: zabbix-web
    ports:
      - "8080:8080" # Porta para acessar o Zabbix no navegador
    env_file: .env
    depends_on:
      - zabbix-db
    restart: always

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "3000:3000" # Porta para acessar o Grafana no navegador
    volumes:
      - ./grafana_data:/var/lib/grafana
    restart: always
```

Para subir tudo: ```docker compose up -d```

3. Configurando o Zabbix Agent (Em outra máquina/VM)
Para monitorar outra máquina, você cria um arquivo separado nela:
```Bash
mkdir zabbix-agent && cd zabbix-agent
nano docker-compose.yml
Código do Agent (Atenção aos comentários):
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

4. Integração Passo a Passo (Zabbix + Grafana)
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
