# HARDENING.md — Hardening de Containers

Documento da atividade prática de hardening (`docs/atividade.md`).
Projeto: aplicação Node.js + MySQL executados com Docker Compose.

---

## 1. Problemas encontrados

Análise feita **antes** de qualquer alteração, lendo `Dockerfile`, `docker-compose.yml`, `app/` e a estrutura do repositório.

### 1.1 Dockerfile (original)

| # | O que estava errado | Risco | Onde |
|---|---------------------|-------|------|
| 1 | Base `ubuntu:latest` | Tag flutuante: o build muda sem aviso (não reprodutível) e pode trazer regressões/vulnerabilidades novas. Ubuntu completo traz centenas de pacotes que a aplicação não usa (cada um é superfície de ataque e fonte de CVEs). | `Dockerfile`, linha 1 |
| 2 | `apt-get update` e `apt-get install -y nodejs npm` sem `--no-install-recommends` e sem limpar `/var/lib/apt/lists` | Instala pacotes recomendados desnecessários, aumenta a imagem e deixa cache de apt nas camadas. O Node vem do repositório do Ubuntu, sem garantia de ser uma versão LTS atual. | linhas 7–8 |
| 3 | `COPY . .` | Copia **todo** o contexto para a imagem (`.git`, `docs/`, e qualquer `.env` local). Um `.env` com segredos acabaria dentro das camadas da imagem. Também invalida o cache a cada mudança em qualquer arquivo. | linha 5 |
| 4 | `RUN npm install` executado em `/app`, onde não existe `package.json` (ele está em `app/`) | O passo de instalação está no diretório errado e não é determinístico (`npm install` em vez de `npm ci`), além de permitir scripts de instalação de pacotes. Obs.: `app/package.json` também estava **vazio** (JSON inválido). | linha 10 e `app/package.json` |
| 5 | `ENV DB_PASSWORD=123456` | Senha em texto puro versionada no Git e gravada nos metadados/camadas da imagem (`docker history`, `docker inspect`). Quem puxar a imagem lê a senha. Mesmo funcionando, é uma credencial vazada por design. | linha 12 |
| 6 | Sem instrução `USER` | O processo roda como **root** dentro do container. | todo o arquivo |
| 7 | Sem `HEALTHCHECK` | O Docker não sabe se a aplicação está saudável; falhas silenciosas passam despercebidas. | todo o arquivo |
| 8 | Sem `.dockerignore` | Nada impede segredos e arquivos irrelevantes de entrarem no contexto de build. | raiz do projeto |

### 1.2 docker-compose.yml (original)

| # | O que estava errado | Risco | Onde |
|---|---------------------|-------|------|
| 9 | `image: mysql:latest` | Tag flutuante: um `pull` pode trocar de versão maior e quebrar o banco/dados, ou introduzir mudanças não revisadas. | linha 9 |
| 10 | `MYSQL_ROOT_PASSWORD: root` literal | Senha fraca, previsível e versionada no Git. A aplicação não tem usuário dedicado, então o uso natural é com o `root` do banco (privilégio máximo). | linha 11 |
| 11 | `ports: "3306:3306"` no serviço `banco` | O banco fica publicado em **todas as interfaces do host** (0.0.0.0), acessível pela rede local/internet. Somado à senha `root`, é acesso direto ao banco. O banco só é usado pela aplicação. | linhas 13–14 |
| 12 | Nenhuma rede definida | App e banco ficam na rede padrão única, sem segmentação; não há como separar o que é exposto do que é interno. | arquivo todo |
| 13 | Sem limites de CPU, memória e processos | Um container comprometido (ou com bug) pode consumir todo o host (DoS), derrubar outros serviços ou executar *fork bomb*. | arquivo todo |
| 14 | Sem `cap_drop`, `no-new-privileges` nem `read_only` | Os containers rodam com o conjunto padrão de *capabilities* do Docker e sistema de arquivos gravável: mais ferramentas para um atacante escalar privilégio ou persistir. | arquivo todo |
| 15 | Dados do MySQL sem volume nomeado | O MySQL cria um volume anônimo (difícil de rastrear/gerenciar); questão de disponibilidade e gestão, não de confidencialidade. | `banco` |
| 16 | Porta `3000:3000` publicada em todas as interfaces | É necessária (o app precisa receber requisições), mas é o único ponto que deve ficar exposto. Em produção, o ideal é colocá-la atrás de um *reverse proxy*/TLS. | linha 6 |

### 1.3 Repositório e processo

| # | O que estava errado | Risco |
|---|---------------------|-------|
| 17 | Sem `.gitignore` | Nada impedia o `.env` de ser commitado por acidente. |
| 18 | Sem `.env.example` | Quem clona o projeto não sabe quais variáveis são necessárias. |
| 19 | Senhas (`123456`, `root`) já estão no **histórico do Git** | Remover do arquivo atual não apaga o histórico. Em um sistema real, essas credenciais precisariam ser **rotacionadas** (trocadas), e não apenas removidas. |

### 1.4 O que foi verificado e **não** existe (Etapa 7)

Foram procurados e **não** estão presentes no projeto original: `privileged: true`, montagem de `/var/run/docker.sock`, `network_mode: host`, `pid: host`, `devices:` e *bind mounts* de diretórios do host. Os riscos de isolamento restantes eram os privilégios padrão (capabilities, root, FS gravável) citados acima.

---

## 2. Medidas aplicadas

### 2.1 Imagem (`Dockerfile`)

| Alteração | Objetivo de segurança | Risco reduzido |
|-----------|----------------------|----------------|
| `FROM node:24.21.0-alpine3.23` (Node 24 LTS, versão fixa, Alpine) em vez de `ubuntu:latest` | Imagem pequena, com poucos pacotes, já com Node pronto e **versão fixa** | Menos CVEs e componentes exploráveis; builds reprodutíveis |
| Removido `apt-get`; nada além do runtime é instalado | Só o necessário na imagem final | Superfície de ataque e tamanho |
| `COPY app/package*.json` → `npm ci --omit=dev --ignore-scripts` → `COPY app/` | Instalação determinística (lockfile), sem devDependencies e sem scripts de pacotes; copia só o código da aplicação | Supply chain, vazamento de arquivos para a imagem |
| `npm cache clean --force` | Não deixa cache de npm nas camadas | Tamanho e lixo na imagem |
| Arquivos da aplicação pertencem ao root e são somente leitura para quem executa | A aplicação não consegue se alterar em disco | Persistência de um atacante |
| `USER 1000:1000` (usuário `node`) | **Menor privilégio**: nada roda como root | Impacto de um comprometimento |
| `ENV DB_PASSWORD` **removido** | Nenhum segredo nas camadas/metadata da imagem | Vazamento de credencial |
| `HEALTHCHECK` (`/health`) | O Docker sabe se a aplicação está saudável | Falhas silenciosas |
| `.dockerignore` (`.env*`, `.git`, `docs`, etc.) | Segredos e arquivos irrelevantes nunca entram no contexto de build | Vazamento durante o build |

### 2.2 Segredos (`.env`, `.env.example`, `.gitignore`)

| Alteração | Objetivo | Risco reduzido |
|-----------|----------|----------------|
| Valores sensíveis movidos para `.env` local (`DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_ROOT_PASSWORD`), com senhas **novas, aleatórias e fortes** (a `123456`/`root` antigas foram descartadas) | Segredos fora do código, do `Dockerfile` e do `docker-compose.yml` | Credenciais versionadas |
| `docker-compose.yml` usa `${VARIAVEL:?mensagem}` | O compose **aborta** com mensagem clara se faltar variável | Subir o ambiente mal configurado ou com senha vazia |
| `.gitignore` com `.env` e `.env.*` (exceto `.env.example`) | O `.env` não vai para o Git | Commit acidental de segredos |
| `.env.example` com **apenas os nomes** das variáveis | Documenta o que é necessário sem expor valores | Onboarding inseguro (copiar senhas de alguém) |
| Usuário de aplicação dedicado no MySQL (`MYSQL_USER`/`MYSQL_PASSWORD`); a app **não** usa o `root` | Menor privilégio também no banco | Impacto de SQL injection / vazamento da credencial da app |

### 2.3 Rede (`docker-compose.yml`)

| Alteração | Objetivo | Risco reduzido |
|-----------|----------|----------------|
| Removido `ports: 3306:3306` do `banco` | **Expor somente o necessário** | Acesso direto ao banco pelo host/rede |
| Rede `backend` com `internal: true` (app + banco) | O banco só conversa com a aplicação, sem saída para a internet nem para o host | Exfiltração e acesso lateral |
| Rede `frontend` só com o `app` | Separa o que recebe tráfego externo do que é interno | Exposição indevida |
| Apenas `app` publica porta (3000) | É o único serviço que precisa receber requisições externas | Redução do número de pontos de entrada |
| `depends_on: condition: service_healthy` | App só sobe com o banco saudável | Falhas de inicialização em cascata |

### 2.4 Recursos

| Serviço | CPU | Memória (sem swap) | Processos |
|---------|-----|--------------------|-----------|
| `app` | 0,50 | 256 MB | 100 |
| `banco` | 1,0 | 1 GB | 200 |

`memswap_limit` igual a `mem_limit` impede o uso de swap como "memória extra". Risco reduzido: DoS por consumo de recursos e *fork bomb*; um container comprometido não derruba o host nem os vizinhos (o kernel encerra por OOM apenas o container que estourou o limite).

### 2.5 Isolamento

| Alteração | Serviço | Objetivo |
|-----------|---------|----------|
| `cap_drop: [ALL]` | app e banco | Remove todas as *capabilities* do kernel |
| `cap_add: CHOWN, DAC_OVERRIDE, FOWNER, SETGID, SETUID` | banco | Somente o necessário para o entrypoint do MySQL ajustar o diretório de dados e trocar para o usuário `mysql` |
| `security_opt: no-new-privileges:true` | app e banco | Impede ganhar privilégios via binários setuid |
| `read_only: true` + `tmpfs: /tmp (noexec,nosuid)` | app | Sistema de arquivos imutável; só `/tmp` é gravável e não executável |
| Versões fixas (`node:24.21.0-alpine3.23`, `mysql:8.4` LTS) | ambos | Previsibilidade |
| Sem `privileged`, `docker.sock`, `network_mode: host` ou *bind mounts* | ambos | Mantido sem acesso ao host |

### 2.6 Ajustes mínimos na aplicação (`app/`)

Para a aplicação ser executável e validável, foram necessários dois ajustes pequenos:

- `app/package.json`: estava vazio (JSON inválido, `npm` não consegue ler). Agora é um manifesto válido, **sem dependências**; foi gerado `app/package-lock.json` para permitir `npm ci`.
- `app/server.js`: mantém a saída original (`Olá mundo!`), mas agora é um servidor HTTP mínimo na porta 3000 (com `/health`) que **exige** as variáveis `DB_*` (falha rápido se faltarem) e **nunca** imprime seus valores. Isso prova que a aplicação continua recebendo a configuração por variáveis de ambiente e dá sentido à porta publicada e ao healthcheck. Não usa dependências externas.

### 2.7 Perguntas das etapas

**Etapa 2 — Por que uma imagem menor e com menos componentes reduz a superfície de ataque?**
Cada pacote, biblioteca ou binário presente na imagem é código que pode ter vulnerabilidades (CVEs) e que um atacante pode usar depois de invadir (shell, `curl`, compiladores, gerenciador de pacotes). Menos componentes significam menos CVEs para corrigir, menos ferramentas disponíveis para exploração, menos coisas para auditar e atualizar, e download/build mais rápidos.

**Etapa 3 — Impacto de um atacante controlar uma aplicação rodando como root no container?**
Ele teria privilégio total dentro do container: pode ler/alterar qualquer arquivo, instalar ferramentas, abrir conexões, ler variáveis de ambiente e, se houver má configuração (capabilities extras, `docker.sock`, volumes do host, kernel vulnerável), aumentar a chance de **escapar do container** e chegar ao host. Rodar como root também facilita escalada e movimentação lateral. Como não-root, `cap_drop: ALL` e FS somente leitura, o mesmo atacante fica muito mais limitado.

**Etapa 4 — Perguntas:**

1. *Por que senha direto no `Dockerfile` é um risco mesmo com a aplicação funcionando?* Porque a senha fica gravada na imagem (camadas e metadata: `docker history`, `docker inspect`), no Git e em qualquer registry/cache onde a imagem for parar. Quem tiver acesso à imagem ou ao repositório obtém a credencial, sem precisar invadir nada. Funcionar não é o critério de segurança.
2. *Por que o `.env` não vai para o repositório?* Porque o Git guarda histórico e é replicado (forks, clones, CI). Um segredo versionado é considerado vazado e só se resolve rotacionando-o; remover o arquivo depois não apaga o histórico.
3. *Qual a finalidade do `.env.example`?* Servir de contrato/documentação: lista os nomes das variáveis necessárias, sem os valores, para que cada pessoa crie seu `.env` local (`cp .env.example .env`).
4. *Diferença entre senha no `docker-compose.yml` e variável vinda do `.env`?* No primeiro caso o segredo faz parte do arquivo versionado (vai para o Git e para quem o ler). No segundo, o arquivo versionado só tem uma referência (`${DB_PASSWORD}`) e o valor real fica fora do repositório, podendo mudar por ambiente sem editar nem commitar o compose. Ressalva: o valor continua visível em `docker inspect` do container em execução; para produção, o ideal é um gerenciador de segredos (Docker/Compose secrets, Vault, etc.).

**Etapa 5 — O banco precisa expor a porta para toda a máquina hospedeira se só a aplicação o usa?**
Não. A aplicação acessa o banco pelo nome `banco` na rede interna do Compose, sem publicar porta. Publicar a 3306 só amplia a superfície (qualquer um que alcance o host pode tentar autenticar, fazer *brute force* ou explorar uma CVE do MySQL). Princípio: expor somente o necessário. Se alguém precisar acessar o banco para administração, deve usar `docker compose exec banco mysql ...` ou um túnel temporário, nunca uma porta permanente.

**Etapa 6 — O que acontece se um container consumir toda a CPU/memória do host?**
Os demais serviços sofrem lentidão e *timeouts*; o kernel pode acionar o OOM Killer e encerrar processos aleatórios (inclusive de outros containers ou do próprio Docker), e o host pode travar: é um ataque de negação de serviço. Com limites de CPU, memória e PIDs, o impacto fica restrito ao container problemático.

---

## 3. Análise final

**1. Qual era o principal risco encontrado?**
O banco de dados exposto em todas as interfaces do host (`3306:3306`) com a senha de `root` fraca (`root`) escrita em arquivo versionado. É o risco mais grave porque é explorável remotamente, sem precisar comprometer a aplicação antes, e dá acesso total aos dados.

**2. Qual alteração foi mais importante? Por quê?**
Remover a publicação da porta do banco e isolá-lo na rede `internal`. Ela elimina o ponto de entrada em vez de apenas mitigar o impacto: sem rota até o banco, a senha fraca e eventuais CVEs do MySQL deixam de ser alcançáveis de fora. Logo depois vêm, em importância, a retirada dos segredos do código/imagem e a execução sem root; segurança é em camadas (defesa em profundidade), nenhuma medida substitui as outras.

**3. O que poderia acontecer se o container da aplicação fosse comprometido?**
Antes: o atacante teria root no container, leria `DB_PASSWORD` (e as demais variáveis), alcançaria o banco com `root`/`root`, poderia consumir todos os recursos do host e tentaria escapar do container aproveitando as capabilities padrão. Depois: o processo roda sem root, sem capabilities, com FS somente leitura, sem novos privilégios e com limites de CPU/memória/PIDs; o usuário do banco que ele recebe só acessa o schema da aplicação, e o banco só é alcançável por uma rede interna. O dano é contido, embora não eliminado (por exemplo, ele ainda lê os dados que a aplicação consegue ler).

**4. Como o princípio do menor privilégio foi aplicado?**
Em várias camadas: (a) usuário não-root (`USER 1000:1000`); (b) `cap_drop: ALL` (e no banco, só 5 capabilities necessárias); (c) `no-new-privileges`; (d) FS somente leitura no app, com `/tmp` `noexec`; (e) arquivos da aplicação não graváveis pelo usuário de execução; (f) a aplicação usa um usuário do MySQL dedicado em vez do `root`; (g) redes separadas: cada serviço só alcança o que precisa; (h) só uma porta publicada; (i) limites de recursos.

**5. Por que o `.env` não deve ser enviado ao repositório?**
Contém segredos reais. O repositório é compartilhado, clonado, replicado em forks/CI e mantém histórico permanente; qualquer pessoa com acesso (ou um vazamento) obteria as credenciais, e só trocá-las resolve. Por isso está no `.gitignore` e no `.dockerignore`.

**6. Qual é a função do `.env.example`?**
Documentar quais variáveis o projeto exige (apenas nomes, sem valores reais) e permitir que cada pessoa crie seu `.env` local. Pode ser versionado justamente porque não contém segredos.

**7. Como o projeto poderia receber novas verificações de segurança automaticamente em uma pipeline CI/CD?**
Com um workflow (ex.: GitHub Actions) a cada *push*/*pull request* que rode, por exemplo: **Hadolint** (lint do Dockerfile); **Gitleaks**/TruffleHog (segredos em código e histórico); **Trivy** (`trivy config .` para o Dockerfile/compose e `trivy image` para CVEs na imagem, falhando o pipeline em severidade HIGH/CRITICAL); `docker compose config` (valida o compose); geração de **SBOM** (Syft) e **assinatura** da imagem (Cosign); **Dependabot/Renovate** para atualizar bases fixas e dependências; e *deploy* só com imagem aprovada, preferencialmente fixada por digest. Exemplo mínimo:

```yaml
name: security
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint do Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
      - name: Segredos
        uses: gitleaks/gitleaks-action@v2
      - name: Trivy (config + dependências)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: "1"
```

**8. Quais medidas adicionais em Kubernetes?**
`securityContext` no pod/container (`runAsNonRoot: true`, `runAsUser` numérico, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`); **Pod Security Admission** em nível *restricted*; **NetworkPolicy** com *default deny* e liberações explícitas (app → banco); `requests`/`limits` + `ResourceQuota`/`LimitRange`; **Secrets** com criptografia em repouso ou gerenciador externo (External Secrets, Vault) em vez de variáveis em texto; **RBAC** mínimo e `automountServiceAccountToken: false`; imagens fixas por **digest**, de registry privado, com verificação de assinatura/política (Kyverno/OPA Gatekeeper); varredura contínua de imagens; Ingress com TLS; *probes* de liveness/readiness; segmentação por *namespaces*; auditoria e detecção em runtime (Falco); atualização regular do cluster e dos nós.

---

## 4. Evidências da validação

### 4.1 Validações já executadas (sem necessidade de daemon Docker)

Executadas com Docker Compose v5.6.0, Hadolint 2.15.1 e Node.js 22 local.

```text
$ hadolint Dockerfile
(sem avisos; exit code 0)

$ docker compose config            # sem .env
error while interpolating services.app.environment.DB_NAME: required variable DB_NAME is missing a value: defina DB_NAME no arquivo .env
error while interpolating services.app.environment.DB_PASSWORD: required variable DB_PASSWORD is missing a value: defina DB_PASSWORD no arquivo .env
(exit code 1: o compose se recusa a subir sem segredos)

$ docker compose config            # com .env preenchido
(resolve sem erros: banco sem "ports", rede backend "internal: true",
 cpus/mem_limit/pids_limit, cap_drop ALL, no-new-privileges, read_only no app)

$ npm ci --omit=dev --ignore-scripts   # mesmo comando do Dockerfile
up to date, audited 1 package ... found 0 vulnerabilities

$ node app/server.js                    # sem variáveis
Variáveis de ambiente ausentes: DB_HOST, DB_USER, DB_PASSWORD, DB_NAME  (exit 1)

$ node app/server.js                    # com variáveis do .env
Olá mundo!
Servidor ouvindo na porta 3000
GET /        -> Olá mundo!
GET /health  -> {"status":"ok"}

$ grep -rnE "123456|MYSQL_ROOT_PASSWORD: root|PASSWORD=[^ ]+" Dockerfile docker-compose.yml app .env.example
(nenhum segredo literal encontrado)

$ git check-ignore -v .env
.gitignore:2:.env	.env
```

### 4.2 Validações no container (executar e colar a saída)

> Estes comandos exigem Docker em execução. Rode-os e **cole a sua saída real** abaixo de cada item.

```bash
cp .env.example .env      # preencha os valores (ou use o seu .env já existente)
docker compose up -d --build
docker compose ps         # app e banco "healthy"
```

| Verificação | Comando | Resultado esperado |
|-------------|---------|--------------------|
| Aplicação funcionando | `curl -s localhost:3000` | `Olá mundo!` |
| Usuário não-root | `docker compose exec app id` | `uid=1000(node) gid=1000(node)` |
| Sem capabilities | `docker compose exec app sh -c 'grep CapEff /proc/1/status'` | `CapEff: 0000000000000000` |
| no-new-privileges | `docker compose exec app sh -c 'grep NoNewPrivs /proc/1/status'` | `NoNewPrivs: 1` |
| FS somente leitura | `docker compose exec app touch /app/teste` | `Read-only file system` |
| Banco não publicado no host | `docker compose port banco 3306` e `nc -zv localhost 3306` | sem porta mapeada / conexão recusada |
| App alcança o banco internamente | `docker compose exec app sh -c 'nc -w 2 banco 3306 </dev/null >/dev/null && echo conectou'` | `conectou` |
| Rede do banco é interna | `docker network inspect hardening-container_backend --format '{{.Internal}}'` | `true` |
| Limites aplicados | `docker inspect -f '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}} {{.HostConfig.PidsLimit}}' $(docker compose ps -q app)` | `268435456 500000000 100` |
| Consumo vs. limite | `docker stats --no-stream` | `MEM USAGE / LIMIT` mostrando 256MiB e 1GiB |
| Sem segredos na imagem | `docker history --no-trunc hardening-container-app:1.0.0 \| grep -i password` e `docker image inspect hardening-container-app:1.0.0 --format '{{.Config.Env}}'` | nenhuma ocorrência de `DB_PASSWORD` |
| `.env` fora da imagem | `docker run --rm --entrypoint ls hardening-container-app:1.0.0 -a /app` | sem `.env` |
| `.env` fora do Git | `git status --short` e `git ls-files .env` | `.env` não aparece |

**Saída obtida:**

```text
(cole aqui a saída dos comandos acima)
```

### 4.3 Estado "antes" (referência)

O estado inicial pode ser reproduzido com `git checkout 74c1bff` (commit original) e `docker compose up --build`. Observações da leitura estática: base `ubuntu:latest`, processo como root, `DB_PASSWORD=123456` visível em `docker inspect`, `3306` publicado em todas as interfaces e ausência de qualquer limite de recurso.
