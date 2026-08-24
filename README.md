# 12 Week Year Tracker

Sistema de acompanhamento de metas baseado na metodologia **12 Week Year**: em vez de planejar o ano
inteiro, você trabalha em ciclos de 12 semanas, com execução medida todo dia.

O sistema quebra a meta em tarefas diárias, calcula sozinho quanto você precisa fazer por dia,
recalcula esse número quando você atrasa e cobra a execução por um bot do Telegram.

---

## Como o sistema organiza as metas

```
Visão            objetivo de longo prazo
 └── Período     ciclo de 12 semanas, com data de início e fim
      └── Meta   o que precisa ser alcançado neste ciclo
           └── Tática        como a meta vai ser atingida
                └── Tarefa   o que é executado no dia a dia
                     └── Log registro diário de execução
```

Cada nível é apagado em cascata junto com o pai (`ON DELETE CASCADE`), e o log diário tem chave
única por tarefa e data — a mesma tarefa não pode ser registrada duas vezes no mesmo dia.

---

## Funcionalidades

**Tarefas com quatro tipos de métrica**
`boolean` (fez ou não fez), `pages`, `hours` e `custom`. Nas métricas quantitativas você informa o
total a alcançar, a unidade, a velocidade por hora e o tempo diário disponível — a meta diária é
calculada a partir disso, não digitada.

**Redistribuição automática**
Quando um dia é perdido, a meta diária é recalculada como *quanto falta ÷ dias restantes no período*,
considerando apenas os dias da semana em que a tarefa está configurada para rodar. Pode ser disparada
para uma tarefa ou para todas as tarefas ativas do período de uma vez.

**Agenda por dia da semana**
Cada tarefa define em quais dias ela vale, através de um bitmask (`Dom=1, Seg=2, Ter=4, Qua=8,
Qui=16, Sex=32, Sáb=64` — o padrão `127` significa todos os dias).

**Bot do Telegram**
Cada tarefa tem seu horário de notificação (padrão 20:00). Um job agendado monta a fila de
notificações, envia a cobrança do dia e recebe a resposta pelo próprio Telegram — o registro do dia
pode ser feito sem abrir o sistema.

**Painel de acompanhamento**
Resumo do período, gráfico semanal de execução, progresso por meta, calendário de histórico e
clonagem de um período inteiro para o ciclo seguinte.

---

## Stack

**Backend** Node.js · Express · MySQL · JWT · bcryptjs · express-validator · node-cron · node-telegram-bot-api
**Frontend** React 18 · Material UI 5 · Recharts · React Router 6 · Axios · date-fns

Em produção o backend serve o build do frontend, então tudo sobe como um processo só.

---

## Como rodar

**Pré-requisitos:** Node.js e um MySQL acessível.

```bash
# 1. instalar as dependências da raiz, do backend e do frontend
npm run install:all

# 2. configurar o ambiente
cp .env.example .env
#    edite o .env com os dados do banco, o JWT_SECRET, o usuário admin
#    e o token do bot do Telegram

# 3. criar as tabelas
npm run migrate

# 4. subir backend e frontend juntos
npm run dev
```

Backend em `http://localhost:3000`, frontend em `http://localhost:3001`.
A API responde sob o prefixo `/api` e tem um `/api/health` para checagem.

**Produção**

```bash
npm run build   # gera o build do frontend
npm start       # sobe o backend, que também serve o frontend
```

---

## Variáveis de ambiente

| Variável | Para que serve |
|---|---|
| `PORT` · `NODE_ENV` | Porta do servidor e ambiente |
| `DB_HOST` · `DB_PORT` · `DB_USER` · `DB_PASSWORD` · `DB_NAME` | Conexão com o MySQL |
| `JWT_SECRET` · `JWT_EXPIRES_IN` | Assinatura e validade do token de sessão |
| `ADMIN_USERNAME` · `ADMIN_PASSWORD` | Credenciais do usuário inicial |
| `TELEGRAM_BOT_TOKEN` | Token do bot, criado no @BotFather |
| `TZ` | Fuso horário usado pelo agendamento |

Sem `TELEGRAM_BOT_TOKEN` o sistema sobe normalmente e apenas desativa o bot.
Sem conexão com o banco, o processo encerra na inicialização em vez de subir quebrado.

---

## Estrutura

```
├── backend
│   ├── migrations        schema SQL + runner com controle de versão aplicada
│   └── src
│       ├── config        conexão com o banco
│       ├── controllers   auth, visions, periods, goals, tactics, tasks, logs, dashboard
│       ├── jobs          job de notificação agendada
│       ├── middleware    autenticação JWT
│       ├── routes        definição da API
│       └── services      cálculo, redistribuição, notificação e bot do Telegram
├── frontend
│   └── src
│       ├── components
│       ├── context
│       ├── pages         Login, Dashboard, Visions, Periods, Goals, Tactics, Tasks, History
│       └── services      cliente HTTP
└── 12-week-year-tracker-spec.md    especificação completa do sistema
```

---

## API

Todas as rotas exigem token JWT, exceto `POST /api/auth/login` e `GET /api/health`.

| Recurso | Rotas |
|---|---|
| Autenticação | `POST /auth/login` · `GET /auth/me` · `PUT /auth/password` |
| Visões | CRUD em `/visions` |
| Períodos | CRUD em `/periods` · `/periods/active` · `/periods/:id/summary` · `/periods/:id/clone` |
| Metas | CRUD em `/goals` · `/goals/:id/progress` |
| Táticas | CRUD em `/tactics` |
| Tarefas | CRUD em `/tasks` · `/tasks/today` · `/tasks/:id/logs` |
| Logs | `POST /tasks/:taskId/complete` · `POST /tasks/:taskId/skip` · `/logs/week/:weekNumber` · `/logs/calendar` |
| Dashboard | `/dashboard/summary` · `/dashboard/weekly-chart` · `/dashboard/goals-progress` |
