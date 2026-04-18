## ***Solução de Problemas e Dicas de Manutenção***
Este documento serve como um guia rápido para resolver erros comuns e realizar manutenções preventivas no ambiente de monitoramento Zabbix + Grafana. Se você encontrar comportamentos inesperados durante a instalação ou atualização, consulte os tópicos abaixo.

# o Debian, por padrão, o comando sudo pode não vir instalado ou o seu usuário pode não estar na lista de "administradores" (sudoers).

e você estiver recebendo erros de "Permissão negada" ou "comando não encontrado", o jeito mais fácil de resolver durante a instalação é virar o superusuário root:

No terminal, digite:
```Bash
su -
```
E coloque a senha do root (aquela que você definiu na instalação da VM).



# A alternativa para não precisar sair (Logoff):

Se você estiver no terminal e não quiser fechar e abrir de novo, existe um macete para atualizar o grupo na hora:

```Bash
newgrp docker
```
O comando vai forçar o terminal a reconhecer que você agora faz parte do grupo docker sem precisar deslogar da máquina.


Quando você mudar alguma configuração dentro do ```arquivo zabbix_server.conf``` direto no Linux (sem Docker), normalmente você vai usar o comando:
```bash
systemctl restart zabbix-server
```

Mas No nosso caso, como estamos usando Docker, se você mudar algo no ```docker-compose.yml```, o comando correto é:
```Bash
docker compose up -d --force-recreate
```

Resumindo: Resumo: Para a permissão do Docker valer, o melhor é o ```newgrp docker``` ou o bom e velho ```exit```. O ```systemctl``` não atua sobre as permissões do seu usuário, apenas sobre o estado do serviço!


# Se os containers do Grafana ou do Banco de Dados não subirem por erro de "Permission Denied" nos logs, execute os comandos abaixo na pasta raiz do projeto:


Define o dono da pasta do Grafana para o usuário padrão do container (ID 1000)
```Bash
sudo chown -R 1000:1000 grafana_data 
```
Garante permissão total de leitura e escrita para as pastas de persistência
```Bash
sudo chmod -R 777 zabbix_db grafana_data
```

# Para que o agente consiga ler os dados dos containers e do sistema (corrigindo o erro de Agent not available), é necessário liberar o socket do Docker:
Bash

Dá permissão de leitura e escrita no socket do Docker dentro do container
```Bash
docker exec -u 0 -it zabbix-agent-self chmod 666 /var/run/docker.sock
```


**Integração Zabbix x Grafana (Usuário de API)**

No Frontend do Zabbix (http://seu-ip), crie o usuário para a integração:
```
    Username: grafana_user

    Group: No access to the frontend (ou Zabbix administrators)

    Permissions: Defina o papel como Admin Role.
```
**Ajuste de Rede Docker (DNS Resolver)** # imagem de como deve ficar

Caso o Zabbix Server não encontre o agente (erro de cannot resolve), certifique-se de configurar o Host no Zabbix Web desta forma:
```
    Interface: Agent

    Connect to: Selecione DNS

    DNS name: zabbix-agent-self

    Port: 10050
```
<img width="1361" height="563" alt="DNS" src="https://github.com/user-attachments/assets/1ffcaa65-709d-4704-b287-4ed589f7a5b1" />


# Firewall do Debian

Se o navegador não abrir a página, garanta que as portas estejam liberadas no sistema operacional:
```Bash
ufw allow 80/tcp    # Porta do Zabbix Web
ufw allow 3000/tcp  # Porta do Grafana
ufw allow 10051/tcp # Porta do Zabbix Server
```


⚠️ Importante: Permissão de Grupo no Zabbix
Não basta o utilizador estar ativo. O User Group associado ao utilizador da API deve ter o Frontend access configurado como System default ou Internal. Se estiver como Disabled, o Grafana não conseguirá autenticar, mesmo com a senha correta.
