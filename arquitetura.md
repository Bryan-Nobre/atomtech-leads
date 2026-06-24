# Arquitetura Técnica — AtomTech Indicações

Documentação técnica da plataforma SaaS de indicações comerciais, comissões e ferramentas de dimensionamento para energia solar e mobilidade elétrica.

---

## 1. Visão geral da arquitetura

A aplicação é um SaaS de página única (SPA) com backend gerenciado, organizada em torno de um princípio central: **o frontend não é confiável**. A interface cuida apenas da experiência do usuário; toda regra de negócio sensível — permissões, status de negócios, cálculo e liberação de comissões — é decidida e validada no servidor.

```text
┌─────────────────────────────────────────────┐
│  Frontend (SPA)                               │
│  React 19 · TypeScript · Vite                 │
│  TanStack Router · TanStack Query             │
│  Tailwind CSS · shadcn/ui                     │
└───────────────┬───────────────────────────────┘
                │  HTTPS (JWT do usuário)
                ▼
┌─────────────────────────────────────────────┐
│  Supabase                                     │
│  ├─ Auth (GoTrue / OTP)                        │
│  ├─ PostgREST  → leitura/escrita com RLS       │
│  ├─ Edge Functions (Deno) → lógica sensível    │
│  └─ Storage → documentos e imagens             │
└───────────────┬───────────────────────────────┘
                ▼
┌─────────────────────────────────────────────┐
│  PostgreSQL                                   │
│  Tabelas · Views · RPC · Triggers · RLS       │
└─────────────────────────────────────────────┘

        Firebase Cloud Messaging  (push)
```

**Camadas:**

- **Apresentação:** componentes React, roteamento por arquivos (TanStack Router) e cache/sincronização de dados (TanStack Query).
- **Acesso a dados:** o cliente Supabase usa exclusivamente a *anon key* + JWT do usuário; o acesso é filtrado por **Row Level Security** no banco.
- **Lógica crítica:** **Edge Functions** em Deno executam operações privilegiadas (criação de usuários, OTP, papéis, comissões, push) com a *service role key*, nunca exposta ao cliente.
- **Persistência:** PostgreSQL com tabelas, *views*, funções (RPC) e *triggers* concentrando as regras de negócio.

---

## 2. Fluxo completo de indicação

1. **Captação** — o indicador chega pela landing page e se cadastra.
2. **Autenticação** — confirmação de identidade por código OTP enviado por e-mail.
3. **Nova indicação** — o indicador informa os dados do potencial cliente (pessoa ou empresa), tipo de projeto e observações.
4. **Documentos de apoio** — upload opcional de conta de energia, fotos do local e arquivos relacionados, armazenados no Supabase Storage.
5. **Persistência** — a indicação é gravada na tabela `indicacoes`, vinculada ao indicador, com status inicial.
6. **Acompanhamento** — a indicação passa a aparecer no dashboard do indicador e no painel administrativo.

A escrita é validada por RLS: o indicador só consegue criar indicações associadas ao próprio usuário e só enxerga as suas.

---

## 3. Fluxo de aprovação e comissões

O ciclo comercial é representado pelo **status** da indicação (ex.: enviado → análise → negociação → fechado/perdido) e por registros correspondentes na tabela `comissoes`.

Regras centrais — todas no backend:

- **O indicador nunca altera status nem valores.** Esses campos são controlados exclusivamente pela equipe administrativa.
- **Cálculo de comissão** ocorre no banco, a partir do valor do projeto definido pelo admin.
- **Negócio perdido cancela a receita:** um *trigger* em `indicacoes` detecta a mudança para o status `perdido` e marca a comissão vinculada como `cancelado`, removendo-a do faturamento e dos acumulados. Caso o status saia de `perdido`, a comissão é reativada.
- **Relatórios e somatórios** (RPC de resumo) ignoram comissões canceladas, garantindo números consistentes em todas as telas.

Esse desenho assegura que valores financeiros sejam sempre derivados de uma única fonte de verdade no banco.

---

## 4. Hierarquia de permissões

Três papéis principais, com escopos distintos:

| Papel             | Escopo                                                                 |
| ----------------- | ---------------------------------------------------------------------- |
| **indicador**     | Cria e acompanha apenas as próprias indicações e comissões.            |
| **visualizador**  | Acessa o painel admin de forma restrita, conforme permissões por módulo. |
| **admin**         | Acesso total: gestão de indicações, usuários, comissões e configurações. |

Para o papel **visualizador**, o acesso é granular por **módulo** (overview, indicadores, projetos, mensagens, analytics, configurações) e por **ação** (ler, criar, editar, excluir, comentar). O frontend usa esse mapa apenas para mostrar/ocultar a interface; a autorização efetiva é reaplicada no backend.

A função `is_admin()` (PostgreSQL, `SECURITY DEFINER`) é a base das políticas de escrita sensíveis.

---

## 5. Dashboard do indicador

Área voltada à ativação e retenção do indicador:

- lista de indicações com status atual e indicação visual de negócios perdidos;
- resumo de comissões (acumulado, pendente, pago);
- funil e métricas por período;
- edição limitada dos próprios dados de contato — bloqueada para negócios já perdidos, tanto na interface quanto via RLS.

Os dados são carregados por *views*/RPC e mantidos em cache pelo TanStack Query.

---

## 6. Painel administrativo

Central operacional da equipe comercial, organizada em abas:

- **Overview / Analytics / Relatórios** — indicadores de desempenho, funil e receita.
- **Indicações e Projetos** — gestão de leads, status, valores, comentários e vínculos.
- **Usuários e Permissões** — papéis e matriz de permissões por módulo.
- **Central de Mensagens** — modelos de mensagem para abordagem comercial.
- **Calculadoras Solares** — ferramentas de dimensionamento (ver seção 16), com uma área de **cadastros** (veículos elétricos e locais de irradiação) acessível via engrenagem.

A renderização das abas e ações é condicionada às permissões do usuário autenticado.

---

## 7. Supabase Auth

Autenticação baseada em **GoTrue** com verificação por **código OTP** enviado por e-mail (SMTP próprio). O cadastro de novos usuários é orquestrado por Edge Function, separando criação de conta do envio/validação do código. A sessão é mantida com refresh automático de token no cliente.

---

## 8. PostgreSQL

O banco concentra a inteligência da aplicação:

- **Tabelas** para usuários, indicações, comissões, atividades, mensagens, notificações, tokens de push e cadastros das calculadoras (veículos e locais de irradiação).
- **Views e RPC** para resumos, funil, relatórios e agregações financeiras, evitando lógica de cálculo no cliente.
- **Triggers** para regras automáticas (ex.: cancelamento de comissão em negócio perdido, sincronização de dados).
- **Migrations versionadas** descrevendo toda a evolução do schema.

Identificadores `int8` são usados para indexação eficiente, e `JSONB` oferece flexibilidade onde necessário.

---

## 9. Row Level Security (RLS)

RLS é o mecanismo central de autorização. Todas as tabelas sensíveis têm RLS habilitado, com políticas como:

- indicadores enxergam e modificam **apenas** seus próprios registros;
- administradores têm acesso amplo via `is_admin()`;
- dados de referência (ex.: veículos, locais de irradiação) são **legíveis** por usuários autenticados, mas **graváveis apenas** por administradores;
- campos críticos (papel do usuário, status, comissão) são protegidos contra alteração indevida diretamente nas políticas.

Como a autorização vive no banco, ela permanece válida independentemente do que o frontend tente fazer.

---

## 10. Edge Functions

Funções serverless em Deno hospedam a lógica que **não pode** rodar no cliente, usando a *service role key* com segurança no servidor:

- cadastro e provisionamento de usuários;
- envio e verificação de OTP;
- alteração de papéis e permissões;
- operações administrativas sobre indicações, projetos, comissões e mensagens;
- disparo de notificações push.

Cada função valida a identidade e as permissões do solicitante antes de executar qualquer ação.

---

## 11. Firebase Cloud Messaging

As notificações push usam **Firebase Cloud Messaging**:

- tokens de dispositivo são registrados e armazenados por usuário;
- eventos relevantes (ex.: mudança de status de uma indicação) disparam notificações via Edge Function;
- um *service worker* dedicado processa as mensagens no navegador.

---

## 12. Upload de documentos

Arquivos de apoio (conta de energia, fotos do local, documentos de projeto, foto de perfil) são enviados ao **Supabase Storage**. O acesso aos *buckets* é regido por políticas que respeitam o vínculo do arquivo com o usuário/indicação, mantendo o mesmo modelo de confiança do restante da plataforma.

---

## 13. Segurança Zero Trust

Princípios aplicados de forma consistente:

- **Backend é a verdade absoluta** — nenhuma decisão sensível depende do cliente.
- **Mínimo privilégio** — RLS por usuário e permissões por módulo/ação.
- **Segredos protegidos** — apenas a *anon key* no frontend; *service role* exclusiva das Edge Functions.
- **Dados mínimos trafegados** — *queries* selecionam apenas o necessário.
- **Proteção financeira** — comissões e status imutáveis pelo indicador.
- **Auditabilidade** — registro de atividades para rastrear criação de indicações e mudanças relevantes.

---

## 14. Decisões arquiteturais

- **SPA + BaaS (Supabase):** acelera a entrega mantendo um backend robusto (Postgres + RLS + Functions) sem servidor próprio para gerenciar.
- **Regra de negócio no banco:** centralizar cálculos e validações em RPC/triggers garante consistência entre todas as telas e impede divergências de lógica no cliente.
- **TanStack Query:** cache, invalidação e sincronização declarativos reduzem código de estado e chamadas redundantes.
- **shadcn/ui + Tailwind:** base de componentes acessível e consistente, com total controle do código.
- **Cadastros reutilizáveis:** dados das calculadoras (veículos, irradiação) vivem em tabelas próprias, consumidas por uma camada única — sem duplicação entre telas.
- **Funções de cálculo puras:** a lógica das calculadoras é isolada em funções puras, facilitando testes e reuso.

---

## 15. Escalabilidade e integração entre módulos

- **Escalabilidade:** identificadores `int8`, índices dedicados, agregações via RPC e estrutura de dados preparada para múltiplos tipos de lead e produtos.
- **Integração entre módulos:** indicações, comissões, usuários, notificações e relatórios compartilham o mesmo modelo de dados e as mesmas políticas de segurança. Uma mudança de status, por exemplo, reverbera automaticamente em comissões (trigger), relatórios (RPC) e notificações (Edge Function + FCM), sem acoplamento no frontend.

---

## 16. Fluxo das calculadoras solares

Ferramentas de apoio comercial, divididas em dois modos:

**Carregadores** — estima quantas placas são necessárias para carregar um veículo elétrico, a partir do consumo do veículo, quilometragem, potência do painel, irradiação local e perdas do sistema.

**Placas solares** — três cálculos selecionáveis:

1. **Geração mensal** de um sistema existente (potência × HSP × dias × PR);
2. **Quantidade de painéis** necessária para um consumo informado;
3. **Potência necessária** para atender determinado consumo.

Características do fluxo:

- entrada simplificada com seleção de **veículo** e **local de irradiação** a partir dos cadastros (preenchimento automático dos valores técnicos);
- apresentação em etapas: **dados → resultados → detalhamento**;
- **memória de cálculo** exibindo cada etapa intermediária e a fórmula aplicada, reforçando transparência e confiança na estimativa;
- lógica implementada em funções puras reutilizáveis, desacopladas da interface.
