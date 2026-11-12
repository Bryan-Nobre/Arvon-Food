# Arvon Food

SaaS multi-tenant de gestão operacional para restaurantes — PDV, mesas e comandas, cozinha (KDS), caixa, estoque e administração da plataforma.

<p align="center">
  <img src="docs/screenshots/dashboard.png" alt="Dashboard do Arvon Food" width="820" />
</p>

---

## Visão geral

O Arvon Food organiza o dia a dia do restaurante em um único produto web: do pedido no balcão ou na mesa até a produção na cozinha, o recebimento no caixa e o controle de insumos.

Cada restaurante é uma **empresa** isolada. Usuários entram como **membros** com papéis definidos. Regras financeiras, estoque, permissões e isolamento entre tenants são garantidos no **PostgreSQL** (RLS e RPCs), não apenas na interface.

O código-fonte deste repositório permanece **privado**. Este README apresenta o produto como case de portfólio.

---

## Principais funcionalidades

- **PDV / Pedidos** — montagem de pedido no balcão ou vinculado a mesa/comanda, com catálogo, adicionais e estados operacionais e financeiros separados
- **Mesas e comandas** — mapa do salão, abertura de comanda, pedido de conta e transferência entre mesas
- **Cozinha (KDS)** — filas por setor, avanço de status e impressão de comanda pelo navegador
- **Caixa** — abertura/fechamento de sessão, sangria, suprimento e recebimentos (incluindo pagamento parcial)
- **Estoque** — movimentações auditáveis, alerta de baixo estoque e bipagem por leitor USB ou câmera
- **Cadastros** — produtos, categorias, adicionais, clientes, equipe e configurações da empresa
- **Convites de equipe** — entrada de membros por link (sem cadastro público de restaurante)
- **Admin da plataforma** — criação de empresas e logins, métricas leves, pausa/reativação e acesso temporário de suporte
- **Offline parcial** — fila local para abrir mesa, criar pedido e pedir conta; operações críticas exigem conexão
- **Realtime** — atualizações por empresa no salão, cozinha e caixa

---

## Destaques técnicos

| Tema | Abordagem no produto |
| ---- | -------------------- |
| Multi-tenancy | Isolamento por `empresa_id` com RLS e helpers de autorização no schema `app` |
| Fonte da verdade | Operações críticas via RPCs Postgres (`SECURITY DEFINER`), não só pelo React |
| Papéis | `owner`, `admin`, `cashier`, `waiter`, `kitchen` — UI filtra módulos; banco autoriza |
| Idempotência | `client_request_id` em ações offline e operações que não podem duplicar |
| Snapshots | Pedidos preservam nome/preço/adicionais do momento da venda |
| Tema | Interface clara com modo escuro |

---

## Stack tecnológica

| Camada | Tecnologias |
| ------ | ----------- |
| Frontend | React 19, TypeScript, Vite, TanStack Start / Router / Query, Tailwind CSS 4, Radix UI, React Hook Form, Zod, Lucide |
| Backend / dados | Supabase (Auth, PostgreSQL, Realtime, Storage), migrations SQL versionadas, RPCs e RLS |
| Offline / leitura | IndexedDB (fila local), Service Worker, ZXing (código de barras) |
| UX | Sonner, Recharts, impressão HTML 80mm via navegador |

---

## Principais interfaces

<p align="center"><strong>Login</strong></p>
<p align="center">
  <img src="docs/screenshots/login.png" alt="Tela de login" width="720" />
</p>
<p align="center"><em>Acesso com e-mail e senha. Cadastro aberto só para aceitar convite de equipe.</em></p>

<br />

<p align="center"><strong>Dashboard</strong></p>
<p align="center">
  <img src="docs/screenshots/dashboard.png" alt="Dashboard operacional" width="820" />
</p>
<p align="center"><em>Resumo do dia: vendas, ticket médio, cozinha e ocupação de mesas.</em></p>

<br />

<p align="center"><strong>Pedidos (PDV)</strong></p>
<p align="center">
  <img src="docs/screenshots/pedidos.png" alt="PDV e pedidos" width="820" />
</p>
<p align="center"><em>Catálogo, filtros de status e montagem de pedido (balcão, mesa ou comanda).</em></p>

<br />

<p align="center"><strong>Mesas</strong></p>
<p align="center">
  <img src="docs/screenshots/mesas.png" alt="Mapa de mesas" width="820" />
</p>
<p align="center"><em>Salão em lista e mapa, com status livre, ocupada e aguardando pagamento.</em></p>

<br />

<p align="center"><strong>Cozinha</strong></p>
<p align="center">
  <img src="docs/screenshots/cozinha.png" alt="KDS da cozinha" width="820" />
</p>
<p align="center"><em>KDS por setor (ex.: Cozinha, Bar) com colunas de novos, em preparo e prontos.</em></p>

<br />

<p align="center"><strong>Caixa</strong></p>
<p align="center">
  <img src="docs/screenshots/caixa.png" alt="Módulo de caixa" width="820" />
</p>
<p align="center"><em>Abertura de sessão com fundo de troco e histórico do último fechamento.</em></p>

<br />

<p align="center"><strong>Estoque</strong></p>
<p align="center">
  <img src="docs/screenshots/estoque.png" alt="Controle de estoque" width="820" />
</p>
<p align="center"><em>KPIs, bipagem, transações recentes e lista de insumos com saldo mínimo.</em></p>

<br />

<p align="center"><strong>Admin da plataforma</strong></p>
<p align="center">
  <img src="docs/screenshots/admin.png" alt="Admin da plataforma" width="820" />
</p>
<p align="center"><em>Provisionamento de empresas, métricas agregadas e suporte temporário.</em></p>

---

## Arquitetura (resumo)

```text
Cliente (React / TanStack)
        │
        ▼
Supabase Auth + API (anon key)
        │
        ▼
PostgreSQL  ── RLS por tenant
            ── RPCs de negócio
            ── Realtime filtrado
            ── Storage (imagens)
```

- **Frontend:** UX, formulários, cache (TanStack Query) e fila offline.
- **Banco:** isolamento multi-tenant, state machines, totais financeiros e auditoria.
- **Plataforma:** admins fora do papel da empresa, com RPCs dedicadas.

Detalhes em **[docs/arquitetura.md](docs/arquitetura.md)**.

Documentação interna complementar: [status do produto](docs/status-do-produto.md), [PRD](docs/prd.md), [roadmap](docs/roadmap.md).

---

## Status do projeto

| Aspecto | Situação |
| ------- | -------- |
| Operação do restaurante | MVP utilizável (vendas, mesas, cozinha, caixa, estoque, equipe) |
| Admin SaaS | MVP (criar empresa/login, pausar, suporte) |
| Impressão | Comanda/recibo via diálogo do navegador (não fiscal) |
| Fora do escopo atual | TEF/maquininha, NFC-e, iFood, cardápio QR, delivery avançado |

Produto em evolução contínua, com foco em segurança multi-tenant e operação real de restaurante.
