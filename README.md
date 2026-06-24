# Docker Swarm Stack para WordPress (Produção)

Este repositório contém uma infraestrutura profissional e parametrizada em Docker Swarm para hospedar sites WordPress com alta performance, segurança reforçada e rotinas automáticas de backup.

---

## 🚀 Arquitetura e Componentes

1. **Traefik v3 (Reverse Proxy & SSL):**
   - Gerencia o tráfego de entrada e faz o roteamento inteligente dos subdomínios.
   - Gera e renova automaticamente os certificados SSL gratuitos (Let's Encrypt) via desafio HTTP.
   - Opera isolado em um arquivo de stack próprio (`traefik-stack.yml`).

2. **Nginx (Front-End & Caching):**
   - Configurado com cache rápido (`fastcgi_cache`) para acelerar as respostas do PHP-FPM.
   - Cabeçalhos de segurança HTTP ativados (proteção contra Clickjacking, XSS e MIME sniffing).
   - Configuração de domínio catch-all (`server_name _;`), permitindo que a stack seja duplicada sem alterar as configurações internas do Nginx.

3. **WordPress (PHP 8.3 FPM & Alpine):**
   - Executa a última versão do WordPress sobre o PHP-FPM Alpine, garantindo uma imagem leve e segura.
   - Integrado com **OPcache** pré-configurado no `php.ini` para máxima performance de processamento PHP.
   - Limite de upload aumentado para **64MB** (sincronizado no Nginx e PHP).

4. **MariaDB (Banco de Dados):**
   - Banco de dados relacional robusto.
   - Configurado com suporte nativo a `utf8mb4_unicode_ci` e limite máximo de pacotes aumentado.

5. **Redis Cache (Object Cache):**
   - Cache em memória RAM para as consultas de banco de dados do WordPress.
   - Reduz drasticamente as consultas ao banco de dados e acelera o tempo de carregamento das páginas.

6. **Backup Automático (`db_backup`):**
   - Container sidecar (`fradelg/mysql-backup`) que gera dumps compactados (`.sql.gz`) do banco de dados a cada 24 horas (às 03:00).
   - Mantém um histórico rotativo de backups dos últimos **7 dias**, limpando automaticamente arquivos antigos.

---

## 🛠️ Pré-requisitos no Servidor (VPS)

Antes de rodar a stack, certifique-se de que o Docker Swarm está inicializado no seu servidor:
```bash
docker swarm init
```

### 1. Criar as Redes Externas
Crie as redes overlay necessárias para o roteamento do Traefik e a comunicação interna dos containers:
```bash
docker network create --driver overlay --scope swarm proxy
docker network create --driver overlay --scope swarm nw-backend
```

### 2. Criar Diretórios de Persistência no Host
Crie as pastas físicas no disco da sua VPS para armazenar os dados e os certificados do SSL:
```bash
# Pasta do SSL do Traefik
mkdir -p /data/letsencrypt_data

# Substitua "takadatintas" pelo valor de PROJECT_NAME do seu .env
mkdir -p /data/wp_data_takadatintas
mkdir -p /data/mysql_data_takadatintas
mkdir -p /data/backups_takadatintas

# Crie a pasta do conteúdo wp-content no host
# Substitua "takadatintas.com.br" pelo valor de DOMAIN_NAME do seu .env
mkdir -p /srv/sites/takadatintas.com.br/wp-content
```

---

## ⚙️ Configuração das Variáveis de Ambiente (`.env`)

Crie ou edite o arquivo `.env` na raiz do projeto. Ele será responsável por parametrizar toda a stack sem que você precise mexer em arquivos `.yml` ou configurações do Nginx:

```env
# Domínio principal do site
DOMAIN_NAME=takadatintas.com.br

# Nome identificador do projeto (usado para isolar volumes e serviços)
PROJECT_NAME=takadatintas

# Credenciais de acesso ao Banco de Dados
MYSQL_USER=taka_takadatintas
MYSQL_PASSWORD=sua_senha_segura
MYSQL_ROOT_PASSWORD=sua_senha_root_segura
MYSQL_DATABASE=takadatintas.sql
WORDPRESS_TABLE_PREFIX=wpog_
```

---

## 📥 Importação Automática de Banco de Dados Existente

Se você já possui um backup de banco de dados e deseja importá-lo automaticamente ao subir a stack pela primeira vez:

1. Coloque o seu arquivo de backup `.sql` ou `.sql.gz` na VPS no seguinte caminho:
   ```bash
   # Exemplo: /srv/sites/takadatintas.com.br/takadatintas.sql
   /srv/sites/${DOMAIN_NAME}/${BACKUP_DATABASE}
   ```
2. Quando a stack for iniciada pela primeira vez (e a pasta `/data/mysql_data_...` estiver vazia), o MariaDB importará este arquivo de banco automaticamente.

> ⚠️ **IMPORTANTE (Erro de Permissão / Permission Denied):**
> O banco de dados MariaDB/MySQL executa com um usuário interno (`mysql`) sem privilégios de administrador. Você **deve** liberar a permissão de leitura nos arquivos e de execução nas pastas de origem na sua VPS, caso contrário o container falhará ao iniciar com o erro `Permission denied`.
> 
> Execute os comandos abaixo no terminal da sua VPS para ajustar as permissões:
> ```bash
> # Substitua "takadatintas.com.br" e "takadatintas.sql" pelos valores do seu .env
> sudo chmod 755 /srv/sites
> sudo chmod 755 /srv/sites/takadatintas.com.br
> sudo chmod 644 /srv/sites/takadatintas.com.br/takadatintas.sql
> 
> # Se tiver problemas de permissão com o wp-config.php ou pasta wp-content:
> sudo chmod 644 /srv/sites/takadatintas.com.br/wp-config.php
> sudo chmod -R 755 /srv/sites/takadatintas.com.br/wp-content
> ```

---

## 🚀 Como Fazer o Deploy na VPS

1. Clone o repositório na sua VPS (ou dê um `git pull` se já o tiver clonado).
2. Certifique-se de que os passos acima (Diretórios, Redes, `.env` e backup se houver) foram concluídos.

### Passo 1: Subir o Traefik (Executar uma vez)
```bash
docker stack deploy -c traefik-stack.yml traefik
```

### Passo 2: Subir a Stack WordPress + Nginx + Redis + Backup

- **No Linux (Terminal SSH / Bash):**
  ```bash
  export $(grep -v '^#' .env | xargs)
  docker stack deploy -c wordpress-stack.yml ${PROJECT_NAME}_wordpress
  ```

- **No Windows (PowerShell - se estiver testando localmente):**
  ```powershell
  Get-Content .env | Foreach-Object { if ($_ -notmatch '^#') { $name, $value = $_.split('=', 2); [System.Environment]::SetEnvironmentVariable($name, $value, "Process") } }
  docker stack deploy -c wordpress-stack.yml ${env:PROJECT_NAME}_wordpress
  ```

---

## ⚡ Pós-Instalação: Ativando o Cache Redis no WordPress

Após a stack subir e você realizar a instalação do WordPress pelo navegador:

1. Acesse o painel administrativo do WordPress (`/wp-admin`).
2. Instale e ative o plugin **Redis Object Cache**.
3. Vá em **Configurações** > **Redis** e clique em **Enable Object Cache** (Ativar Cache de Objeto).
4. Pronto! O WordPress começará a salvar as consultas de banco de dados no container do Redis.

---

## 🛠️ Comandos Úteis de Gerenciamento

**Verificar serviços em execução:**
```bash
docker service ls
```

**Ver logs de um serviço específico:**
```bash
# Exemplo para a stack "takadatintas"
docker service logs -f takadatintas_wordpress_nginx
docker service logs -f takadatintas_wordpress_wordpress
docker service logs -f takadatintas_wordpress_mysql
```

**Escalar o WordPress e o Nginx para múltiplos containers (Alta Disponibilidade):**
```bash
docker service scale takadatintas_wordpress_nginx=3
docker service scale takadatintas_wordpress_wordpress=3
```
