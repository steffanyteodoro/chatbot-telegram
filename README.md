# 🌤️ Chatbot de Temperatura no Telegram (N8N)

Chatbot do Telegram que informa a **temperatura atual de qualquer cidade do Brasil**.
O usuário envia o nome da cidade (no formato `Cidade,UF,BR`), o fluxo consulta a API
gratuita da **OpenWeather**, processa a resposta e devolve uma mensagem curta, clara e
amigável com a temperatura, a sensação térmica e a umidade.

Exemplo de resposta:

> 🌤️ A temperatura em Belo Horizonte é de 25°C.
> 🌡️ Sensação térmica: 26°C | Céu limpo
> 💧 Umidade: 60%

---

## ✨ Funcionalidades

- Recebe mensagens de texto via **Telegram Trigger**.
- Normaliza a entrada (remove acentos, espaços extras e converte para minúsculas).
- Consulta a OpenWeather em graus Celsius e em português (`pt_br`).
- Valida a resposta (status HTTP) e trata o caso de **cidade não encontrada**.
- Monta a mensagem de forma **determinística** (também serve como *fallback*).
- Envia a resposta formatada de volta ao usuário no Telegram.

---

## 🧩 Estrutura do Workflow

A ordem dos nós é a seguinte:

1. **Telegram Trigger** — dispara o fluxo ao receber uma mensagem de texto.
2. **Formatar Entrada** (Set) — cria as variáveis:
   - `queue`: texto normalizado da cidade (usado na consulta);
   - `chat_id`: id do chat para responder;
   - `raw_text`: texto original enviado pelo usuário.
3. **Consultar OpenWeather** (HTTP Request) — faz `GET` em
   `https://api.openweathermap.org/data/2.5/weather` com os parâmetros de query:
   - `q` → `{{ $json.queue }}` (cidade normalizada)
   - `units` → `metric` (Celsius)
   - `lang` → `pt_br`
   - `appid` → `{{ $env.OPENWEATHER_API_KEY }}` (variável de ambiente, **sem chave embutida**)
4. **Resposta Válida?** (IF) — verifica se o `statusCode` é `200`.
   - **Verdadeiro** → segue para a extração dos dados.
   - **Falso** → envia a mensagem de erro.
5. **Extrair e Formatar Dados** (Code) — extrai a temperatura, arredonda o valor,
   escolhe um emoji conforme a faixa de temperatura e monta a `message` final.
   *Esta é a geração determinística da mensagem.*
6. **Fallback Gemini** (Code) — repassa `message` e `ok`. É o ponto de **fallback**:
   se você adicionar um nó do Google Gemini (opcional), este Code node garante a
   mesma mensagem mesmo sem credenciais do Gemini.
7. **Enviar Resposta ao Usuário** (Telegram) — envia a mensagem formatada.
8. **Enviar Mensagem de Erro** (Telegram) — em caso de cidade inválida, envia:
   `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

---

## ✅ Pré-requisitos

- Uma instância do **N8N** rodando (cloud ou local).
- Um **bot do Telegram** criado via [@BotFather](https://t.me/BotFather) e o respectivo token.
- Uma conta na **OpenWeather** com uma **API Key** ativa
  (criada em <https://home.openweathermap.org/api_keys>).

---

## 🔐 Variáveis e credenciais esperadas

Este projeto **não contém nenhuma credencial real**. Você precisa configurar:

| Variável                | Onde é usada                              | Como configurar |
|-------------------------|-------------------------------------------|-----------------|
| `OPENWEATHER_API_KEY`   | Nó *Consultar OpenWeather* (parâmetro `appid`) | Variável de ambiente do N8N |
| `TELEGRAM_BOT_TOKEN`    | Credencial do Telegram no N8N             | Cadastrado como credencial `telegramApi` |

> **Importante:** nunca suba a chave da OpenWeather nem o token do Telegram para o
> repositório. No JSON exportado, o `appid` está como `{{ $env.OPENWEATHER_API_KEY }}`
> justamente para evitar isso.

---

## 📥 Como importar o workflow no N8N

1. Abra o N8N.
2. No menu superior direito, clique em **⋯ → Import from File**
   (ou na tela de Workflows: **Add workflow → Import from File**).
3. Selecione o arquivo **`workflow-chatbot-telegram.json`** deste repositório.
4. O fluxo será importado com todos os nós e conexões.

---

## ⚙️ Configurando as credenciais

### 1) Telegram (`TELEGRAM_BOT_TOKEN`)

1. No N8N, vá em **Credentials → Add credential → Telegram API**.
2. No campo **Access Token**, cole o token fornecido pelo BotFather.
3. Salve a credencial.
4. Abra os nós **Telegram Trigger**, **Enviar Resposta ao Usuário** e
   **Enviar Mensagem de Erro** e selecione essa credencial em cada um deles.

### 2) OpenWeather (`OPENWEATHER_API_KEY`)

A chave é lida de uma **variável de ambiente**, não de uma credencial fixa.
Defina a variável no ambiente que roda o N8N:

**Local / terminal:**
```bash
export OPENWEATHER_API_KEY="sua_chave_aqui"
```

**Docker / docker-compose** (exemplo no bloco abaixo):
```yaml
environment:
  - OPENWEATHER_API_KEY=sua_chave_aqui
```

> Caso o N8N não consiga ler a variável via `$env`, verifique se o acesso a variáveis
> de ambiente em expressões está habilitado (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`).

---

## ▶️ Como executar e testar

1. Garanta que a variável `OPENWEATHER_API_KEY` está definida e que as credenciais
   do Telegram estão configuradas.
2. **Ative** o workflow (botão *Active* no canto superior direito).
3. No Telegram, abra uma conversa com o seu bot e envie uma cidade. Exemplos:

| Você envia            | Retorno esperado |
|-----------------------|------------------|
| `São Paulo,SP,BR`     | 🌤️ A temperatura em São Paulo é de XX°C. (+ sensação e umidade) |
| `Belo Horizonte,MG,BR`| 🌤️ A temperatura em Belo Horizonte é de XX°C. |
| `Curitiba,PR,BR`      | 🌤️ A temperatura em Curitiba é de XX°C. |
| `Cidadeinexistente,XX,BR` | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |

> Os valores de temperatura variam conforme o horário e a condição do clima.

---

## 🤖 Nó opcional do Google Gemini

O fluxo já contém o nó **Fallback Gemini** (Code), que garante a mensagem
determinística mesmo sem credenciais do Gemini.

Para usar o Gemini de fato:

1. Adicione um nó **Google Gemini** **entre** *Extrair e Formatar Dados* e
   *Fallback Gemini*.
2. Configure as credenciais do Gemini em **Credentials → Add credential**.
3. Use **temperatura baixa (0–0.2)** e instrua a saída em português, em formato
   simples, por exemplo:
   ```json
   {"message": "...", "ok": true}
   ```
4. Mantenha o nó **Fallback Gemini** ligado logo em seguida: se o Gemini falhar ou
   não tiver credenciais, a mensagem determinística é usada como *fallback*.

---

## 🐳 (Opcional) Rodando o N8N com Docker

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - OPENWEATHER_API_KEY=sua_chave_aqui
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=false
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Suba com:
```bash
docker compose up -d
```

E acesse o N8N em <http://localhost:5678>.

---

## 🔒 Segurança

- Nenhuma chave da OpenWeather ou token do Telegram está embutido neste repositório.
- A API Key da OpenWeather é injetada via variável de ambiente `OPENWEATHER_API_KEY`.
- O token do Telegram fica armazenado apenas como credencial dentro do N8N.
