# Arvon Food

SaaS multi-tenant de gestão operacional para restaurantes — PDV, mesas e comandas, cozinha (KDS), caixa, estoque e administração da plataforma.

<p align="center">
  <img src="https://i.ibb.co/WpkSQTrw/dashboard.png" alt="Dashboard do Arvon Food" width="820" />
</p>

> Repositório de **portfólio**. O código-fonte do produto permanece privado; aqui está a apresentação do case.

---

## Visão geral

O Arvon Food organiza o dia a dia do restaurante em um único produto web: do pedido no balcão ou na mesa até a produção na cozinha, o recebimento no caixa e o controle de insumos.

Cada restaurante opera de forma isolada. Usuários entram com papéis definidos. Regras financeiras, estoque, permissões e isolamento entre clientes são garantidos no **banco de dados**, não apenas na interface.

---

## Principais funcionalidades

- **PDV / Pedidos** — montagem de pedido no balcão ou vinculado a mesa/comanda, com catálogo e estados operacionais e financeiros separados
- **Mesas e comandas** — mapa do salão, abertura de comanda, pedido de conta e transferência entre mesas
- **Cozinha (KDS)** — filas por setor, avanço de status e impressão de comanda pelo navegador
- **Caixa** — abertura/fechamento de sessão, sangria, suprimento e recebimentos (incluindo pagamento parcial)
- **Estoque** — movimentações com histórico, alerta de baixo estoque e bipagem por leitor ou câmera
- **Cadastros** — produtos, categorias, adicionais, clientes, equipe e configurações do restaurante
- **Convites de equipe** — entrada de membros por convite (sem auto-cadastro público de restaurante)
- **Admin da plataforma** — provisionamento de restaurantes, métricas leves e acesso temporário de suporte
- **Offline parcial** — algumas ações de salão podem ser enfileiradas sem internet; operações críticas exigem conexão
- **Tempo real** — salão, cozinha e caixa atualizam entre dispositivos do mesmo restaurante

---

## Destaques técnicos

| Tema | Abordagem |
| ---- | --------- |
| Multi-tenancy | Isolamento por restaurante, com políticas no banco |
| Fonte da verdade | Operações críticas validadas e persistidas no servidor |
| Papéis | Proprietário, administrador, caixa, garçom e cozinha — a UI filtra; o banco autoriza |
| Consistência | Evita duplicar operações críticas em retries e sincronização |
| Histórico de venda | Pedidos preservam o que foi vendido, mesmo se o cardápio mudar depois |
| Experiência | Interface clara, com modo escuro |

---

## Stack tecnológica

| Camada | Tecnologias |
| ------ | ----------- |
| Frontend | React, TypeScript, Vite, TanStack (Start / Router / Query), Tailwind CSS, Radix UI, React Hook Form, Zod |
| Backend / dados | Supabase (Auth, PostgreSQL, Realtime, Storage), SQL versionado, funções e políticas no banco |
| Offline / leitura | Persistência local, Service Worker, leitura de código de barras |
| UX | Toasts, gráficos, impressão de comanda/recibo pelo navegador |

---

## Principais interfaces

<p align="center"><strong>Login</strong></p>
<p align="center">
  <img src="https://i.ibb.co/b5cDgxvV/login.png" alt="Tela de login" width="720" />
</p>
<p align="center"><em>Acesso com e-mail e senha. Cadastro aberto só para aceitar convite de equipe.</em></p>

<br />

<p align="center"><strong>Dashboard</strong></p>
<p align="center">
  <img src="https://i.ibb.co/WpkSQTrw/dashboard.png" alt="Dashboard operacional" width="820" />
</p>
<p align="center"><em>Resumo do dia: vendas, ticket médio, cozinha e ocupação de mesas.</em></p>

<br />

<p align="center"><strong>Pedidos (PDV)</strong></p>
<p align="center">
  <img src="https://i.ibb.co/9dvgMZ9/pedidos.png" alt="PDV e pedidos" width="820" />
</p>
<p align="center"><em>Catálogo, filtros de status e montagem de pedido (balcão, mesa ou comanda).</em></p>

<br />

<p align="center"><strong>Mesas</strong></p>
<p align="center">
  <img src="https://i.ibb.co/bgvbZSjX/mesas.png" alt="Mapa de mesas" width="820" />
</p>
<p align="center"><em>Salão em lista e mapa, com status livre, ocupada e aguardando pagamento.</em></p>

<br />

<p align="center"><strong>Cozinha</strong></p>
<p align="center">
  <img src="https://i.ibb.co/wZDy6gsr/cozinha.png" alt="KDS da cozinha" width="820" />
</p>
<p align="center"><em>KDS por setor com colunas de novos, em preparo e prontos.</em></p>

<br />

<p align="center"><strong>Caixa</strong></p>
<p align="center">
  <img src="https://i.ibb.co/v9DBs40/caixa.png" alt="Módulo de caixa" width="820" />
</p>
<p align="center"><em>Abertura de sessão com fundo de troco e histórico do último fechamento.</em></p>

<br />

<p align="center"><strong>Estoque</strong></p>
<p align="center">
  <img src="https://i.ibb.co/23wRS8Xw/estoque.png" alt="Controle de estoque" width="820" />
</p>
<p align="center"><em>Indicadores, bipagem, movimentações recentes e insumos com saldo mínimo.</em></p>

<br />

<p align="center"><strong>Admin da plataforma</strong></p>
<p align="center">
  <img src="https://i.ibb.co/JjLYqqrg/admin.png" alt="Admin da plataforma" width="820" />
</p>
<p align="center"><em>Provisionamento de restaurantes, métricas agregadas e suporte temporário.</em></p>

---

## Arquitetura (resumo)

```text
Cliente web (React)
        │
        ▼
Autenticação + API gerenciada
        │
        ▼
PostgreSQL  ── isolamento por restaurante
            ── regras de negócio no servidor
            ── tempo real por tenant
            ── armazenamento de imagens
```

- **Frontend:** experiência, formulários, cache e fila offline limitada.
- **Banco:** autorização, totais, estados válidos e auditoria.
- **Plataforma:** gestão SaaS separada dos papéis do restaurante.

Detalhes em **[docs/arquitetura.md](docs/arquitetura.md)**.

---

## Status do projeto

| Aspecto | Situação |
| ------- | -------- |
| Operação do restaurante | MVP utilizável (vendas, mesas, cozinha, caixa, estoque, equipe) |
| Admin SaaS | MVP (criar restaurante/login, pausar, suporte) |
| Impressão | Comanda/recibo via navegador (não fiscal) |
| Próximas evoluções | Integrações de pagamento, fiscal e canais de pedido |

Produto em evolução, com foco em segurança multi-tenant e operação real de restaurante.
