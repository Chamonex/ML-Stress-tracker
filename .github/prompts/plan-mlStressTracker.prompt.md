# Plan: Backend API para Google Calendar

Backend Node.js com Express que autentica via Service Account para ler eventos da sua agenda pessoal do Google. A API expõe endpoints REST para consultar eventos (título, horário, duração) em períodos específicos. Escolhemos Service Account pela simplicidade (sem fluxo OAuth), ideal para automação backend de uso pessoal. A estrutura modular facilita futuras expansões para análise de stress via ML.

## Steps

1. **Configurar Google Cloud Console**
   - Acessar [console.cloud.google.com](https://console.cloud.google.com)
   - Criar novo projeto "ML-Stress-Tracker"
   - Habilitar Google Calendar API em "APIs & Services > Library"
   - Criar Service Account em "Credentials > Create Credentials > Service Account"
   - Baixar chave JSON (será `service-account-key.json`)
   - **Importante:** Copiar o email da service account (formato: `nome@projeto.iam.gserviceaccount.com`)
   - Compartilhar sua Google Agenda com esse email (dar permissão "Ver todos os detalhes dos eventos")

2. **Inicializar projeto Node.js**
   - Criar `package.json` com `npm init -y`
   - Instalar dependências principais: `googleapis` (client oficial do Google), `express` (servidor HTTP), `dotenv` (variáveis de ambiente)
   - Instalar dependências de desenvolvimento: `nodemon` (auto-reload durante desenvolvimento)
   - Configurar scripts no `package.json`: `start` (node), `dev` (nodemon)

3. **Estruturar arquivos do projeto**
   - Criar `src/config/google-auth.js` - carrega service account key e cria client autenticado do Google Calendar API
   - Criar `src/config/environment.js` - centraliza variáveis de ambiente (caminho da chave, calendar ID, porta)
   - Criar `src/services/calendar-service.js` - lógica de negócio para buscar eventos, com parâmetros de data início/fim e tratamento de paginação (API retorna max 250 eventos)
   - Criar `src/routes/calendar-routes.js` - define endpoints Express (GET `/events`, GET `/events/today`, GET `/events/week`)
   - Criar `src/utils/error-handler.js` - middleware para capturar erros da API do Google (401, 403, 404, 429) e retornar respostas HTTP apropriadas
   - Criar `src/index.js` - entry point que inicializa Express, registra rotas e inicia servidor

4. **Configurar gerenciamento de credenciais**
   - Criar arquivo `.env` na raiz com variáveis: `GOOGLE_SERVICE_ACCOUNT_KEY_PATH`, `CALENDAR_ID=primary`, `PORT=3000`
   - Criar `.gitignore` incluindo: `.env`, `service-account-key.json`, `node_modules/`, `*.log`
   - Adicionar `service-account-key.json` na raiz (arquivo baixado do Google Cloud)

5. **Implementar autenticação Google**
   - Em `google-auth.js`: usar `google.auth.GoogleAuth` com keyFile e scope `https://www.googleapis.com/auth/calendar.readonly`
   - Criar função que retorna instância autenticada do `google.calendar({ version: 'v3', auth })`
   - Implementar tratamento de erro se arquivo de chave não existir ou for inválido

6. **Implementar Calendar Service**
   - Em `calendar-service.js`: criar função `getEvents(timeMin, timeMax)` que chama `calendar.events.list()`
   - Configurar parâmetros: `calendarId: 'primary'`, `timeMin/timeMax` (ISO 8601), `singleEvents: true` (expande eventos recorrentes), `orderBy: 'startTime'`
   - Extrair e formatar dados relevantes: `{ id, summary, start: {dateTime, timeZone}, end: {dateTime, timeZone}, duration }` (calcular duração)
   - Implementar paginação iterativa usando `pageToken` para >250 eventos

7. **Criar endpoints REST**
   - Em `calendar-routes.js`: definir router Express
   - `GET /events?start=YYYY-MM-DD&end=YYYY-MM-DD` - range customizado, valida query params
   - `GET /events/today` - eventos do dia atual (usa date-fns ou luxon para calcular início/fim do dia)
   - `GET /events/week` - próximos 7 dias
   - Cada endpoint chama `calendar-service.getEvents()` e retorna JSON

8. **Implementar error handling**
   - Em `error-handler.js`: criar middleware `errorHandler(err, req, res, next)`
   - Mapear códigos de erro da API: 401 → "Credenciais inválidas", 403 → "Permissão negada (verifique compartilhamento)", 404 → "Calendário não encontrado", 429 → "Limite de requisições excedido"
   - Registrar erros com `console.error` incluindo stack trace
   - Retornar JSON estruturado: `{ error: true, message, code, timestamp }`

9. **Configurar servidor Express**
   - Em `index.js`: importar express, criar app, aplicar `express.json()` middleware
   - Registrar rotas: `app.use('/api/calendar', calendarRoutes)`
   - Aplicar error handler como último middleware
   - Iniciar servidor: `app.listen(PORT)` com log de confirmação
   - Implementar graceful shutdown (capturar SIGTERM/SIGINT)

10. **Documentar uso no README**
    - Atualizar `README.md` com: descrição do projeto, pré-requisitos (Node.js 18+, conta Google), instrução de setup (passos do Google Cloud Console), configuração (.env template), instalação (`npm install`), execução (`npm run dev`), exemplos de requisições (curl/Postman para cada endpoint)

## Verification

1. **Setup**: Verificar que `service-account-key.json` existe e está no `.gitignore`
2. **Instalação**: Executar `npm install` sem erros
3. **Autenticação**: Executar servidor com `npm run dev` - deve iniciar sem erros de autenticação
4. **Endpoint básico**: `curl http://localhost:3000/api/calendar/events/today` - deve retornar JSON com eventos de hoje ou array vazio
5. **Endpoint com range**: `curl "http://localhost:3000/api/calendar/events?start=2026-02-01&end=2026-02-28"` - deve retornar eventos de fevereiro
6. **Error handling**: Testar com credenciais inválidas ou calendário não compartilhado - deve retornar erro 403 com mensagem clara
7. **Logs**: Verificar console para requisições e possíveis erros da API

## Decisions

- **Service Account over OAuth 2.0**: Para uso pessoal, elimina complexidade do fluxo de consentimento do usuário. Requer apenas compartilhar calendário uma vez.
- **Express over CLI/Script**: Mantém aplicação rodando, permite fácil integração futura com frontend ou pipelines ML via HTTP.
- **Scope readonly**: Segurança - aplicação só lê dados. Se precisar escrever no futuro, alterar scope para `calendar.events`.
- **primary calendar ID**: Simplifica início. Suporte a múltiplos calendários pode ser adicionado depois aceitando `calendarId` como query param.
- **Estrutura modular**: Separa responsabilidades (auth, service, routes) facilitando testes e manutenção quando expandir para ML.
