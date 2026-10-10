# Kangaroo Tech - infraestrutura Docker

Infraestrutura reproduzivel para publicar os tres projetos em qualquer servidor
Linux com Docker. Cada projeto possui seu proprio PostgreSQL e todos compartilham
um unico Nginx reverso com certificados Let's Encrypt.

## Dominios

- `https://catalogo.kangaroo-tech.net`
- `https://loja.kangaroo-tech.net`
- `https://portfolio.kangaroo-tech.net`

## Arquitetura

- `infra`: Nginx central, Certbot e rede `projects-deploy-proxy`.
- `catalog`: API, frontend, worker e PostgreSQL com pgvector.
- `store`: API, frontend e PostgreSQL.
- `portifolio`: API, frontend e PostgreSQL.
- Somente o Nginx publica as portas `80` e `443`.
- Bancos e servicos internos nao publicam portas no host.
- Dados persistem nos volumes `catalog-postgres-data`, `store-postgres-data`
  e `portfolio-postgres-data`.

## Requisitos recomendados

- Linux com Docker Engine e Docker Compose v2.
- Arquitetura `x86_64`, compativel com as imagens publicadas.
- 2 vCPUs, 4 GB de RAM e 30 GB de disco como ponto de partida.
- TCP `80` e `443` publicas; SSH restrito ao IP do administrador.
- Tres registros DNS `A` apontando para o IP publico do servidor.

Com apenas 1 GB de RAM, suba um projeto por vez ou crie swap. Os tres projetos
simultaneos podem sofrer encerramentos por falta de memoria.

## Segredos e arquivos .env

Os `.env` reais sao ignorados pelo Git, mesmo em repositorio privado:

```sh
cp infra/.env.example infra/.env
cp catalog/.env.example catalog/.env
cp store/.env.example store/.env
cp portifolio/.env.example portifolio/.env
```

Substitua todos os valores `change-me`. Gere segredos fortes com:

```sh
openssl rand -base64 48
```

Nunca versione senhas, chaves JWT, chave Groq, chaves de pagamento ou
`infra/certbot/conf`.

## Primeira instalacao

Execute a partir da raiz. Primeiro crie a rede sem iniciar o Nginx, pois os
upstreams ainda nao existem:

```sh
docker compose -f infra/docker-compose.yml create nginx
```

Suba os projetos:

```sh
docker compose -f portifolio/docker-compose.yml pull
docker compose -f portifolio/docker-compose.yml up -d
docker compose -f catalog/docker-compose.yml pull
docker compose -f catalog/docker-compose.yml up -d
docker compose -f store/docker-compose.yml pull
docker compose -f store/docker-compose.yml up -d
```

Rode o seed:
```sh
docker exec -it 07153296893e node seed-dist/prisma/seed.js --count=50
```

Suba o Nginx HTTP para o desafio ACME:

```sh
docker compose -f infra/docker-compose.yml up -d nginx
curl http://127.0.0.1/health
```

A resposta esperada e `ok`.

## Primeiro certificado SSL

Valide o webroot antes de chamar o Certbot:

```sh
mkdir -p infra/certbot/www/.well-known/acme-challenge
printf 'acme-ok\n' > infra/certbot/www/.well-known/acme-challenge/teste
curl http://catalogo.kangaroo-tech.net/.well-known/acme-challenge/teste
curl http://loja.kangaroo-tech.net/.well-known/acme-challenge/teste
curl http://portfolio.kangaroo-tech.net/.well-known/acme-challenge/teste
```

Os tres comandos precisam retornar `acme-ok`. Depois emita um certificado SAN:

```sh
docker compose -f infra/docker-compose.yml run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  --email vaguiiinho5@gmail.com \
  --agree-tos \
  --no-eff-email \
  -d catalogo.kangaroo-tech.net \
  -d loja.kangaroo-tech.net \
  -d portfolio.kangaroo-tech.net
```

Somente depois de `Successfully received certificate`, ative HTTPS:

```sh
docker compose \
  -f infra/docker-compose.yml \
  -f infra/docker-compose.ssl.yml \
  up -d --force-recreate nginx certbot-renew
```

Valide:

```sh
docker exec projects-nginx nginx -t
curl -I https://catalogo.kangaroo-tech.net
curl -I https://loja.kangaroo-tech.net
curl -I https://portfolio.kangaroo-tech.net
```

HTTP deve responder `301` e redirecionar para HTTPS.

## Renovacao dos certificados

`certbot-renew` verifica a renovacao a cada 12 horas. Recarregue o Nginx
diariamente adicionando com `crontab -e`:

```cron
30 4 * * * /usr/bin/docker exec projects-nginx nginx -s reload >/dev/null 2>&1
```

Teste a renovacao:

```sh
docker compose -f infra/docker-compose.yml run --rm certbot renew --dry-run
```

## Atualizacao

```sh
git pull
docker compose -f portifolio/docker-compose.yml pull
docker compose -f portifolio/docker-compose.yml up -d
docker compose -f catalog/docker-compose.yml pull
docker compose -f catalog/docker-compose.yml up -d
docker compose -f store/docker-compose.yml pull
docker compose -f store/docker-compose.yml up -d
docker compose -f infra/docker-compose.yml -f infra/docker-compose.ssl.yml up -d
```

## Operacao

```sh
docker ps
docker stats --no-stream
df -h /
free -h
docker logs --tail 100 projects-nginx
```

Parar sem apagar o banco:

```sh
docker compose -f catalog/docker-compose.yml down
```

Nao use `down -v` nem `docker volume prune` sem backup confirmado.

## Backup dos bancos

```sh
mkdir -p backups
docker exec portfolio-postgres pg_dump -U portfolio -d portfolio -Fc > backups/portfolio.dump
docker exec catalog-postgres pg_dump -U catalog -d product_catalog -Fc > backups/catalog.dump
docker exec store-postgres pg_dump -U store -d store -Fc > backups/store.dump
```

Copie os dumps para outro equipamento ou armazenamento. Volumes Docker nao
substituem backups. Antes de usar `pg_restore`, pare as APIs e confirme o banco
de destino, pois a restauracao pode sobrescrever dados.

## Migracao para outro provedor

1. Prepare Linux com Docker, 4 GB de RAM e disco suficiente.
2. Clone este repositorio privado.
3. Crie os `.env` pelos modelos e restaure os segredos com seguranca.
4. Copie e restaure os dumps dos tres bancos.
5. Atualize os DNS para o IP novo.
6. Libere TCP `80` e `443` nos firewalls do provedor e do sistema.
7. Execute a primeira instalacao e emita novos certificados.
8. Valide HTTPS antes de desligar o servidor anterior.

Os certificados nao sao versionados. Em migracoes, reemita-os no servidor novo
depois que o DNS apontar para ele.

## Solucao de problemas

- `no space left on device`: execute `df -h /` e `docker system df`;
  expanda o disco, sem remover volumes PostgreSQL.
- Nginx reiniciando: execute `docker logs --tail 100 projects-nginx` e
  `docker exec projects-nginx nginx -t`.
- ACME 404: confirme `server_name`, a montagem `/var/www/certbot` e o teste
  `acme-ok`; nao repita o Certbot enquanto o teste falhar.
- HTTPS local funciona, mas externo expira: confira `443:443`, firewall e
  Security Group.
- Se `default.conf` virar diretorio, restaure
  `infra/nginx/default.conf` pelo Git e recrie o Nginx.

## Custos e desligamento

Alertas de orcamento nao bloqueiam gastos imediatamente. Ao terminar um teste,
pare a instancia no provedor. O armazenamento persistente pode continuar sendo
cobrado com a maquina parada.
