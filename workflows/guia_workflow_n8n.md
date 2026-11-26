# Guia de Implementação do Workflow no n8n (3 Nós)

O fluxo completo opera com **3 nós** conectados em sequência:

```text
[ Schedule Trigger ] ──► [ Montar Mensagem (Code) ] ──► [ HTTP Request (Evolution API) ]
```

---

## Nó 1: `Schedule Trigger`

Define os horários em que o robô dispara ofertas automaticamente ao longo do dia.

* **Trigger Rules:** `Custom (Cron)` (ou regras de dias/horas)
* **Exemplo de expressão Cron (4x ao dia):** `0 9,12,16,20 * * *` *(executa às 09:00, 12:00, 16:00 e 20:00)*
* **Exemplo de expressão Cron (12x ao dia para 1 produto por vez):** `0 8-20 * * *`
* **Timezone:** `America/Manaus` (herdado automaticamente do container via `GENERIC_TIMEZONE`).

---

## Nó 2: `Montar Mensagem` (Node `Code`)

Adicione um nó **Code** (linguagem `JavaScript`, modo `Run Once for All Items`) e cole o conteúdo completo do arquivo [`scripts/montar-mensagem.js`](../scripts/montar-mensagem.js).

> **Credenciais via Ambiente:** O script lê `SHOPEE_APP_ID` e `SHOPEE_SECRET` diretamente das variáveis configuradas no `.env` e injetadas pelo `docker-compose.yml` (`$env.SHOPEE_APP_ID` e `$env.SHOPEE_SECRET`), evitando que chaves fiquem expostas no código do workflow.

### Principais Parâmetros Ajustáveis no Topo do Script

| Constante | Padrão | Função |
| :--- | :--- | :--- |
| `MAX_TENTATIVAS` | `5` | Quantas buscas aleatórias tentar por execução até achar um produto qualificado. |
| `QTD` | `1` | Quantidade de produtos retornados por execução do trigger. |
| `MIN_NOTA` | `4.5` | Avaliação mínima (estrelas) exigida do produto. |
| `CHANCE_NATAL` | `0.4` | Probabilidade (`0.0` a `1.0`) de sortear categorias sazonais de Natal (`0` desativa fora de época). |
| `USAR_SUBIDS` | `true` | Gera link curto (`generateShortLink`) com tags de rastreio (`whatsapp`, hora, termo, categoria, faixa). |
| `FAIXAS` | Array | Define comissão mínima e vendas mínimas conforme o preço do produto. |

### Campos Retornados pelo Nó

Para cada produto aprovado, o nó entrega um item JSON contendo:
* `texto`: Copy formatada com título, preço em `R$`, nota, vendas e link de afiliado.
* `imagem`: URL da imagem principal do produto (`imageUrl`).
* `ganhoEstimado`: Valor estimado da comissão em reais.
* `linkTipo`: `'curto'` (quando gerado com `subIds`) ou `'longo'` (fallback `offerLink`).
* `subIds`: Etiquetas aplicadas ao link para análise no relatório da Shopee.

---

## Nó 3: `HTTP Request` (Disparo para o WhatsApp)

Conecte um nó **HTTP Request** logo após o nó `Montar Mensagem` para enviar a imagem com legenda ao grupo via Evolution API v2.3.7.

* **Method:** `POST`
* **URL:**
  ```text
  http://evolution-api:8080/message/sendMedia/NOME_DA_SUA_INSTANCIA
  ```
  *(Como o n8n e a Evolution API rodam na mesma rede Docker, utilize o nome do serviço `evolution-api:8080` em vez de `localhost:8085`).*

* **Authentication:** `Generic Credential Type` → `Header Auth`
  * **Name:** `apikey`
  * **Value:** Mesmo valor de `AUTHENTICATION_API_KEY` definido no `.env`.

* **Send Body:** Ativado (`Body Content Type: JSON` / `Specify Body: Using JSON`)
  ```json
  {
    "number": "123363000000000000@g.us",
    "mediatype": "image",
    "media": "={{ $json.imagem }}",
    "caption": "={{ $json.texto }}"
  }
  ```

### Proteção Anti-Ban para Envios em Lote (`QTD > 1`)

Se você alterar `QTD` para `3` no código para enviar 3 produtos na mesma execução, ative o intervalo entre disparos para evitar bloqueios no WhatsApp:
1. Abra a aba **Settings** dentro do nó `HTTP Request`.
2. Ative a opção **Batching**.
3. Configure **Items per Batch:** `1` e **Batch Interval (ms):** `12000` *(aguarda 12 segundos entre cada produto postado no grupo)*.