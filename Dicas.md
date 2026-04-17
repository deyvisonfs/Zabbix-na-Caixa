## ***Solução de Problemas e Dicas de Manutenção***
Este documento serve como um guia rápido para resolver erros comuns e realizar manutenções preventivas no ambiente de monitoramento Zabbix + Grafana. Se você encontrar comportamentos inesperados durante a instalação ou atualização, consulte os tópicos abaixo.



A alternativa para não precisar sair (Logoff):
Se você estiver no terminal e não quiser fechar e abrir de novo, existe um "truque" para atualizar o grupo na hora:

```Bash
newgrp docker
```
O comando vai forçar o terminal a reconhecer que você agora faz parte do grupo docker sem precisar deslogar da máquina.


Quando você mudar alguma configuração dentro do arquivo zabbix_server.conf direto no Linux (sem Docker), normalmente você vai usar o comando:
```bash
systemctl restart zabbix-server
```

Mas No nosso caso, como estamos usando Docker, se você mudar algo no docker-compose.yml, o comando correto é:
```Bash
docker compose up -d --force-recreate
```

Resumindo: Resumo: Para a permissão do Docker valer, o melhor é o ```newgrp docker``` ou o bom e velho ```exit```. O ```systemctl``` não atua sobre as permissões do seu usuário, apenas sobre o estado do serviço!
