# Arquitetura — Arvon Food

Documentação técnica do produto com base no código e nas migrations atuais. Complementa o [README](../README.md).

---

## Visão geral

O Arvon Food é um **SaaS multi-tenant** em que cada restaurante é uma linha em `empresas` e cada usuário autorizado é um `membros_empresa` com um `papel_usuario`.

```text
auth.users
    │
    ├── membros_empresa ──► empresa ──► pedidos, mesas, caixa, estoque…
    │
    └── admins_plataforma (opcional) ──► RPCs de gestão SaaS / suporte
```

Princípio de segurança: **o navegador não é confiável**. A interface controla UX e esconde ações; **RLS, constraints e RPCs Postgres** controlam autorização, totais e estados válidos.

---

## Organização dos módulos

### Rotas da aplicação (`src/routes`)

| Rota | Função |
| ---- | ------ |
| `/login`, `/cadastro`, `/aceitar-convite` | Autenticação e entrada por convite |
| `/onboarding` | Fluxo residual quando o usuário autenticado ainda não tem empresa (admin de plataforma sem vínculo vai ao painel SaaS) |
| `/` | Dashboard operacional |
| `/pedidos`, `/pdv` | PDV e gestão de pedidos |
| `/mesas` | Salão e comandas |
| `/cozinha` | KDS por setor |
| `/caixa` | Sessão de caixa e movimentações |
| `/estoque`, `/estoque/historico` | Insumos e histórico |
| `/produtos`, `/clientes`, `/relatorios` | Gestão (acesso típico via Configurações) |
| `/configuracoes`, `/ajuda` | Empresa, equipe, preferências e guias |
| `/admin` | Admin da plataforma |

### Camadas do frontend

```text
routes / components
        │
        ▼
hooks (TanStack Query)
        │
        ▼
services (cliente Supabase)
        │
        ▼
Supabase Auth / PostgREST / RPC / Realtime / Storage
```

Pastas relevantes:

| Pasta | Papel |
| ----- | ----- |
| `src/components` | UI por domínio (`pdv`, `mesas`, `estoque`, `layout`, `ui`…) |
| `src/hooks` | Consultas e mutações tipadas |
| `src/services` | Chamadas a tabelas e RPCs |
| `src/providers` | Auth, empresa ativa, tema, offline, fila |
| `src/lib` | Permissões de menu, erros, impressão, offline, formatação |
| `src/types/db.ts` | Tipos gerados a partir do schema |

### Menu e módulos opcionais

Em `src/lib/permissoes.ts`:

- **Base do menu:** dashboard, PDV, caixa, estoque, configurações
- **Opcionais** (`empresas.modulos_extras`): mesas, cozinha, produtos, clientes, relatórios

A tabela de papéis (`MODULOS_POR_PAPEL`) só filtra navegação. Comentários no código deixam explícito que a autorização real está no banco.

---

## Estrutura do frontend e do “backend”

Não há API Node própria no MVP. O **backend de negócio** é o **PostgreSQL + funções** expostas pelo Supabase.

| Responsabilidade | Onde vive |
| ---------------- | --------- |
| Sessão e identidade | Supabase Auth |
| Isolamento e policies | RLS em tabelas públicas |
| Regras de pedido, caixa, estoque, mesas | RPCs `SECURITY DEFINER` (schema `app` / `public`) |
| Tempo real | Supabase Realtime, canais filtrados por empresa |
| Imagens | Storage (ex.: produtos em WebP, logo da empresa) |
| Offline | IndexedDB + Service Worker no cliente |

O frontend usa a **anon/public key** (`VITE_*`). Chaves de service role e segredos administrativos **não** entram no bundle.

---

## Banco de dados e relacionamentos

### Núcleo multi-tenant (fundação)

Entidades centrais (migration `01_fundacao_base` e seguintes):

| Entidade | Relação |
| -------- | ------- |
| `empresas` | Tenant |
| `membros_empresa` | `usuario_id` + `empresa_id` + `papel` |
| `convites` | Convite pendente por e-mail/token |
| `sequencias_empresa` | Numeração de pedido/comanda por empresa |
| `auditoria` | Trilha de ações sensíveis |

Schema interno `app`: helpers como `eh_membro`, exigências de papel e funções usadas pelas policies — **não** exposto diretamente pela API REST.

### Domínios operacionais (migrations)

| Domínio | Exemplos no schema |
| ------- | ------------------ |
| Catálogo | categorias, produtos, adicionais, setores de cozinha, storage |
| Salão | mesas, comandas, transferências |
| Pedidos | pedidos, itens, estados operacional/financeiro, RPCs de criação e transição |
| Caixa | sessões, movimentações, pagamentos, sangria/suprimento, fechamento |
| Estoque | itens, movimentações, ficha técnica por item, baixa em venda |
| Relatórios | funções de agregação no servidor |
| Plataforma | `admins_plataforma`, `acessos_suporte`, RPCs de métricas e provisionamento |

### Relacionamento operacional típico

```text
Mesa
  └── Comanda (no máximo uma aberta por mesa)
        └── Pedidos
              └── Itens (+ adicionais / snapshots)
                    └── Pagamentos ──► Movimentações de caixa
                    └── Movimentações de estoque (quando aplicável)
```

Estados **operacionais** do pedido (ex.: confirmado → preparando → pronto → entregue / cancelado) são independentes dos estados **financeiros** (não pago, parcial, pago, estornado).

---

## Autenticação e permissões

### Autenticação

- Login e cadastro via **Supabase Auth** (`src/services/auth.ts`).
- Cadastro público de **restaurante** não é o fluxo padrão: `/cadastro` redireciona para login **sem** token de convite; com convite, cria conta de membro.
- Empresas e logins de dono podem ser criados pelo **admin da plataforma** (RPC dedicada).

### Papéis de empresa

| Papel | Uso típico na UI |
| ----- | ---------------- |
| `owner` | Acesso amplo + gestão |
| `admin` | Operação e administração |
| `cashier` | PDV, caixa, mesas, clientes |
| `waiter` | PDV, mesas, cozinha |
| `kitchen` | Cozinha |

### Admin da plataforma

Tabelas `admins_plataforma` e `acessos_suporte` + RPCs (`admin_plataforma_*`, `sou_admin_plataforma`, etc.):

- listar empresas e métricas leves
- criar empresa + usuário dono
- pausar / reativar
- iniciar / encerrar suporte (membro temporário)

Esse papel é **ortogonal** ao `papel_usuario` da empresa.

---

## Segurança e validações

| Camada | O que faz |
| ------ | --------- |
| Zod / React Hook Form | Validação de formulário e UX |
| RLS | Impede leitura/escrita cross-tenant |
| RPCs | Recalculam totais, validam caixa aberto, estoque, transição de status, papéis |
| Constraints / CHECKs / FKs | Integridade no Postgres |
| Idempotência | `client_request_id` evita duplicar pedido/ação após retry |
| Auditoria | Registro de cancelamentos, pagamentos, convites e afins |
| Frontend | Sem `dangerouslySetInnerHTML` como padrão; queries seletivas nos services |

Offline: a fila local **não** conclui a operação. Só após o servidor aceitar (revalidando permissão, preço, comanda, etc.) o estado passa a sincronizado.

---

## Integrações externas

| Integração | Situação no código atual |
| ---------- | ------------------------ |
| Supabase (Auth, DB, Realtime, Storage) | Em uso |
| Impressão de comanda/recibo | HTML 80mm + `window.print()` no navegador — **não** é cupom fiscal |
| E-mail transacional de convite | Não envia; fluxo é link / WhatsApp manual |
| TEF / maquininha, NFC-e, iFood | Fora do escopo atual (planejados em docs de produto) |
| ZXing | Leitura de código de barras no estoque |

---

## Fluxos principais

### Pedido no balcão ou mesa

```text
Catálogo → carrinho (UI)
    → RPC criar pedido (snapshot de preços)
    → (opcional) envio automático à cozinha
    → KDS atualiza via Realtime
    → pagamento no caixa/PDV (caixa aberto)
    → movimentação de caixa + possível baixa de estoque
```

### Mesa / comanda

```text
Abrir mesa → comanda aberta
    → pedidos na comanda
    → pedir conta
    → pagamento
    → fechar / liberar mesa
```

Transferência move a **comanda inteira** para outra mesa disponível (regras no banco).

### Offline parcial

Ações suportadas na fila (`src/lib/offline/fila.ts`):

- abrir comanda
- criar pedido
- pedir conta

Pagamento, caixa, estoque, transferência e cadastros **exigem** conexão.

### Provisionamento SaaS

```text
Admin plataforma autenticado
    → RPC criar empresa + login do dono
    → credenciais entregues fora do app
    → dono entra e opera / convida equipe
```

---

## Decisões arquiteturais

1. **Zero trust no cliente** — permissões de menu e botões são UX; RLS/RPC são a barreira real.
2. **Postgres como motor de negócio** — state machines, totais e efeitos colaterais (caixa/estoque) em funções atômicas.
3. **Dois eixos de status no pedido** — operacional ≠ financeiro.
4. **Snapshots em venda** — histórico não muda quando o cardápio é editado depois.
5. **Soft-delete / desativação** em cadastros usados historicamente (ex.: produto inativo).
6. **Offline deliberadamente limitado** — evita inconsistência financeira e de estoque.
7. **Menu configurável por empresa** — `modulos_extras` liga mesas/cozinha/etc. sem forçar ERP completo.
8. **Admin SaaS separado** — provisionamento e suporte sem misturar com papéis do restaurante.
9. **Impressão pelo navegador** — entrega valor imediato sem agente ESC/POS no MVP.
10. **Migrations versionadas** — evolução do schema só via `supabase/migrations`.

---

## Possibilidades de escalabilidade

| Vetor | Como o desenho atual ajuda |
| ----- | -------------------------- |
| Mais restaurantes | Isolamento por `empresa_id` + índices por tenant |
| Mais usuários simultâneos | Realtime por empresa; sem broadcast global |
| Crescimento de volume | Paginação nas listagens; queries tipadas nos services |
| Novos canais (iFood, TEF, fiscal) | Encaixe natural como integrações **server-side** / Edge Functions, mantendo o Postgres como fonte da verdade |
| Impressão dedicada | Evolução para agente local ESC/POS sem mudar o domínio de pedidos |
| Observabilidade | Auditoria já modelada; logs de app sem secrets |

Limites conscientes do MVP: sem NF-e, sem TEF, offline parcial e impressão via diálogo do SO — escolhas de produto, não lacunas acidentais de arquitetura.

---

## Referências no repositório

| Artefato | Conteúdo |
| -------- | -------- |
| `supabase/migrations/` | Schema, RLS, RPCs |
| `src/types/db.ts` | Contrato tipado do banco |
| `src/lib/permissoes.ts` | Papéis e módulos de menu |
| `src/lib/offline/` | Fila, IndexedDB, conectividade |
| [status-do-produto.md](./status-do-produto.md) | O que está pronto vs. planejado |
| [prd.md](./prd.md) / [roadmap.md](./roadmap.md) | Produto e evolução |
