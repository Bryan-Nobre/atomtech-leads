# AtomTech Indicações

Plataforma SaaS para gestão de indicações, geração de oportunidades comerciais e controle de comissões voltada para os mercados de energia solar e carregadores para veículos elétricos.

O sistema conecta indicadores, potenciais clientes e equipe comercial em um fluxo centralizado de conversão, permitindo acompanhamento completo desde a indicação até o fechamento do negócio.

> Código-fonte privado.
>
> Este repositório apresenta a arquitetura, funcionalidades e decisões técnicas do projeto.

---

## Principais Recursos

- Sistema de indicações online
- Controle de comissões
- Upload de documentos
- Dashboard do indicador
- Painel administrativo
- Controle de status e oportunidades
- Analytics e relatórios
- Notificações push
- Calculadoras solares
- Controle de permissões por papel
- Segurança baseada em RLS e Edge Functions

---

## Destaques Técnicos

- Arquitetura SaaS
- Controle de permissões por papel
- Row Level Security (RLS)
- Supabase Edge Functions
- Firebase Cloud Messaging
- Dashboard analítico
- Sistema de comissões
- Upload de documentos
- Calculadoras técnicas para energia solar
- Arquitetura Zero Trust

---

## Stack Tecnológica

### Frontend

- React 19
- TypeScript
- Vite
- TanStack Router
- TanStack Query
- Tailwind CSS
- shadcn/ui

### Backend

- Supabase
- PostgreSQL
- Edge Functions
- Storage
- Firebase Cloud Messaging

---

## Principais Interfaces

### Landing Page

Página pública de apresentação da plataforma, utilizada para captação de novos indicadores e divulgação das soluções de energia solar e mobilidade elétrica.

<p align="center">
  <img src="https://i.ibb.co/Y4MwsCN6/atomtech-landpage-indicadores.png" width="900">
</p>

### Cadastro de Indicações

Fluxo simplificado para envio de oportunidades comerciais, permitindo cadastro de pessoas ou empresas interessadas e envio de documentos de apoio.

<p align="center">
  <img src="https://i.ibb.co/0ydS9Rgg/atomtech-indicado-1-indicadores.png" width="48%">
  <img src="https://i.ibb.co/1JLVGS5Y/atomtech-indicado02-indicadores.png" width="48%">
</p>

### Dashboard do Indicador

Área exclusiva onde o indicador acompanha suas indicações, status dos negócios, comissões recebidas e desempenho geral.

<p align="center">
  <img src="https://i.ibb.co/qYWvq7WK/atomtech-dashboard-Cliente-indicadores.png" width="900">
</p>

### Painel Administrativo

Central de gerenciamento para equipes internas, com controle de usuários, indicações, projetos, comissões, relatórios e permissões.

<p align="center">
  <img src="https://i.ibb.co/C5jBS2Mq/atomtech-admin-indicadores.png" width="900">
</p>

### Calculadoras Solares

Ferramentas técnicas utilizadas pela equipe comercial para estimativas de geração, dimensionamento de sistemas fotovoltaicos e cálculo da quantidade de painéis necessários para carregamento de veículos elétricos. Além do resultado final, a aplicação apresenta a memória de cálculo utilizada na estimativa.

<p align="center">
  <img src="https://i.ibb.co/hFDyqHsY/atomtech-calculo-indicadores.png" width="900">
</p>

---

## Arquitetura

```text
React + TypeScript
          ↓
    TanStack Query
          ↓
       Supabase
          ↓
 PostgreSQL + RLS
          ↓
    Edge Functions
```

A plataforma segue uma arquitetura Zero Trust, onde todas as regras críticas de negócio, permissões, cálculos de comissão e validações de acesso são executadas no backend.

📖 Documentação técnica completa: [Ver documentação técnica](docs/arquitetura.md)

---

## Status

Projeto ativo utilizado para gestão de indicações comerciais, acompanhamento de comissões e apoio à operação de vendas da AtomTech.
