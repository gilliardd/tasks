# 12 Week Year Tracker

**Meta anual não funciona porque doze meses é tempo demais para sentir urgência.**
Este sistema aplica a metodologia *12 Week Year*: ciclos de doze semanas, execução medida todo dia
e cobrança automática por Telegram — sem precisar abrir o sistema para registrar o dia.

![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![Material UI](https://img.shields.io/badge/Material%20UI-007FFF?style=flat-square&logo=mui&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram%20Bot-26A5E4?style=flat-square&logo=telegram&logoColor=white)

---

## O que ele faz de diferente

**Você não digita a meta diária — o sistema calcula.**
Informe o total a alcançar, a unidade, sua velocidade por hora e quanto tempo por dia você tem.
A partir disso o sistema deriva quanto precisa ser feito hoje. Quatro tipos de métrica são
suportados: sim/não, páginas, horas e unidade personalizada.

**Perdeu um dia? A meta se reajusta sozinha.**
A redistribuição recalcula o alvo diário como *quanto falta ÷ dias restantes* — contando apenas os
dias da semana em que aquela tarefa realmente roda. O atraso vira número novo em vez de virar culpa,
e dá para reajustar uma tarefa ou todas as tarefas ativas do ciclo de uma vez.

**A cobrança chega até você.**
Cada tarefa tem seu horário de notificação. Um job agendado monta a fila do dia, dispara a mensagem
no Telegram e recebe a resposta ali mesmo — o registro do dia acontece na conversa.

**Cada tarefa vale só nos dias certos.**
Treino três vezes por semana e leitura todo dia convivem no mesmo ciclo, com metas diárias
diferentes calculadas sobre calendários diferentes.

---

## Como está organizado

```
Visão            objetivo de longo prazo
 └── Período     ciclo de doze semanas, com data de início e fim
      └── Meta   o que precisa ser alcançado neste ciclo
           └── Tática        como a meta vai ser atingida
                └── Tarefa   o que é executado no dia a dia
                     └── Log registro diário de execução
```

No fim de um ciclo, o período inteiro pode ser clonado para o seguinte, sem remontar a estrutura.

O painel fecha o ciclo de volta: resumo do período, gráfico de execução semanal, progresso por meta
e calendário de histórico.

---

## Decisões técnicas que valem nota

**Dias da semana em bitmask.** Cada tarefa guarda seus dias num único `TINYINT`
(`Dom=1, Seg=2, Ter=4, Qua=8, Qui=16, Sex=32, Sáb=64`; `127` = todos). Uma coluna em vez de sete,
e a consulta de "quais tarefas valem hoje" vira uma operação de bit.

**Log diário idempotente.** `UNIQUE (task_id, log_date)` no banco — a mesma tarefa não pode ser
registrada duas vezes no mesmo dia, mesmo com clique duplo ou resposta repetida no Telegram.
A regra é do banco, não do código.

**Integridade em cascata.** Toda a hierarquia usa `ON DELETE CASCADE`: apagar uma visão não deixa
período, meta, tática, tarefa ou log órfão para trás.

**Falha ruidosa na partida.** Sem conexão com o banco o processo encerra em vez de subir quebrado.
Já o bot do Telegram é opcional: sem o token, o sistema sobe normalmente e apenas o desativa.

**Um processo em produção.** Em produção o backend serve o build do frontend, então o deploy é
um único processo, com uma porta e um vhost.

---

## Rodando localmente

Requer Node.js e um MySQL acessível.

```bash
npm run install:all          # dependências da raiz, do backend e do frontend
cp .env.example .env         # preencha banco, JWT_SECRET, admin e token do bot
npm run migrate              # cria as tabelas
npm run dev                  # backend :3000 e frontend :3001
```

Em produção: `npm run build` e depois `npm start`.
A API responde sob `/api`, com `/api/health` para checagem.

---

<details>
<summary><b>Referência — variáveis de ambiente, estrutura e API</b></summary>

### Variáveis de ambiente

| Variável | Para que serve |
|---|---|
| `PORT` · `NODE_ENV` | Porta do servidor e ambiente |
| `DB_HOST` · `DB_PORT` · `DB_USER` · `DB_PASSWORD` · `DB_NAME` | Conexão com o MySQL |
| `JWT_SECRET` · `JWT_EXPIRES_IN` | Assinatura e validade do token de sessão |
| `ADMIN_USERNAME` · `ADMIN_PASSWORD` | Credenciais do usuário inicial |
| `TELEGRAM_BOT_TOKEN` | Token do bot, criado no @BotFather |
| `TZ` | Fuso horário usado pelo agendamento |

### Estrutura

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

### API

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

</details>
