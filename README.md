# Chatbot Telegram de Temperatura (n8n)

Atividade avaliativa da Pós em IA e Automação (Rocketseat), módulo de n8n.

O usuário manda o nome de uma cidade brasileira pro bot no Telegram (formato `Cidade,UF,BR`), o workflow consulta a API da OpenWeather e responde com a temperatura atual. Tem um node opcional de Google Gemini que reescreve a mensagem final de forma mais natural, com fallback pra mensagem determinística caso o Gemini falhe ou não tenha credencial configurada.

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
- API key do [Google AI Studio](https://aistudio.google.com/), se quiser usar o node Gemini

## Importando o workflow

No n8n, vá em Workflows → Import from File e selecione o `workflow-chatbot-telegram.json` deste repositório. Ele importa inativo e sem nenhuma credencial preenchida (o JSON não tem token nenhum embutido).

## Configurando as credenciais

Aqui usei Credentials do n8n pra guardar os segredos, em vez de variável de ambiente. Assim o JSON fica portável entre instâncias sem expor chave nenhuma.

**Telegram API**

1. Fala com o [@BotFather](https://t.me/BotFather), roda `/newbot`, escolhe nome e username (tem que terminar em `bot`) e guarda o token que ele retorna.
2. No n8n: Credentials → New → Telegram API, cola o token em Access Token.
3. Seleciona essa credencial nos nodes Telegram Trigger, Enviar Sucesso e Enviar Erro.

**OpenWeather API**

1. Cria uma conta grátis em [openweathermap.org](https://openweathermap.org/api) e gera a API key em "My API keys".
2. No n8n: Credentials → New → Query Auth. Name = `appid`, Value = a chave.
3. No node OpenWeather (HTTP Request), em Authentication, escolhe Generic Credential Type → Query Auth e seleciona essa credencial. A chave é injetada automaticamente como parâmetro `appid` na chamada.

Usei Query Auth em vez de `$env.OPENWEATHER_API_KEY` porque a instância que usei pra montar e testar isso é Community Edition e não tem o recurso de Variables liberado (licença bloqueia). A credencial resolve o mesmo problema (chave fora do node) sem precisar mexer em variável de ambiente do servidor.

**Google Gemini (opcional)**

1. Gera uma API key em [Google AI Studio](https://aistudio.google.com/).
2. No n8n: Credentials → New → Google Gemini(PaLM) Api, cola a chave.
3. Seleciona essa credencial no node Gemini Reescrever Mensagem.

Se não tiver credencial do Gemini configurada (ou o node der erro por qualquer motivo), o workflow continua funcionando: ele cai no node Usar Fallback (Gemini Falhou), que manda a mesma mensagem determinística do node Mensagem Sucesso, sem IA nenhuma. Dá pra rodar o workflow inteiro sem essa credencial.

## Resumo dos segredos usados

- `TELEGRAM_BOT_TOKEN` → credencial Telegram API, gerado no @BotFather
- `OPENWEATHER_API_KEY` → credencial Query Auth (`appid`), gerado em openweathermap.org
- API key do Google AI Studio → credencial Google Gemini(PaLM) Api, opcional

## Ativando o workflow

Depois de configurar as credenciais (pelo menos as duas obrigatórias), ativa o workflow no toggle Active lá no canto superior direito do editor.

## Testando

Manda mensagem direto pro bot no Telegram, no formato `Cidade,UF,BR`:

- `São Paulo,SP,BR` → `🌤️ A temperatura em São Paulo é de 20°C.` (ou reescrita pelo Gemini, se tiver ativo)
- `Belo Horizonte,MG,BR` → `🌤️ A temperatura em Belo Horizonte é de 25°C.`
- `Curitiba,PR,BR` → `🌤️ A temperatura em Curitiba é de 16°C.`
- Cidade inexistente (ex.: `Xyzabc123`) → `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

Os valores de temperatura mudam de acordo com o clima real na hora do teste.
