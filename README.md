# Sistema de Chamados de Manutenção — IFMS Campus Jardim

Protótipo de TCC para automatizar o processo de comunicação de problemas de
infraestrutura entre alunos, servidores e o setor de manutenção do campus.

> **Projeto completo — todas as 6 etapas do roadmap concluídas.**
> Veja `DESIGN_SYSTEM.md` para os tokens visuais definidos.

## Estrutura do repositório

```
ifms-manutencao/
├── backend/     # API REST (Node + Express + Prisma + SQLite)
└── frontend/    # SPA (React + Vite + TailwindCSS)
```

### Backend

```
backend/src/
├── config/         # configuração de infraestrutura (ex: cliente Prisma)
├── controllers/     # recebem a requisição HTTP e devolvem a resposta
├── services/         # regras de negócio
├── repositories/       # acesso a dados (Prisma)
├── middlewares/          # auth, tratamento de erros, upload, etc.
└── routes/                 # definição dos endpoints
```

### Frontend

```
frontend/src/
├── components/   # componentes reutilizáveis (Button, Card, Badge...)
├── pages/         # telas (Login, Dashboard, Chamados...)
├── layouts/        # esqueletos de página (sidebar + topbar, etc.)
├── hooks/            # hooks customizados
├── services/          # chamadas HTTP à API
├── contexts/            # estado global (ex: autenticação)
├── routes/                # configuração de rotas (react-router)
├── utils/                  # funções auxiliares
└── types/                    # tipos/JSDoc compartilhados
```

## Pré-requisitos

- Node.js 18 ou superior
- npm

## Como rodar o backend

```bash
cd backend
npm install
cp .env.example .env
npx prisma migrate dev --name init
npx prisma db seed
npm run dev
```

A API sobe em `http://localhost:3333`. Teste em `http://localhost:3333/api/health`.

Usuários criados pelo seed:

| Papel | E-mail | Senha |
|---|---|---|
| Administrador | admin@ifms.edu.br | admin123 |
| Usuário | aluno@ifms.edu.br | usuario123 |

O seed também cria 14 chamados de exemplo (variando status, categoria e
prioridade) para o Dashboard já nascer com dados reais para exibir.

## Como rodar o frontend

Em outro terminal:

```bash
cd frontend
npm install
npm run dev
```

A aplicação sobe em `http://localhost:5173`. As chamadas para `/api` são
redirecionadas automaticamente para o backend (ver `vite.config.js`).

Se tudo estiver certo, a tela inicial deve mostrar "Conexão com a API: OK".

## Endpoints disponíveis (Etapa 2)

| Método | Rota | Descrição | Autenticação |
|---|---|---|---|
| GET | `/api/health` | Verifica se a API está no ar | Não |
| POST | `/api/auth/login` | Autentica com `{ email, senha }` e devolve `{ usuario, token }` | Não |
| GET | `/api/auth/me` | Devolve os dados do usuário autenticado | Sim (Bearer token) |
| GET | `/api/dashboard/resumo` | Indicadores, distribuições por status/categoria/prioridade, chamados recentes e atividades | Sim (Bearer token) |
| GET | `/api/locais` | Lista os ambientes ativos do campus | Sim (Bearer token) |
| GET | `/api/locais/todos` | Lista todos os locais (inclusive inativos), com bloco e código | Sim, apenas Administrador |
| GET | `/api/locais/codigo/:codigo` | Resolve um local a partir do código do QR Code | Sim (Bearer token) |
| POST | `/api/locais` | Cadastra um novo local e gera o código do QR Code | Sim, apenas Administrador |
| GET | `/api/chamados` | Lista chamados com filtros (`status`, `categoria`, `prioridade`, `localId`, `busca`), paginação (`page`, `pageSize`) e ordenação (`ordenarPor`, `ordem`). Usuário comum só vê os próprios chamados; administrador vê todos | Sim (Bearer token) |
| POST | `/api/chamados` | Abre um novo chamado. `multipart/form-data` com `titulo`, `descricao`, `categoria`, `prioridade`, `localId` e um campo opcional `imagem` (até 5MB) | Sim (Bearer token) |
| GET | `/api/chamados/:id` | Detalhes completos do chamado (anexos, observações, histórico). Usuário comum só pode ver o próprio chamado | Sim (Bearer token) |
| PATCH | `/api/chamados/:id` | Atualiza `status`, `prioridade` e/ou `responsavelId`. Gera histórico e notificações internas automaticamente | Sim, apenas Administrador |
| POST | `/api/chamados/:id/observacoes` | Adiciona uma observação ao chamado e notifica o solicitante | Sim, apenas Administrador |
| GET | `/api/usuarios/administradores` | Lista administradores ativos (usado no select de "Responsável") | Sim, apenas Administrador |
| GET | `/api/notificacoes` | Lista as notificações do usuário logado (`?apenasNaoLidas=true` para filtrar) | Sim (Bearer token) |
| GET | `/api/notificacoes/nao-lidas/contagem` | Contagem de notificações não lidas (usado no sininho do Topbar) | Sim (Bearer token) |
| PATCH | `/api/notificacoes/:id/lida` | Marca uma notificação como lida | Sim (Bearer token) |
| PATCH | `/api/notificacoes/lidas` | Marca todas as notificações do usuário como lidas | Sim (Bearer token) |
| PATCH | `/api/usuarios/me` | Atualiza o próprio nome | Sim (Bearer token) |
| PATCH | `/api/usuarios/me/senha` | Troca a própria senha (`senhaAtual`, `novaSenha`) | Sim (Bearer token) |

## Funcionalidade: notificações internas

Sempre que o administrador altera o **status**, atribui um **responsável** ou
adiciona uma **observação** em um chamado, uma notificação interna é criada
automaticamente para a pessoa afetada (o solicitante ou o novo responsável —
nunca para quem fez a própria alteração). Elas aparecem no sininho do Topbar,
com contagem de não lidas atualizada a cada 30s, e ao clicar levam direto
para o chamado correspondente.

## Funcionalidade: abertura de chamado por QR Code

Cada local cadastrado tem um **código único** (gerado a partir do nome, ex:
`LABORATORIO-01`) e um QR Code que aponta para `/chamados/novo/:codigo`.

Fluxo: o usuário escaneia o QR fixado no ambiente → cai direto na tela de
"Novo chamado" já com o **local identificado automaticamente** → só precisa
escolher a **categoria**, escrever a **descrição** e, opcionalmente, anexar
uma **foto**. Título e prioridade são preenchidos automaticamente (a
prioridade pode ser reclassificada pelo administrador depois, na triagem).

Se o usuário ainda não estiver logado, o próprio fluxo de autenticação já
existente cuida disso: ele é enviado para o login e, ao entrar, volta
automaticamente para a tela do chamado — nenhuma rota nova de auth foi
necessária para isso.

A tela de **cadastro de locais e impressão dos QR Codes** fica em
`/locais`, visível apenas para o Administrador (link "Locais" no menu
lateral). Os QR Codes são gerados inteiramente no navegador (biblioteca
`qrcode`, sem chamada a nenhuma API externa) e podem ser impressos
individualmente para fixar no ambiente.

> **Atenção ao migrar:** o campo `codigo` do model `Local` é obrigatório e
> único. Se você já tinha um `dev.db` de uma etapa anterior, apague-o antes
> de rodar a nova migração:
> ```bash
> # dentro de backend/
> rm prisma/dev.db
> rm -rf prisma/migrations
> npx prisma migrate dev --name add_qrcode_locais
> npx prisma db seed
> ```

## Stack técnica

- **Frontend:** React + Vite, TailwindCSS, React Router, Recharts (gráficos), Lucide React (ícones), qrcode (geração de QR Codes 100% local)
- **Backend:** Node.js, Express, Prisma ORM
- **Banco de dados:** SQLite (protótipo — arquitetura preparada para migrar para PostgreSQL/MySQL no futuro)
- **Autenticação:** JWT (implementada na Etapa 2)

## Roadmap de etapas

1. ✅ Fundação do projeto (estrutura, banco de dados, setup base)
2. ✅ Autenticação e identidade visual (login, JWT, layout com sidebar/topbar, design system aplicado)
3. ✅ Dashboard (indicadores, gráficos por status/categoria/prioridade, chamados recentes, atividades)
4. ✅ Abertura e listagem de chamados (formulário com anexo de imagem, filtros, busca, paginação)
5. ✅ Detalhes e edição de chamado (status/prioridade/responsável, observações, linha do tempo, notificações internas)
6. ✅ Perfil (editar nome, trocar senha), Configurações (conta, atalhos, sobre o sistema) e refinamentos finais (menu lateral responsivo com drawer mobile)
