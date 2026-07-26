# Chatbot Telegram de Temperatura (n8n)

Atividade avaliativa da Pós em IA e Automação (Rocketseat), módulo de n8n.

O usuário manda o nome de uma cidade brasileira para o bot no Telegram (formato `Cidade,UF,BR`), o workflow consulta a API da OpenWeather e responde com a temperatura atual da cidade. Um node opcional de Google Gemini reescreve a mensagem final de forma mais natural, com fallback determinístico caso o Gemini falhe ou não tenha credencial configurada.

## Fluxo do workflow

```
Telegram Trigger
  → Tratar Cidade (Set: normaliza o texto e guarda o chat_id)
  → OpenWeather (HTTP Request)
  → Cidade Encontrada? (IF)
      ├─ Sim → Mensagem Sucesso (Code, determinística)
      │          → Gemini Reescrever Mensagem
      │               ├─ sucesso → Extrair Mensagem Gemini → Enviar Sucesso
      │               └─ erro    → Usar Fallback (Gemini Falhou) → Enviar Sucesso
      └─ Não → Mensagem Erro (Code) → Enviar Erro
```

## Pré-requisitos

- Instância n8n (self-hosted ou cloud)
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)
- API key da [OpenWeather](https://openweathermap.org/api) (free tier)
- (Opcional) API key do [Google AI Studio](https://aistudio.google.com/) para o node Gemini

## Como importar o workflow

1. No n8n, vá em **Workflows → Import from File**.
2. Selecione o arquivo `workflow-chatbot-telegram.json` deste repositório.
3. O workflow será importado **inativo** e com os campos de credencial vazios (nenhum token vem incluso no JSON).

## Configurando as credenciais

Este workflow usa **Credentials do n8n** para guardar os segredos (em vez de variáveis de ambiente), o que faz o JSON ser 100% portável entre instâncias sem expor nenhuma chave.

### 1. Telegram API

1. Fale com o [@BotFather](https://t.me/BotFather) no Telegram, rode `/newbot`, escolha nome e username (precisa terminar em `bot`) e guarde o token retornado.
2. No n8n: **Credentials → New → Telegram API**.
3. Cole o token no campo **Access Token**.
4. Nos nodes **Telegram Trigger**, **Enviar Sucesso** e **Enviar Erro**, selecione essa credencial.

### 2. OpenWeather API

1. Crie uma conta gratuita em [openweathermap.org](https://openweathermap.org/api) e gere uma API key em "My API keys".
2. No n8n: **Credentials → New → Query Auth**.
3. Preencha:
   - **Name**: `appid`
   - **Value**: sua chave da OpenWeather
4. No node **OpenWeather** (HTTP Request), em Authentication, selecione **Generic Credential Type → Query Auth** e escolha essa credencial. A chave é injetada automaticamente como parâmetro `appid` na chamada — nunca aparece em texto no node.

> Por que Query Auth e não `$env.OPENWEATHER_API_KEY`? A instância usada para construir e testar este workflow é Community Edition e não tem o recurso *Variables* liberado (`feat:variables` bloqueado por licença). A credencial Query Auth cumpre o mesmo papel — nenhuma chave hardcoded no node — sem depender de configuração no servidor.

### 3. Google Gemini (opcional)

1. Gere uma API key em [Google AI Studio](https://aistudio.google.com/).
2. No n8n: **Credentials → New → Google Gemini(PaLM) Api**. Cole a chave.
3. No node **Gemini Reescrever Mensagem**, selecione essa credencial.
4. Se você não tiver credencial Gemini configurada (ou o node falhar por qualquer motivo), o workflow **não quebra**: o node tem `onError: continueErrorOutput`, e o fluxo cai automaticamente no node **Usar Fallback (Gemini Falhou)**, que reenvia a mesma mensagem determinística gerada pelo node **Mensagem Sucesso** (sem IA). O avaliador pode rodar o workflow inteiro sem essa credencial.

## Variáveis/segredos necessários (resumo)

| Nome                     | Onde é usado                          | Como configurar                         |
|--------------------------|----------------------------------------|------------------------------------------|
| `TELEGRAM_BOT_TOKEN`     | Credencial **Telegram API**            | Gerado via @BotFather                    |
| `OPENWEATHER_API_KEY`    | Credencial **Query Auth** (`appid`)    | Gerado em openweathermap.org             |
| Google AI Studio API key | Credencial **Google Gemini(PaLM) Api** | Opcional — só para o node Gemini         |

## Ativando o workflow

Depois de configurar as três credenciais (ou só as duas obrigatórias), ative o workflow com o toggle **Active** no canto superior direito do editor.

## Como testar

Envie mensagens diretamente para o seu bot no Telegram, no formato `Cidade,UF,BR`:

- `São Paulo,SP,BR` → `🌤️ A temperatura em São Paulo é de 20°C.` (ou reescrita pelo Gemini, se ativo)
- `Belo Horizonte,MG,BR` → `🌤️ A temperatura em Belo Horizonte é de 25°C.`
- `Curitiba,PR,BR` → `🌤️ A temperatura em Curitiba é de 16°C.`
- Uma cidade inexistente (ex.: `Xyzabc123`) → `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

Os valores de temperatura variam conforme o clima real no momento do teste.

## Observação sobre o node de Telegram do n8n

Por padrão, o node **Telegram** do n8n acrescenta uma atribuição automática ("This message was sent automatically with n8n") às mensagens enviadas. Nos nodes **Enviar Sucesso** e **Enviar Erro**, esse comportamento foi desativado em **Additional Fields → Append n8n Attribution → off**, para manter a formatação exata exigida pela atividade.
