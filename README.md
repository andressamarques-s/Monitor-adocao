# Monitor de Adoção 

Ferramenta interna que lê os backups (`.zip`) exportados do sistema SOC (RH/saúde
ocupacional) de cada cliente, cruza os dados com a lista de instituições cadastradas
e gera:

- um **relatório de acompanhamento de unidades** (HTML/PDF) por empresa, com
  indicadores de adoção por módulo (Exames, EPI, Vacinação, Treinamentos,
  Questionários, Plano de Ação, entre outros);
- um **Dashboard** interativo (`/dashboard`) pra consultar os mesmos indicadores em
  gráficos, com filtros de empresa/período, sem precisar gerar o relatório completo.

O site é servido por um `http.server` simples (sem framework), com login por CPF +
senha e dois perfis de acesso (`administrador`, `usuario`).

> **Sobre esta versão:** o `monitor-adocao` (este repositório) lê o **backup em
> XML** exportado manualmente do sistemas. 

## Requisitos

- Python 3.10 ou mais recente
- Dependências Python: `pip install -r requirements.txt` (Jinja2 + MarkupSafe)
- **Microsoft Edge** instalado na máquina que gera os PDFs (`site.py` chama
  `msedge --headless --print-to-pdf` pra converter o relatório HTML em PDF). 

## Configuração (`.env`)

Crie um arquivo `.env` na raiz do projeto (mesma pasta de `site.py`):

```ini
PASTA_BACKUP=\\servidor\caminho\para\a\pasta\com\os\zips\de\backup
PASTA_RELATORIOS=\\servidor\caminho\onde\salvar\os\relatorios\html
```

- `PASTA_BACKUP`: pasta onde os `.zip` de backup do cliente ficam disponíveis. O
  site lista o(s) mais recente(s) dali automaticamente.
- `PASTA_RELATORIOS`: pasta onde os relatórios HTML gerados são salvos (opcional -
  sem essa variável, usa `relatorios_gerados/html` dentro do próprio projeto). Os
  PDFs sempre vão pra `relatorios_gerados/pdf`, dentro do projeto.


## Primeiro uso

```powershell
pip install -r requirements.txt
python cadastrar_usuario.py      
python site.py                   
```

Todo usuário novo nasce com a senha padrão (`suporte123`, ver `usuarios.SENHA_PADRAO`)
e é obrigado a trocá-la no primeiro login. 

Pra cadastrar mais gente depois, rode `python cadastrar_usuario.py` de novo (pede
CPF, nome e perfil — `administrador` ou `usuario`).

## Estrutura do projeto

```
site.py                  orquestra as rotas HTTP, extração e geração de relatório/PDF
front_end_site/telas.py  monta o HTML de cada tela do site (login, dashboard, wizard)
front_end_site/estilo.css  CSS do site (não é o CSS do relatório)
relatorio_html.py        monta os dados e o HTML do relatório completo (Jinja2)
templates/                templates e CSS do relatório em si (relatorio.html)
controller.py / nexus.py  extraem e agregam os dados do backup (.zip -> CSVs)
banco_dados.py           cache de extração por backup (evita reprocessar o mesmo zip)
usuarios.py / sessoes.py  login, senha (PBKDF2), sessão, perfis/permissões
validacao_backup.py       registra quando um backup foi validado (orion.py)
log_acessos.py            log de acesso/ações, append-only (log_acessos.csv)
cadastrar_usuario.py      script de linha de comando pra cadastrar usuário novo
ambiente.py                lê o .env (PASTA_BACKUP, PASTA_RELATORIOS, HOST)
```

## Deploy em produção (Linux, `/opt` + systemd)

Os passos abaixo assumem um servidor Linux (Debian/Ubuntu como exemplo — adapte o
gerenciador de pacotes se for outra distro) dedicado a rodar o site continuamente.

### 1. Preparar a pasta e o usuário de sistema

Rodar como um usuário dedicado (não root) é mais seguro — se algo no site for
comprometido, o dano fica limitado ao que esse usuário pode acessar.

```bash
sudo mkdir -p /opt/monitor_adocao
sudo useradd --system --home /opt/monitor_adocao --shell /usr/sbin/nologin soc-monitor
sudo chown -R soc-monitor:soc-monitor /opt/monitor_adocao
```

Copie o conteúdo do projeto pra `/opt/monitor_adocao` (rsync, scp, git clone — o
que for mais prático no seu fluxo). 

### 2. Ambiente virtual e dependências.

```bash
cd /opt/monitor_adocao
sudo -u soc-monitor python3 -m venv venv
sudo -u soc-monitor venv/bin/pip install -r requirements.txt
```

### 3. `.env` de produção

Crie `/opt/monitor_adocao/.env` (dono `soc-monitor`, permissão `600` — tem caminho
de rede interno, não precisa estar legível por todo mundo):

```ini
PASTA_BACKUP=/mnt/backups-soc
PASTA_RELATORIOS=/mnt/relatorios-soc
```

Se `PASTA_BACKUP`/`PASTA_RELATORIOS` hoje apontam pra um compartilhamento Windows
(`\\servidor\...`), no Linux isso vira um ponto de montagem CIFS/SMB (pacote
`cifs-utils`, entrada em `/etc/fstab` ou `systemd.mount`) — o `.env` aponta pro
caminho local onde esse compartilhamento fica montado, não pro caminho Windows
original.

```bash
sudo chmod 600 /opt/monitor_adocao/.env
sudo chown soc-monitor:soc-monitor /opt/monitor_adocao/.env
```

### 4. Cadastrar o primeiro usuário administrador

```bash
cd /opt/monitor_adocao
sudo -u soc-monitor venv/bin/python cadastrar_usuario.py
```

### 5. Navegador para gerar PDF (atenção)

`site.py` (`_localizar_msedge`) hoje só procura `msedge.exe` em caminhos do
Windows, com fallback pra `msedge` no `PATH`. **Isso não existe por padrão num
servidor Linux** — a geração de PDF vai falhar até que exista um binário
compatível. Duas opções:

- Instalar o **Microsoft Edge para Linux** (pacote oficial da Microsoft) e criar um
  link simbólico `msedge` apontando pro binário real (geralmente
  `microsoft-edge-stable`), **ou**
- Pedir pra eu ajustar `_localizar_msedge()` pra também procurar
  `microsoft-edge`/`google-chrome`/`chromium` no Linux (mesmos flags
  `--headless --print-to-pdf` funcionam em qualquer navegador Chromium) — não fiz
  essa mudança agora porque não foi pedida, mas é pequena se você quiser.

O site em si (login, Dashboard, relatório na tela) funciona sem isso — só o botão
"Baixar PDF" depende de um navegador Chromium disponível no servidor.


```

### 6. Ativar e iniciar

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monitor-adocao
sudo systemctl status monitor-adocao
journalctl -u monitor-adocao -f     # acompanhar o log em tempo real
```

### 7. Acesso pela rede

Por padrão o site escuta só em `127.0.0.1` (`HOST` no `.env`, ver seção de
configuração) — ninguém fora da própria máquina consegue acessar diretamente. Pra
um time inteiro acessar, duas opções:

- **Recomendado**: colocar um proxy reverso (nginx, por exemplo) na frente, com
  TLS (HTTPS)
- Definir `HOST=0.0.0.0` no `.env` pra escutar em todas as interfaces de rede sem
  proxy — .


