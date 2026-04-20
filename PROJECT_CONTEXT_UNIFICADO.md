# Contexto Unificado do Projeto

Atualizado em: 2026-04-20

## Objetivo deste arquivo

Este documento consolida, em um ponto unico, o contexto operacional e tecnico mais importante do projeto `EMaffiliates`.

Ele foi montado a partir de:

- `PROJECT_CONTEXT.md`
- `COMMANDS.md`
- `REVIEW_EXTERNAL_ASSESSMENT.md`
- estrutura atual do repositorio
- cobertura de testes encontrada em `tests/`

## Limite desta consolidacao

Este arquivo nao e uma transcricao literal de todas as conversas anteriores.
Eu nao encontrei, no workspace atual, um historico local completo dos chats.

Entao a consolidacao abaixo representa:

- o que ja esta documentado no projeto
- o que o codigo atual confirma
- o que aparece como decisao recorrente nas documentacoes existentes

## Resumo executivo

O projeto e uma maquina local de conteudo para afiliados, com foco em operar uma esteira semi-automatizada de produtos, enriquecimento, geracao de audio/video e publicacao social.

Fluxo central:

`painel -> banco -> enrich -> audio -> video -> publish instagram -> publish facebook -> github pages`

O sistema esta funcional ponta a ponta no fluxo principal e ja opera com orquestracao via n8n.

O foco atual nao e mais provar o conceito, e sim:

- manter estabilidade operacional
- reduzir retrabalho manual
- melhorar qualidade visual dos videos
- reforcar diagnostico, monitoramento e seguranca

## Workspace oficial

Diretorio principal:

- `C:\Projetos\EMaffiliates`

Decisao importante:

- o projeto foi migrado para fora do OneDrive
- a pasta fora do OneDrive deve ser tratada como workspace oficial

## O que o sistema ja faz hoje

- usa o painel web como fonte operacional principal
- sincroniza painel com banco SQLite
- remove do banco itens excluidos do painel
- pode limpar artefatos locais ligados a itens removidos
- enriquece dados de produto com OpenAI
- gera roteiro estruturado
- adapta roteiro para TTS
- gera audio com OpenAI TTS
- usa `edge-tts` como fallback
- gera timestamps com Whisper quando disponivel, com fallback proporcional
- gera thumbnail/capa
- gera video vertical com overlays animados
- valida artefatos de midia antes da publicacao
- publica no Instagram
- publica na Pagina do Facebook
- atualiza status e observacoes no painel
- salva legendas locais para Instagram e Facebook
- sobe previews de imagem para R2 quando configurado
- atualiza catalogos publicos no GitHub Pages
- roda automaticamente via n8n
- envia alertas por Telegram quando configurado

## Fonte de verdade e estados

### Fonte de verdade

O painel e a fonte de verdade do catalogo operacional.

Comportamentos consolidados:

- produto entrou no painel -> entra no banco na proxima sincronizacao
- produto saiu do painel -> sai do banco na proxima sincronizacao
- produto voltou para `pendente` -> volta para `raw`
- `force_regenerate` -> volta para `raw`
- `force_publish_only` -> volta para `video_ready` quando o video existe
- `force_facebook_only` -> permite republicacao isolada no Facebook

### Estados principais do pipeline

- `raw`
- `enriched`
- `audio_ready`
- `video_ready`
- `publishing`
- `published`
- `failed`

## Arquitetura consolidada

### Pipeline

Arquivos centrais do fluxo atual:

- `pipeline/cli.py`
- `pipeline/input_flow.py`
- `pipeline/product_flow.py`
- `pipeline/panel_pipeline.py`
- `pipeline/publish_flow.py`
- `pipeline/runtime_utils.py`
- `pipeline/shopee_affiliate_flow.py`
- `pipeline/runner.py`

Leitura arquitetural atual:

- `runner.py` ainda e importante como orquestrador
- a refatoracao ja avancou e extraiu responsabilidades para modulos especializados
- o projeto esta menos acoplado do que em avaliacoes antigas

### API

Ponto de entrada principal:

- `api/server.py`

Modulos auxiliares ja extraidos:

- `api/panel_sync.py`
- `api/stage_utils.py`

### Painel web atual

Estado consolidado em `2026-04-17` apos reverter a tentativa de autenticacao via backend:

- o painel publicado no `GitHub Pages` voltou ao fluxo antigo de login no frontend
- a senha e validada no navegador via `password_hash` presente em `config.json`
- o `GITHUB_TOKEN` usado para leitura e escrita no repo `painel` volta a ser armazenado no `config.json` em formato ofuscado/criptografado pelo setup
- a sessao do painel volta a depender de `sessionStorage`, o que preserva o login em `F5` na mesma aba

Arquivos principais da implementacao vigente:

- `painel/index.html`
- `scripts/setup_painel.py`

Arquivos e abordagem removidos da linha principal:

- `api/panel_store.py`
- rotas `/api/auth/*`
- rotas `/api/painel/*`
- `scripts/start_panel_api_tunnel.py`
- `scripts/start_panel_api_tunnel.bat`

Leitura operacional atual:

- o `config.json` publicado no repo `painel` contem `password_hash`, `github_user`, `github_repo` e o token ofuscado
- o painel acessa `produtos.json` e `candidates.json` diretamente pela API do GitHub
- o backend `api/server.py` continua existindo para pipeline, diagnostico e automacoes, mas nao e mais responsavel pelo login do painel

Limitacao importante que deve ficar explicita:

- esse modelo e operacionalmente simples e resolveu o problema de persistencia do login no navegador
- ele nao oferece seguranca forte, porque a autenticacao e feita no cliente
- se o projeto voltar a priorizar seguranca real para edicao do painel, a direcao futura continua sendo mover escrita e autenticacao para backend privado estavel

### Publicacao

Integracoes principais:

- Instagram via Meta Graph API
- Facebook Page API
- GitHub Pages para catalogos publicos
- Telegram para alerta operacional

## Camada de candidatos e multiplas origens

Uma decisao importante do projeto foi separar descoberta de operacao principal.

Em vez de mandar tudo direto para a esteira, existe uma camada anterior de candidatos para triagem visual.

Direcao consolidada:

- discovery e importacao assistida antes do painel principal
- consolidacao de candidatos em uma guia unica
- suporte a multiplas origens e contas
- fila de candidatos antes de importar para o painel principal
- acao humana minima para importar ou descartar

Estado atual conhecido:

- a vertical Mercado Livre ja deixou de ser apenas plano e hoje e a origem mais avancada nessa camada
- existe implementacao real de discovery por categoria para Mercado Livre
- existe page scraper enriquecido para Mercado Livre
- existe fila de candidatos
- existe workflow auxiliar de token no n8n
- a guia `Candidatos` usa `candidates.json` como fonte
- a guia suporta filtro por origem e status
- ja existem acoes de importar e descartar
- a vertical Shopee ja foi estruturada em pasta propria seguindo o padrao de `scrapers/mercadolivre`
- a Shopee ja possui discovery integrado a `candidates.json`
- a Shopee ja possui endpoints HTTP dedicados de precheck e discovery na API local
- a Shopee ja possui workflow manual no n8n para discovery de candidatos
- a Shopee ja usa a guia `Categorias` do painel para dirigir discovery por categoria, e nao mais apenas por `query` livre

### Estado atual da vertical Shopee

Atualizacao consolidada em `2026-04-18`:

- foi criada a pasta `scrapers/shopee/` no mesmo estilo estrutural da vertical Mercado Livre
- a Shopee agora possui:
  - `scrapers/shopee/discovery.py`
  - `scrapers/shopee/page_scraper.py`
  - `scrapers/shopee/export_cookies.py`
  - `scrapers/shopee/import_chrome_cookies.py`
  - `scrapers/shopee/video_url_scraper.py`
- o discovery da Shopee ja esta integrado em `api/server.py`
- existem endpoints ativos para Shopee:
  - `POST /api/discovery/shopee/precheck`
  - `POST /api/discovery/shopee`
- o workflow manual da Shopee no n8n ja foi criado e importado, espelhando o fluxo do Mercado Livre para candidatos
- o workflow manual da Shopee ja esta operacional no mesmo padrao do Mercado Livre:
  - le categorias habilitadas
  - respeita `per_category_limit`
  - chama precheck
  - roda discovery
  - opcionalmente enriquece com `page_scraper`
  - publica em `candidates.json`

### Categorias da Shopee

Decisao e estado atual:

- a Shopee nao ficou mais dependente de `query` fixa no workflow
- a origem Shopee agora usa a guia `Categorias` do painel para discovery, no mesmo estilo operacional do Mercado Livre
- categorias reais da Shopee foram importadas para `discovery_category_config.json`
- esse mesmo arquivo de configuracao tambem foi publicado no repo remoto do painel

Estado consolidado da configuracao:

- `30` categorias da Shopee foram importadas
- `10` categorias ficaram habilitadas por padrao
- `per_category_limit = 2` foi definido como base inicial
- o workflow e a API leem essas categorias habilitadas e respeitam o limite por categoria
- o campo `limit` hardcoded foi removido do workflow manual da Shopee
- a quantidade da rodada agora vem prioritariamente da propria configuracao da guia `Categorias`

### Regras operacionais atuais da descoberta Shopee

Atualizacao consolidada em `2026-04-18`:

- o discovery da Shopee agora aplica `min_sales = 100` por padrao
- a intencao e evitar promover produtos com historico muito fraco de vendas
- esse filtro esta ativo em:
  - `scrapers/shopee/discovery.py`
  - `api/server.py`
  - `n8n/workflows/shopee_discovery_candidates_manual.json`
- o precheck e o endpoint principal retornam estatisticas relacionadas a esse filtro:
  - `min_sales`
  - `low_sales_skipped`

Regra nova importante de robustez contra duplicidade:

- o discovery da Shopee nao para mais na primeira pagina de resultados
- se a pagina inicial vier com produtos duplicados ou abaixo de `min_sales`, ele continua buscando proximas paginas da Affiliate API
- a busca continua ate:
  - preencher a cota util da categoria
  - esgotar as paginas disponiveis
  - ou atingir o limite de seguranca de paginas
- as estatisticas agora tambem expõem:
  - `pages_fetched`
  - `offers_fetched`
  - `duplicates_skipped`
  - `low_sales_skipped`

### Estrategia atual da Shopee para discovery e scraping

Decisao operacional consolidada:

- a Shopee deve usar a `Affiliate API` como fonte principal para discovery e sinais comerciais
- o page scraper da Shopee nao deve duplicar o que a API ja fornece bem
- o scraper da Shopee deve complementar apenas o que falta

Na pratica, hoje a Shopee esta dividida assim:

Dados que devem vir prioritariamente da `Shopee Affiliate API`:

- `productLink`
- `offerLink`
- `imageUrl` principal
- `priceMin`
- `priceMax`
- `ratingStar`
- `sales`
- `commissionRate`
- `shopId`
- `shopName`

Dados que o scraper interno da Shopee complementa:

- galeria de imagens do produto
- videos do produto
- descricao textual completa do item
- `reviews_count`
- `discount_text`
- `main_features`

### Mudanca importante na politica de cookies da Shopee

Decisao operacional aprovada:

- o fallback de abrir a pagina da Shopee em browser automatizado foi removido
- o fluxo `--login` via Playwright foi descontinuado para Shopee
- a razao e que a Shopee nao permite de forma confiavel login em navegadores controlados por codigo

Fluxo aceito a partir de agora:

1. exportar cookies validos do navegador real
2. importar esses cookies para `data/shopee_cookies.json`
3. usar a API interna da Shopee com esse arquivo

Importante:

- se os cookies estiverem ausentes, o scraper falha com erro claro
- se os cookies estiverem expirados ou invalidos, a chamada interna retorna erro claro
- nao existe mais fallback para abrir browser controlado por codigo

Atualizacao importante de `2026-04-18`:

- a linha de `Chrome real + CDP` foi testada e depois abandonada por decisao operacional do projeto
- o motivo e manter a Shopee sem qualquer dependencia de navegador gerenciado
- o estado atual volta a ser estritamente `HTTP + cookies importados`
- o scraper funcional da Shopee foi reorientado para `HTTP puro`, usando:
  - cookies exportados do navegador real
  - `curl_cffi` com impersonacao de Chrome
  - extração do HTML SSR da pagina
- nesse desenho, o projeto ja conseguiu capturar com sucesso:
  - galeria de imagens
  - video do produto
  - descricao
  - `reviews_count`
  - `discount_text`
- o endpoint/fluxo antigo que dependia da API interna sujeita a `403` deixou de ser a referencia principal para captura de midia

### Importador de cookies do Chrome para Shopee

Nova capacidade adicionada:

- existe agora um importador dedicado para cookies exportados do Chrome
- arquivo:
  - `scrapers/shopee/import_chrome_cookies.py`

Formatos aceitos:

- JSON com lista de cookies
- JSON com objeto raiz contendo `cookies`
- arquivo Netscape `.txt`

Resultado esperado:

- o importador filtra apenas cookies da Shopee
- grava o arquivo final em `data/shopee_cookies.json`
- esse e o arquivo consumido pelo scraper da Shopee

### Validacao e testes recentes da Shopee

O estado atual foi reforcado por testes locais.

Cobertura consolidada:

- `tests/test_shopee_scraper.py`
- `tests/test_shopee_discovery.py`
- `tests/test_shopee_import_chrome_cookies.py`
- `tests/test_shopee_video_url_scraper.py`
- `tests/test_sync_candidates_from_discovery.py`
- `tests/test_api_server.py`

Validacoes ja realizadas nesta frente:

- estrutura da pasta `scrapers/shopee`
- mapeamento de candidatos da Shopee
- uso das categorias habilitadas da Shopee
- normalizacao de videos e imagens no `page_scrape`
- importacao de cookies exportados do Chrome
- remocao do fluxo de login automatizado da Shopee
- captura real de imagens e video da Shopee via HTTP puro
- normalizacao da saida da Shopee para o shape da guia `Candidatos`
- filtro de `100+` vendas no discovery da Shopee
- paginação automatica para continuar buscando apos duplicados

### O que ainda falta fazer na frente Shopee

Pendencias reais no estado atual:

- continuar monitorando se a captura HTTP da Shopee se mantem estavel com o mesmo formato de SSR
- validar em rodadas maiores se a paginação por categoria esta suficiente ou se o limite de paginas precisa ajuste
- revisar se vale adicionar suporte futuro a mais imagens da descricao, se isso realmente trouxer ganho de criativo
- expandir testes de integracao ponta a ponta para discovery real com cookies
- definir se a configuracao de Shopee na guia `Categorias` deve ganhar mais campos proprios alem de `per_category_limit` e `enabled`

Regra de produto que deve ficar explicita:

- a guia `Candidatos` nao deve ficar restrita ao Mercado Livre
- a intencao do projeto e que ela concentre candidatos de todas as contas e origens relevantes
- isso inclui, no minimo, Mercado Livre, Shopee, Shein e Amazon
- a filtragem por origem deve acontecer dentro da mesma guia, sem quebrar a experiencia em paginas separadas por conta

Regra de negocio importante:

- candidato nao deve ser reintroduzido facilmente se ja estiver descoberto, importado, publicado ou descartado

---

## Integracao Amazon — estado completo em 2026-04-20

### Visao geral

A Amazon foi integrada como a quarta plataforma do projeto, seguindo exatamente o mesmo padrao das demais (Mercado Livre, Shopee, Shein): scraper de discovery, endpoints na API local, workflow n8n, guia de candidatos, conta no painel e pagina publica no GitHub Pages.

### Cookies da Amazon

- os cookies da Amazon sao armazenados no Redis com a chave `cookies-amazon`
- essa chave ja era usada pelo workflow n8n `GERAR LINKS AFILIADO AMAZON` existente
- o scraper de discovery le os cookies dessa mesma chave via socket Redis direto (sem dependencia de biblioteca redis)
- a variavel de ambiente `AMAZON_REDIS_COOKIE_KEY` permite sobrescrever a chave, mas o padrao `cookies-amazon` e o correto

### Fluxo de geracao de link de afiliado

- ja existia o workflow n8n `GERAR LINKS AFILIADO AMAZON` (id: `rKigEEwVZw3rS2Xn`)
- esse workflow recebe `POST /webhook/afiliados-amazon` com body `{"url": "https://amazon.com.br/dp/<ASIN>"}`
- internamente ele le os cookies do Redis, monta a URL com tag `?tag=myshoplist0d-20` e chama a API do Amazon SiteStripe
- o scraper de discovery da Amazon chama esse webhook para cada candidato descoberto
- a URL padrao do webhook e configurada via variavel de ambiente `AMAZON_AFFILIATE_API_URL`; default: `http://localhost:5678/webhook/afiliados-amazon`

### Scraper de discovery

Arquivo: `scrapers/amazon/discovery.py`

O que o scraper faz:
- faz busca HTTP na pagina de resultados do Amazon.com.br (`/s?k=<query>&i=<search-alias>`)
- usa `curl_cffi` com impersonacao de Chrome 144 para evitar bloqueios
- parseia o HTML com `BeautifulSoup` e `lxml`
- extrai por produto: ASIN, titulo, thumbnail, galeria de imagens do srcset, preco atual, preco original, desconto calculado, rating, contagem de avaliacoes, badge de destaque, URL canonica
- detecta resposta de CAPTCHA e lanca erro claro com instrucao de renovar cookies
- cada categoria usa seu `id` (`search-alias`) como parametro `i=` da busca, filtrando por departamento com precisao
- o campo `query` da categoria e usado como termo de busca `k=`; se vazio, usa o `name` da categoria
- aplica filtro de `min_rating` (padrao `3.5`) para descartar produtos com avaliacao baixa
- suporta paginacao para preencher cotas por categoria
- deduplica por ASIN/source_url cruzando com candidatos ja existentes

Arquivos da vertical Amazon:
- `scrapers/amazon/__init__.py`
- `scrapers/amazon/discovery.py`
- `scrapers/amazon/output/` — pasta de saida para arquivos JSON locais

### Endpoints da API local

Dois endpoints novos em `api/server.py`:

`POST /api/discovery/amazon/precheck`
- valida categorias habilitadas
- verifica presenca de cookies no Redis
- verifica URL de afiliado configurada
- retorna projecao de limite, lista de categorias e avisos

`POST /api/discovery/amazon`
- executa o discovery completo por categoria
- aquece e reutiliza a sessao curl_cffi entre categorias
- chama o webhook de afiliado para cada candidato
- merge com `candidates.json` local
- publica no repo `painel` se `publish_panel=true`
- grava log em `logs/discovery_amazon_runs.jsonl`
- usa lock para evitar execucoes paralelas

Constantes e variaveis relevantes em `api/server.py`:
- `_amazon_discovery_lock` — threading.Lock() para execucoes exclusivas
- `DISCOVERY_AMAZON_RUN_LOG_PATH` — `logs/discovery_amazon_runs.jsonl`
- `AMAZON_AFFILIATE_API_URL` — URL do webhook de afiliado (env: `AMAZON_AFFILIATE_API_URL`)
- `DEFAULT_AMAZON_MIN_RATING` — 3.5

Parametros aceitos pelo endpoint `POST /api/discovery/amazon`:
- `source` (padrao: `"amazon"`)
- `limit` — limite total de candidatos
- `min_rating` — filtro de avaliacao minima
- `delay` — pausa entre categorias em segundos (padrao: 3)
- `local_categories_config_only` — usar apenas config local, sem GitHub
- `publish_panel` — publicar em `candidates.json` no painel (padrao: true)
- `with_affiliate` — chamar webhook de afiliado por produto (padrao: true)
- `affiliate_api_url` — sobrescrever URL do webhook
- `affiliate_delay_min_seconds`, `affiliate_delay_max_seconds`, `affiliate_max_retries`, `affiliate_retry_backoff_seconds`

### Workflow n8n

Arquivo: `n8n/workflows/amazon_discovery_candidates_manual.json`

Nome no n8n: `IAvideos - Discovery Amazon para Candidatos`
ID importado: `TewRkTbLRZjMeYQq`

Nos do workflow:
1. `Executar Manualmente` — trigger manual
2. `Config Discovery Amazon` — define parametros: source, min_rating, delay, local_categories_config_only, publish_panel, with_affiliate, affiliate_api_url
3. `Health API Local` — verifica se a API local esta de pe
4. `Precheck Discovery Amazon` — chama `POST /api/discovery/amazon/precheck`
5. `Validar Precheck` — lanca erro se precheck falhar, loga avisos de cookies
6. `Discovery Amazon -> Painel` — chama `POST /api/discovery/amazon` com timeout de 30 minutos
7. `Resumo Discovery` — loga estatisticas: categorias processadas, produtos buscados, descobertos, duplicados, affiliate_ready
8. `Fim` — NoOp

Diferenca em relacao ao workflow da Shein:
- nao tem etapa de validacao de sessao (Amazon nao exige warmup de sessao como a Shein)
- fluxo mais simples e direto

### Categorias da Amazon

As 40 categorias oficiais da Amazon.com.br foram importadas para `discovery_category_config.json` no repo remoto do painel em 2026-04-20.

Estado da importacao:
- 40 categorias no total
- 25 categorias habilitadas (produtos fisicos de consumo)
- 15 categorias desabilitadas (digitais, servicos, escopo amplo demais)
- `per_category_limit = 3` como base inicial
- o campo `id` de cada categoria e o `search-alias` da Amazon (ex: `electronics`, `home`, `beauty`, `hpc`)

Categorias habilitadas (25):
- Alimentos e Bebidas (`grocery`)
- Automotivo (`automotive`)
- Bebes (`baby`)
- Beleza (`beauty`)
- Beleza de luxo (`luxury-beauty`)
- Bolsas, Malas e Mochilas (`fashion-luggage`)
- Brinquedos e Jogos (`toys`)
- Casa (`home`)
- Computadores e Informatica (`computers`)
- Cozinha (`kitchen`)
- Eletrodomesticos (`appliances`)
- Eletronicos (`electronics`)
- Esportes e Aventura (`sporting`)
- Ferramentas e Materiais de Construcao (`hi`)
- Games (`videogames`)
- Instrumentos Musicais (`mi`)
- Jardim e Piscinas (`garden`)
- Livros (`stripbooks`)
- Material para Escritorio e Papelaria (`office-products`)
- Moveis e Decoracao (`furniture`)
- Pet Shop (`pets`)
- Roupas, Calcados e Joias (`fashion`)
- Roupas femininas (`fashion-womens`)
- Roupas masculinas (`fashion-mens`)
- Saude e Cuidados Pessoais (`hpc`)

Categorias desabilitadas (15):
- Todos os departamentos (`aps`), Alexa Skills, Amazon Quase Novo, Apps e Jogos, Audiolivros Audible, CD e Vinil, Dispositivos Amazon, DVD e Blu-Ray, Industrial, Loja Kindle, Prime Video, Programe e Poupe, Roupas para meninas, Roupas para meninos, Roupas para bebes

### Painel web — ajustes para Amazon

Alteracoes em `painel/index.html`:
- dropdown `Conta` no formulario de novo produto agora inclui `Amazon`
- `ACCOUNT_LABELS` agora inclui `amazon: 'Amazon'`
- `accountLabel` na renderizacao da tabela de produtos agora inclui `amazon: 'Amazon'`
- arquivo foi publicado no repo `painel` no GitHub Pages

### Conta no config de contas

Entrada `amazon` adicionada em `config/accounts.json`:
- `name`: `Prime Ofertas NET`
- `platform`: `amazon`
- `github_repo`: `am-afiliados`
- `facebook_store_url`: `https://melhoresachados.github.io/am-afiliados/`
- `watermark`: `@primeofertasnet`
- tema laranja Amazon: `accent: [255, 153, 0]`
- `post_times`: `["10:00", "19:00"]`
- `max_posts_per_day`: 2
- hashtags: `#amazon`, `#amazonbrasil`, `#achadosamazon`, `#ofertaamazon`, `#comprasazon`, `#amazonprime`

### Pagina publica GitHub Pages — am-afiliados

Repositorio: `https://github.com/melhoresachados/am-afiliados`
URL publica: `https://melhoresachados.github.io/am-afiliados/`

Arquivos publicados:
- `index.html` — pagina de catalogo no padrao das demais (Shopee/ML/Shein)
- `data.json` — catalogo inicial vazio com `account_name: "Prime Ofertas NET"` e `theme_color: "#FF9900"`

Identidade visual da pagina:
- cor principal: `#FF9900` (laranja Amazon)
- cor clara: `#FFB347`
- cor escura: `#E47911`
- botao de produto: "Ver na Amazon" (diferente de "Ver Produto")
- tab popular: "Mais Vendidos" (diferente de "Ofertas Populares")
- placeholder de imagem: emoji 🛒
- nome no cabecalho: "Prime Ofertas NET"
- subtitulo: "As melhores ofertas da Amazon!"

### Macrotemas da Amazon no sistema de inferencia

O metodo `infer_candidate_macrotheme` em `scrapers/mercadolivre/discovery.py` ja reconhece a conta `amazon` e classifica:
- gadgets → `"Gadgets confiaveis"`
- lifestyle/fitness/viagem/pet → `"Lifestyle utilitario"`
- casa/cozinha/home → `"Casa e organizacao"`
- default → `"Produtos evergreen orientados a review"`

### Workflows n8n existentes da Amazon (anteriores a esta integracao)

Dois workflows ja existiam antes desta integracao e continuam ativos:

1. `EXTRAIR LINKS AMAZON` (id: `pySlGXLiqUBLXFMP`)
   - usa browserAct (Apify) para navegar na Amazon
   - le categorias do Google Sheets
   - grava resultados no Google Sheets
   - nao foi alterado e nao e necessario para o fluxo atual

2. `GERAR LINKS AFILIADO AMAZON` (id: `rKigEEwVZw3rS2Xn`)
   - webhook: `POST /webhook/afiliados-amazon`
   - body: `{"url": "https://amazon.com.br/dp/<ASIN>"}`
   - le cookies do Redis (`cookies-amazon`)
   - monta URL com tag `?tag=myshoplist0d-20`
   - chama Amazon SiteStripe API para encurtar URL
   - retorna URL curta de afiliado
   - este workflow e chamado pelo discovery da Amazon para cada candidato

### O que ainda falta na frente Amazon

Pendencias conhecidas:
- validar se o scraper HTML consegue parsed corretamente as paginas de busca em producao (Amazon muda layout)
- testar rodada completa de discovery com categorias reais e cookies validos
- monitorar CAPTCHA: se aparecer em categorias especificas, pode ser necessario rotacionar delay ou mudar ordem de categorias
- definir se o campo `query` das categorias deve ser preenchido com termos especificos (ex: "organizador de gaveta" para `home`) ou se o nome generico da categoria e suficiente
- avaliar se vale adicionar suporte a paginas de mais vendidos (`/bestsellers/`) como fonte alternativa ao search
- criar testes automatizados para `scrapers/amazon/discovery.py`
- apos primeiras rodadas, ajustar `per_category_limit` e selecao de categorias habilitadas com base nos resultados reais

## Modelo semantico de produto

O projeto consolidou dois campos com papeis diferentes:

- `name`: titulo original completo
- `display_name`: nome curto para thumbnail, overlays e contexto visual

Tambem existe uma camada dedicada de normalizacao comercial em:

- `enrichment/commerce_normalization.py`

Uso dessa camada:

- encurtar desconto no visual
- encurtar vendidos no visual
- preservar formato mais natural para contexto e narracao

## Midia e geracao

Decisoes consolidadas da esteira de geracao:

- entrada manual de midia foi consolidada em torno de URLs publicas
- a operacao principal nao deve depender de caminhos locais como entrada
- thumbnail e video precisam ser validados antes de publicar
- o gerador de video foi refinado para melhorar encaixe de titulo, capa e composicao visual
- a duracao do video acompanha o audio quando audio existe
- os `12` segundos sao fallback, nao regra fixa

## Estrategia editorial por conta

Diretriz editorial consolidada:

- o projeto nao deve operar todas as contas como vitrines totalmente genericas e indistintas
- a descoberta de produtos pode continuar ampla na guia `Candidatos`
- a publicacao deve ser organizada por macrotemas por conta
- essa separacao existe para melhorar coerencia editorial, sinal para o algoritmo, expectativa do publico e reaproveitamento de identidade visual

### Regra geral

- `Candidatos` deve aceitar produtos de todas as origens
- cada candidato deve ser classificado por `origem` e tambem por `macrotema`
- sempre que possivel, o destino da publicacao deve respeitar o macrotema principal da conta
- excecoes podem existir, mas a regra padrao e preservar coerencia editorial

### Estrategia recomendada

#### Shein

Macrotema principal:

- `Moda e Beleza`

Papel da conta:

- foco em vestuario feminino
- acessorios femininos
- itens de estilo, composicao de look e apelo visual
- beleza adjacente ao universo feminino da conta

Leitura pratica:

- esta deve ser a conta mais claramente editorializada do projeto
- a conta da Shein nao deve virar vitrine geral de utilidades ou eletronicos aleatorios
- o objetivo e consolidar uma identidade de look, estilo, feminilidade comercial e descoberta visual

#### Shopee

Macrotemas principais:

- `Achadinhos virais`
- `Casa e utilidades`
- `Beleza acessivel`
- `Gadgets leves e acessorios de impulso`

Papel da conta:

- funcionar como conta de maior elasticidade editorial
- absorver produtos de ticket mais impulsivo e forte apelo de video curto
- servir como conta mais voltada a volume, testes e repertorio viral

Leitura pratica:

- Shopee pode ser a conta mais ampla entre as operacoes atuais
- mesmo assim, o centro de gravidade deve ficar em produtos demonstraveis, baratos, uteis, curiosos ou visualmente satisfatorios
- evitar transformar a conta em catalogo caotico sem linha de descoberta

## Atualizacao Tecnica - 2026-04-18

### Shein

Estado atual:

- `quickView` da Shein funciona em HTTP puro com:
  - cookies exportados do navegador
  - `x-csrf-token`
  - `x-ad-flag`
- o `page_scraper` individual da Shein ja funciona por `goods_id` ou URL de produto
- a `source_url` canonica do produto ja esta sendo montada corretamente
- a descoberta automatica da listagem ainda pode cair em `risk/challenge`, entao o gargalo atual continua sendo obter os `goods_id` da categoria

Categorias:

- a secao `shein` de `discovery_category_config.json` ja foi preenchida com um catalogo inicial de categorias top-level
- esse catalogo inclui `id`, `slug`, `url` e `macrotheme`
- defaults ativos hoje priorizam:
  - `Roupas femininas`
  - `Moda praia`
  - `Sapatos`
  - `Tamanhos Grandes`
  - `Joias e acessorios`
  - `Pijamas e roupas intimas`
  - `Beleza e saude`
  - `Bolsas e malas`
  - `Local`

Base de codigo criada:

- `scrapers/shein/categories.py`
  - catalogo inicial de categorias da Shein
  - helper para montar `listing_url` por categoria
- `scripts/import_shein_discovery_categories.py`
  - atualiza `discovery_category_config.json` com as categorias da Shein
- `scrapers/shein/discovery.py`
  - agora aceita `--category-id`, `--child-cat-id` e `--page`
  - resolve a URL de listagem a partir do catalogo local quando necessario

#### Mercado Livre

Macrotemas principais:

- `Casa e utilidades`
- `Gadgets e eletronicos`
- `Ferramentas e itens de resolucao de problema`
- `Ofertas de alto valor percebido`

Papel da conta:

- explorar produtos em que comparacao de preco, desconto, beneficio e funcionalidade tenham peso maior
- aproveitar a forca de discovery por categoria e page scraping ja existente no projeto
- favorecer itens com prova visual de utilidade, desempenho ou economia

Leitura pratica:

- Mercado Livre nao deve copiar a identidade da Shein
- aqui faz mais sentido destacar beneficio, oportunidade, performance, praticidade e achado forte
- produtos de casa, organizacao, automacao leve, ferramentas e gadgets tendem a conversar melhor com essa conta

#### Amazon

Macrotemas principais:

- `Casa e organizacao`
- `Gadgets confiaveis`
- `Lifestyle utilitario`
- `Produtos evergreen orientados a review`

Papel da conta:

- funcionar como frente de produtos com apelo mais confiavel, pesquisavel e evergreen
- privilegiar itens que se beneficiam de review, prova de uso, comparacao e reputacao de marketplace
- servir como conta mais forte para conteudo de recomendacao utilitaria, presenteavel e duradoura

Leitura pratica:

- quando a conta Amazon estiver operacional, ela nao precisa competir com a Shopee em impulso barato
- a melhor estrategia tende a ser produtos com percepcao de qualidade, utilidade recorrente e busca mais intencional

### Resumo estrategico do portfolio

- `Shein`: conta mais nichada e visual, centrada em `Moda e Beleza`
- `Shopee`: conta mais viral, ampla e orientada a impulso
- `Mercado Livre`: conta mais forte em utilidade, oferta e funcionalidade
- `Amazon`: conta mais evergreen, review-driven e orientada a confianca

### Implicacao operacional para a guia Candidatos

- a guia `Candidatos` deve ter um campo explicito de `macrotema`
- esse campo deve complementar a `origem` e a `conta`, nao substitui-las
- nao faz sentido adicionar `conta sugerida de destino`, porque os produtos ja entram na guia com a conta definida
- a classificacao por `macrotema` deve ajudar a decidir importacao, priorizacao e consistencia editorial dentro da propria conta
- a operacao pode continuar ampla na descoberta, mas deve ficar mais disciplinada na distribuicao por conta

## Automacao e n8n

Workflows relevantes documentados:

- `n8n/workflows/pipeline_automatico.json`
- `n8n/workflows/staged_pipeline.json`
- `n8n/workflows/pipeline_full_visibility.json`
- `n8n/workflows/pipeline_full_visibility_force.json`
- `n8n/workflows/thumbnail_manual.json`
- `n8n/workflows/force_publish_manual.json`
- `n8n/workflows/mercadolivre_discovery_candidates_manual.json`
- `n8n/workflows/shopee_discovery_candidates_manual.json`
- `n8n/workflows/shein_discovery_candidates_manual.json`
- `n8n/workflows/amazon_discovery_candidates_manual.json`

Leitura operacional consolidada:

- o fluxo detalhado com mais visibilidade existe
- modos especiais foram separados para evitar poluir o fluxo principal
- `force_publish` e thumbnail manual foram tratados como operacoes explicitamente separadas

## Comandos essenciais

Ambiente:

- `./.venv/Scripts/python.exe -m pipeline.runner --painel`
- `./.venv/Scripts/python.exe -m api.server`
- `./.venv/Scripts/python.exe -m unittest discover -s tests -v`
- `./.venv/Scripts/python.exe -m scrapers.shopee.import_chrome_cookies --input <arquivo_exportado>`

Operacoes recorrentes:

- processar painel completo
- gerar apenas audio
- gerar apenas thumbnail
- gerar sem publicar
- publicar so Facebook por `product_id`
- enviar alerta de teste ao Telegram
- importar workflows do n8n
- sincronizar catalogos publicos

## Cobertura de testes encontrada

Existe uma base real de testes automatizados no repositorio.

Arquivos encontrados em `tests/` indicam cobertura para:

- API (`test_api_server.py`, `test_api_panel_sync.py`)
- pipeline (`test_cli.py`, `test_input_flow.py`, `test_panel_pipeline.py`, `test_product_flow.py`, `test_publish_flow.py`, `test_runtime_utils.py`)
- normalizacao e copy (`test_normalizer.py`, `test_caption_builder.py`, `test_commerce_normalization.py`, `test_display_name.py`)
- validacao de midia e fila (`test_media_validation.py`, `test_queue_manager.py`)
- Mercado Livre (`test_mercadolivre_discovery.py`, `test_mercadolivre_page_scraper.py`)
- Shopee (`test_shopee_scraper.py`, `test_shopee_discovery.py`, `test_shopee_import_chrome_cookies.py`)
- token diagnostics e Telegram (`test_token_diagnostics.py`, `test_telegram_alerts.py`)
- audio (`test_tts_generator.py`)
- video (`test_video_generator.py`, `test_video_cover_fit.py`, `test_video_cover_title.py`, `test_video_proof_copy.py`, `test_video_scene_plan.py`)

Conclusao pratica:

- o projeto nao esta mais sem testes
- ainda vale expandir cobertura de integracao ponta a ponta

## Riscos e prioridades mais consistentes

Pontos que aparecem como prioridade real nas documentacoes:

- autenticacao da API Flask
- evoluir estrategia de token/expiracao
- continuar modularizacao do pipeline
- fortalecer monitoramento proativo
- manter documentacao sincronizada com o codigo

Prioridade media recorrente:

- alternativa mais estavel ao ngrok
- validacao ainda mais forte de scraping
- organizacao continua de legado e documentacao

## Decisoes operacionais importantes

- o painel continua sendo a fonte de verdade
- GitHub Pages continua fazendo parte do produto final, nao apenas da documentacao
- quando for preciso alterar algo publicado no GitHub Pages, a expectativa operacional e aplicar a mudanca e publicar o arquivo remoto como procedimento padrao
- artefatos estritamente locais devem permanecer nao versionaveis por padrao
- a automacao deve privilegiar estabilidade antes de paralelismo amplo

## Arquivos de apoio que continuam valendo

Mesmo com este arquivo unificado, estes documentos seguem uteis como referencia detalhada:

- `PROJECT_CONTEXT.md`
- `COMMANDS.md`
- `REVIEW_EXTERNAL_ASSESSMENT.md`
- `PLATFORM_EDITORIAL_GUIDE.md`

## Leitura final

Se alguem novo entrar no projeto, a interpretacao mais fiel hoje e:

1. este nao e mais um experimento solto; e uma esteira operacional real
2. o coracao do sistema e painel + banco + pipeline + publicacao social
3. a camada de candidatos do Mercado Livre virou frente concreta de expansao
4. o codigo ja evoluiu em modularizacao e testes, mas ainda ha reforcos importantes de seguranca e operacao
5. a guia `Candidatos` deve ser entendida como uma camada unificada multi-origem, nao como uma tela exclusiva do Mercado Livre
6. este arquivo deve ser tratado como ponto de entrada unico, e os outros como detalhamento
