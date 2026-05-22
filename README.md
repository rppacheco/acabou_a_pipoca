# 🗞️ GeekNews Instagram Bot

Automação completa para uma página de notícias geek no Instagram.  
Dois fluxos independentes: **coleta de imagens** e **publicação de posts**.

---

## 📁 Estrutura do projeto

```
geek_news_bot/
│
├── main.py                   # Entry point — orquestra os dois fluxos
├── fetch_press_kit.py        # Fluxo 1: busca imagens de press releases
├── press_kit_scraper.py      # Scraper base de imagens por tema/personagem
├── image_fallback.py         # Fallback: template padrão quando não há imagem
├── instagram_publisher.py    # Fluxo 2: publica no Instagram via instagrapi
│
├── generator.py              # (a integrar) Monta templates com Pillow
│
├── press_kit_raw/            # Imagens organizadas por tema/personagem
│   ├── marvel/
│   │   ├── spider-man/
│   │   ├── avengers/
│   │   └── _outros/
│   ├── dc/
│   ├── star_wars/
│   ├── nintendo/
│   ├── playstation/
│   ├── anime/
│   └── _fallbacks/           # Templates genéricos gerados automaticamente
│
├── output/                   # Templates prontos para publicar
│   ├── feed/
│   ├── story/
│   └── carrossel/
│
├── runs/                     # Relatório JSON de cada execução
├── session.json              # Sessão do Instagram (gerada automaticamente)
├── fallback_alerts.jsonl     # Log de posts publicados com imagem genérica
└── geek_news.log             # Log geral da aplicação
```

---

## ⚙️ Setup

### 1. Instalar dependências

```bash
pip install requests beautifulsoup4 pillow instagrapi tqdm
```

### 2. Variáveis de ambiente

Crie um arquivo `.env` na raiz do projeto (ou exporte no terminal):

```bash
# Instagram
IG_USERNAME=seu_usuario_instagram
IG_PASSWORD=sua_senha_instagram

```

> **Dica:** No Claude Code você pode pedir  
> *"adiciona suporte a python-dotenv para carregar o .env automaticamente"*

### 3. Verificar instalação

```bash
python main.py presskit --status
```

---

## 🔵 Fluxo 1 — Press Kit

Busca e organiza imagens oficiais de press releases nas fontes de cada franquia.  
**Rode antes de publicar** para garantir imagens disponíveis.

### Comandos

```bash
# Buscar imagens de um personagem específico
python main.py presskit --tema marvel --personagem spider-man

# Buscar tudo de um tema
python main.py presskit --tema playstation

# Resolver todas as pendências abertas
python main.py presskit --pendencias

# Ver estado do sistema de pastas
python main.py presskit --status

# Limitar número de imagens baixadas
python main.py presskit --tema anime --personagem goku --max 10
```

### Temas disponíveis

| Tema | Personagens exemplo |
|---|---|
| `marvel` | spider-man, avengers, thunderbolts, x-men... |
| `dc` | batman, superman, wonder-woman, joker... |
| `star_wars` | mandalorian, grogu, andor, ahsoka... |
| `nintendo` | zelda, mario, kirby, metroid... |
| `playstation` | god-of-war, the-last-of-us, horizon... |
| `anime` | jujutsu-kaisen, one-piece, demon-slayer... |

```bash
# Listar todos os personagens suportados
python fetch_press_kit.py --personagens
```

### Fontes por tema

| Tema | Fontes |
|---|---|
| Marvel | marvel.com/articles · news.disney.com · press.disneyplus.com |
| DC | dc.com/news · press.wbd.com |
| Star Wars | starwars.com/news · lucasfilm.com/blog |
| Nintendo | nintendo.com/whatsnew |
| PlayStation | blog.playstation.com · sonyinteractive.com/news |
| Anime | crunchyroll.com/news · animenewsnetwork.com |

---

## 🟡 Fluxo 2 — Publicação

Busca notícias, monta templates e publica no Instagram.  
Funciona mesmo sem imagens no press kit — publica template padrão com o título.

### Comandos

```bash
# Publicar temas específicos (feed + story por padrão)
python main.py publicar --temas marvel playstation

# Publicar todos os temas
python main.py publicar --temas todos

# Escolher formatos
python main.py publicar --temas anime --formatos feed
python main.py publicar --temas marvel --formatos feed story carrossel

# Simular sem publicar nada (recomendado antes do primeiro uso real)
python main.py publicar --temas marvel --dry-run
```

### Comportamento quando não há imagem

1. Verifica `press_kit_raw/` via `index.json`
2. Se não encontrar → gera **template padrão** com fundo temático + título da notícia
3. Publica normalmente
4. Exibe aviso no terminal e registra em `fallback_alerts.jsonl`
5. Você pode resolver depois: `python main.py presskit --pendencias`

### Primeiro login no Instagram

Na primeira execução o bot vai pedir usuário e senha (ou código 2FA se ativado).  
A sessão é salva em `session.json` — nas próximas execuções o login é automático.

```bash
# O login acontece automaticamente ao publicar
python main.py publicar --temas marvel --dry-run
```

> ⚠️ **Atenção:** Não compartilhe `session.json` — ele dá acesso à sua conta.  
> Adicione ao `.gitignore` se usar Git.

---

## 📋 Fluxo de trabalho sugerido

```bash
# 1. Antes de publicar, alimente o press kit
python main.py presskit --tema marvel --personagem thunderbolts

# 2. Verifique o que está disponível
python main.py presskit --status

# 3. Simule a publicação primeiro
python main.py publicar --temas marvel --dry-run

# 4. Publique de verdade
python main.py publicar --temas marvel

# 5. Se houve imagens genéricas, resolva
python main.py presskit --pendencias
```

---

## 🔌 Integrando seu scraper de notícias

Edite a função `_buscar_noticias()` em `main.py`:

```python
def _buscar_noticias(tema: str) -> list[dict]:
    # Exemplo com RSS:
    import feedparser
    feeds = {
        "marvel":      "https://www.marvel.com/articles.rss",
        "playstation": "https://blog.playstation.com/feed/",
        "anime":       "https://www.crunchyroll.com/newsrss",
    }
    feed = feedparser.parse(feeds.get(tema, ""))
    return [
        {
            "title":      e.title,
            "excerpt":    e.get("summary", "")[:300],
            "category":   tema.title(),
            "theme":      tema,
            "character":  "_outros",   # ou detectar pelo título
            "date":       e.get("published", ""),
            "source_url": e.link,
        }
        for e in feed.entries[:3]   # máx 3 por execução
    ]
```

---

## 🔌 Integrando o generator de templates

Edite `_gerar_template()` em `main.py`:

```python
def _gerar_template(tema, news, img_path, formato) -> Path:
    from generator import generate_feed, generate_story
    out = Path(f"output/{formato}/{tema}_{datetime.now():%Y%m%d_%H%M%S}.png")
    out.parent.mkdir(parents=True, exist_ok=True)
    if formato == "feed":
        generate_feed(tema, news, str(img_path), str(out))
    elif formato == "story":
        generate_story(tema, news, str(img_path), str(out))
    return out
```

---

## 📂 Arquivos gerados automaticamente

| Arquivo | Conteúdo |
|---|---|
| `docs/index.html` | Página pública de links recentes para GitHub Pages/Linktree |
| `docs/links.json` | Dados usados pela página de links recentes |
| `session.json` | Sessão salva do Instagram |
| `press_kit_raw/index.json` | Índice de todas as imagens disponíveis |
| `press_kit_raw/.downloaded_hashes.json` | Hashes para evitar re-download |
| `fallback_alerts.jsonl` | Log de posts com imagem genérica |
| `geek_news.log` | Log geral da aplicação |
| `runs/run_YYYYMMDD_HHMMSS.json` | Relatório de cada execução |

---

## 🔗 Página de links para Linktree

Gera uma página estática com links das matérias dos últimos 3 dias.  
Use o GitHub Pages apontando para a pasta `docs/` e coloque a URL dessa página no Linktree.

```bash
python main.py links
```

Opções úteis:

```bash
# Mudar a janela de tempo
python main.py links --days 5

# Incluir posts ainda em revisão
python main.py links --status pending_review approved published

# Gerar em outra pasta
python main.py links --output docs
```

A página inclui UTM nos links:

```text
utm_source=instagram
utm_medium=linktree
utm_campaign=acabou_a_pipoca
```

No painel local de revisão também existe o botão `Atualizar links`, que regenera `docs/index.html` e `docs/links.json`.

---

## ⚠️ Boas práticas com o Instagram

- Não publique mais de **3–5 posts por hora**
- Sempre rode o `--dry-run` antes de uma nova configuração
- Use sempre o mesmo IP/máquina para evitar bloqueios
- Não compartilhe `session.json` nem `.env`

---

## 🛠️ Usando com Claude Code

```bash
# Abra o projeto no Claude Code
cd geek_news_bot
claude

# Exemplos do que pedir dentro da sessão:
# "roda o fluxo 1 para marvel e me mostra o status"
# "esse erro aqui, o que está acontecendo?"
# "adiciona suporte a Reels no publisher"
# "cria o generator.py com Pillow para o tema marvel"
```

---

## 📦 Dependências

```
requests>=2.31
beautifulsoup4>=4.12
pillow>=10.0
instagrapi>=2.0
tqdm>=4.66
```

```bash
pip install requests beautifulsoup4 pillow instagrapi tqdm
```
