# Automação de Ofertas Shopee para Grupos de WhatsApp

Sistema automatizado que busca produtos mais vendidos na **Shopee Affiliate Open API (GraphQL)**, aplica filtros inteligentes de comissão e volume de vendas, gera links curtos rastreáveis com `subIds` e envia os anúncios formatados com foto em grupos do WhatsApp via **n8n** e **Evolution API v2.3.7**.

## Arquitetura do Sistema

O fluxo no n8n é composto por **3 nós**:

```text
[ Schedule Trigger ] ──► [ Code: Montar Mensagem ] ──► [ HTTP Request (Evolution API) ]
                                  │
                                  ├── Assinatura SHA-256 + Query GraphQL (productOfferV2)
                                  ├── Sorteio ponderado de categorias + sazonal (Natal)
                                  ├── Filtro por faixa de preço, comissão, nota e vendas
                                  ├── Memória anti-repetição (últimos 1.000 itemIds)
                                  └── Geração de ShortLink com subIds dinâmicos
```

### Estrutura de Pastas

```text
shopee-whatsapp-ofertas/
├── docs/
│   └── GUIA_WORKFLOW_N8N.md      # Passo a passo técnico dos 3 nós no n8n
├── scripts/
│   └── montar-mensagem.js        # Código JavaScript do nó Code
├── .env.example                  # Modelo público de variáveis de ambiente
├── .gitignore                    # Bloqueia envio de senhas (.env) e volumes locais ao Git
├── docker-compose.yml            # Infraestrutura Docker (n8n, Evolution API, Postgres e Redis)
└── README.md                     # Manual de instalação, conexão WhatsApp e deploy
```

### Funcionalidades Principais
* **Curadoria Automática por Faixa de Preço:** Exige comissões e volumes mínimos proporcionais ao ticket do produto (ex.: itens até R$ 100 exigem comissão ≥ 7% e ≥ 300 vendas; nota mínima de `4.5`).
* **Rotação de Categorias com Pesos:** Alterna entre 12 categorias regulares (casa, cozinha, beleza, moda, pet, auto/moto, tecnologia, ferramentas, fitness, infantil e ticket alto) e categorias sazonais configuráveis (`CHANCE_NATAL`).
* **Anti-Repetição Nativo:** Utiliza `$getWorkflowStaticData('global')` do próprio n8n para armazenar os últimos 1.000 `itemId` enviados, sem depender de planilhas ou bancos externos.
* **Rastreamento Avançado (`subIds`):** Gera links curtos (`generateShortLink`) etiquetados com origem (`whatsapp`), hora do disparo (`h14`), palavra-chave, categoria e faixa de preço.

---

## Pré-requisitos

* **Docker** e **Docker Compose** instalados.
* Conta ativa no **Programa de Afiliados da Shopee** com credenciais da Open API (`AppID` e `AppSecret`) liberadas.
* Número de WhatsApp secundário (recomendado) para atuar como disparador no grupo.

---

## Como Obter as Credenciais da Shopee (Open API)

1. Acesse o portal de afiliados em `https://affiliate.shopee.com.br/open_api` com sua conta logada.
2. No canto superior direito, abra a **Central de Ajuda**, role até o final da página e clique em **Fale conosco por e-mail**.
3. Preencha o formulário informando:
   * Seu **ID de Afiliado** (exibido no painel superior).
   * O telefone cadastrado na conta.
   * **Tema:** *Cenário em relação ao cadastro/conta* → **Quero ativar a API**.
4. Após a aprovação (prazo médio de até 2 semanas), a Shopee enviará por e-mail seu **`AppID`** e **`AppSecret`**.

---

## Instalação e Inicialização

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/SEU_USUARIO/shopee-whatsapp-ofertas.git
   cd shopee-whatsapp-ofertas
   ```

2. **Crie e configure o arquivo `.env`:**
   ```bash
   cp .env.example .env
   ```
   Abra o `.env` e preencha:
   * `SHOPEE_APP_ID` e `SHOPEE_SECRET` com suas chaves da Shopee.
   * `POSTGRES_PASSWORD` e `AUTHENTICATION_API_KEY` com senhas fortes *(evite usar o caractere `$` ou escreva `$$` para evitar falhas de leitura no Docker Compose)*.

3. **Suba os containers:**
   ```bash
   docker compose up -d
   ```

4. **Verifique o status dos serviços:**
   ```bash
   docker compose ps
   ```
   Aguarde até que `postgres` e `redis` estejam `healthy` e `n8n` e `evolution-api` estejam `Up`.

---

## Como Conectar o WhatsApp (Evolution Manager)

1. Acesse o painel web da Evolution API:
   ```text
   http://localhost:8085/manager
   ```
2. Faça login informando:
   * **Server URL:** `http://localhost:8085`
   * **API Key:** O valor exato definido em `AUTHENTICATION_API_KEY` no seu `.env`.
3. Clique em **+ Instance** para criar uma instância:
   * **Name:** `ofertas-shopee` (ou outro nome sem espaços).
   * **Token:** Deixe em branco para geração automática e clique em **Save**.
4. Abra a instância criada, clique em **Get QR Code** (ou **Connect**) e escaneie o código pelo aplicativo do WhatsApp no celular (*Configurações > Aparelhos Conectados > Conectar um aparelho*).
5. Aguarde o status mudar para **Connected** (verde).

---

## Como Descobrir o ID do Grupo (`@g.us`)

Com o WhatsApp conectado na instância, execute uma requisição `GET` (via navegador, cURL, Postman ou pelo próprio n8n) para:

```bash
curl -X GET "http://localhost:8085/group/fetchAllGroups/ofertas-shopee?getParticipants=false" \
     -H "apikey: SUA_AUTHENTICATION_API_KEY"
```

Localize o nome do seu grupo (`"subject"`) na resposta JSON e copie o valor do campo `"id"` correspondente (formato `120363000000000000@g.us`).

---

## Configuração do Fluxo no n8n

1. Acesse `http://localhost:5678` e crie sua conta administrativa local.
2. Siga o guia completo em [`docs/GUIA_WORKFLOW_N8N.md`](docs/GUIA_WORKFLOW_N8N.md) para montar os **3 nós** do fluxo e colar o código de [`scripts/montar-mensagem.js`](scripts/montar-mensagem.js).

---

## Publicando no GitHub

Para enviar o projeto ao seu repositório público mantendo as credenciais protegidas:

```bash
git init
git add .gitignore .env.example docker-compose.yml README.md docs/ scripts/
git status   # Confirme que o arquivo .env NÃO aparece na lista
git commit -m "chore: estrutura inicial da automacao Shopee + n8n + Evolution API"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/shopee-whatsapp-ofertas.git
git push -u origin main
```