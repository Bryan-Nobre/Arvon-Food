# Arquitetura — Arvon Food

Visão técnica do case de portfólio. Complementa o [README](../README.md).  
Descreve **como o produto foi pensado**, sem expor caminhos internos, contratos de API ou detalhes operacionais do código-fonte privado.

---

## Visão geral

O Arvon Food é um **SaaS multi-tenant**: cada restaurante é um tenant isolado; cada pessoa autorizada é membro com um papel.

```text
Usuário autenticado
    │
    ├── vínculo com restaurante(s) ──► operação (pedidos, mesas, caixa, estoque…)
    │
    └── admin da plataforma (opcional) ──► provisionamento e suporte
```

Princípio central: **o navegador não é confiável**. A interface organiza a experiência; **autorização, totais e estados válidos ficam no servidor/banco**.

---

## Organização dos módulos

### Módulos do produto

| Módulo | Função |
| ------ | ------ |
| Autenticação e convites | Entrada na conta e adesão à equipe |
| Dashboard | Resumo operacional do dia |
| Pedidos / PDV | Venda no balcão ou ligada a mesa/comanda |
| Mesas | Salão, comandas e transferência |
| Cozinha | Produção por setor (KDS) |
| Caixa | Sessão, movimentações e recebimentos |
| Estoque | Insumos, bipagem e histórico |
| Cadastros e gestão | Produtos, clientes, relatórios, configurações |
| Admin da plataforma | Criação de restaurantes e suporte |

### Camadas do frontend

```text
Telas e componentes
        │
        ▼
Hooks de dados (cache e mutações)
        │
        ▼
Serviços de integração
        │
        ▼
Auth · API · tempo real · arquivos
```

Separação típica: UI → hooks → services → backend gerenciado.  
O menu pode variar por papel e por módulos ativos do restaurante; isso controla **navegação**, não a segurança final.

---

## Frontend e backend

Não há API Node própria no MVP. O **motor de regras** fica no **PostgreSQL**, acessado via plataforma gerenciada (Supabase).

| Responsabilidade | Onde fica |
| ---------------- | --------- |
| Sessão e identidade | Autenticação gerenciada |
| Isolamento entre restaurantes | Políticas no banco (RLS) |
| Pedidos, caixa, estoque, mesas | Funções e validações no servidor |
| Tempo real | Canais limitados ao tenant |
| Imagens | Storage com controle de acesso |
| Offline | Fila local no dispositivo |

Segredos administrativos e chaves privilegiadas **não** vão para o frontend. O cliente usa apenas credenciais públicas previstas pela plataforma.

---

## Dados e relacionamentos

### Núcleo multi-tenant

| Conceito | Papel |
| -------- | ----- |
| Restaurante (empresa) | Tenant |
| Membro | Usuário + restaurante + papel |
| Convite | Entrada controlada na equipe |
| Sequências | Numeração operacional por restaurante |
| Auditoria | Registro de ações sensíveis |

### Domínios

| Domínio | Conteúdo |
| ------- | -------- |
| Catálogo | Categorias, produtos, adicionais, setores de cozinha |
| Salão | Mesas e comandas |
| Pedidos | Itens, status operacional e financeiro |
| Caixa | Sessões, pagamentos e movimentações |
| Estoque | Insumos, movimentações e baixas |
| Relatórios | Agregações no servidor |
| Plataforma | Admins SaaS e acessos de suporte |

### Relacionamento operacional

```text
Mesa
  └── Comanda (uma aberta por mesa)
        └── Pedidos
              └── Itens
                    └── Pagamentos → movimentações de caixa
                    └── movimentações de estoque (quando aplicável)
```

Status **operacional** (produção/entrega) e status **financeiro** (pagamento) são eixos independentes.

---

## Autenticação e permissões

- Login com e-mail e senha via autenticação gerenciada.
- Restaurantes não se auto-cadastram publicamente; a equipe entra por convite, e o provisionamento pode ser feito pela plataforma.
- Papéis de restaurante: proprietário, administrador, caixa, garçom e cozinha.
- Admin da plataforma é um papel **separado**: cria restaurantes, acompanha métricas leves e pode abrir suporte temporário.

A interface esconde o que o perfil não deve ver. O banco bloqueia o que o perfil não pode fazer.

---

## Segurança e validações

| Camada | Função |
| ------ | ------ |
| Formulários no cliente | UX e validação inicial |
| Políticas no banco | Impedem acesso cross-tenant |
| Funções de negócio | Recalculam totais, validam caixa, estoque e transições |
| Constraints | Integridade relacional |
| Idempotência | Evita duplicar ação crítica em retry |
| Auditoria | Rastreia cancelamentos, pagamentos e mudanças sensíveis |

Offline: a fila local **não** fecha a operação. Só vale depois que o servidor aceita e revalida.

---

## Integrações

| Integração | Situação |
| ---------- | -------- |
| Plataforma de Auth / DB / Realtime / Storage | Em uso |
| Impressão de comanda e recibo | Via navegador (não fiscal) |
| Convite por e-mail automático | Não; o fluxo atual é por link |
| Maquininha, nota fiscal, marketplaces | Fora do escopo atual |
| Leitor de código de barras | Em uso no estoque |

---

## Fluxos principais

### Pedido

```text
Catálogo → montagem do pedido
    → gravação no servidor (com snapshot do que foi vendido)
    → envio à cozinha
    → atualização em tempo real no KDS
    → pagamento (com caixa aberto)
    → caixa e, se aplicável, estoque
```

### Mesa

```text
Abrir mesa → comanda
    → pedidos
    → pedir conta
    → pagamento
    → liberar mesa
```

A transferência move a comanda inteira para outra mesa disponível.

### Offline parcial

Permitido em ações leves de salão (abrir mesa, lançar pedido, pedir conta).  
Pagamento, caixa, estoque, transferência e cadastros pedem conexão.

### Provisionamento SaaS

```text
Admin da plataforma
    → cria restaurante e acesso do responsável
    → credenciais entregues fora do app
    → equipe opera e convida membros
```

---

## Decisões arquiteturais

1. **Zero trust no cliente** — menu e botões são UX; o banco autoriza.
2. **Postgres como motor de negócio** — regras críticas em funções atômicas.
3. **Dois eixos de status no pedido** — operacional ≠ financeiro.
4. **Snapshot na venda** — histórico estável após alteração de cardápio.
5. **Desativação em vez de apagar** cadastros já usados em vendas.
6. **Offline limitado de propósito** — protege caixa e estoque.
7. **Módulos configuráveis por restaurante** — sem forçar um ERP completo.
8. **Admin SaaS separado** dos papéis do restaurante.
9. **Impressão pelo navegador** no MVP, sem depender de agente local.
10. **Schema evolui por migrations versionadas**.

---

## Escalabilidade

| Vetor | Direção |
| ----- | ------- |
| Mais restaurantes | Isolamento por tenant |
| Mais dispositivos | Tempo real por restaurante |
| Mais volume | Listagens paginadas e consultas específicas |
| Novos canais | Integrações no servidor, mantendo o banco como fonte da verdade |
| Impressão dedicada | Agente local sem mudar o domínio de pedidos |
| Observabilidade | Auditoria e logs sem segredos |

Limites conscientes do MVP: sem nota fiscal, sem TEF e offline parcial — escolhas de produto, não improviso de arquitetura.
