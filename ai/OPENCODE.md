# OPENCODE.MD — Fiel? SaaS · Guia de Arquitetura para Implementação

> Este documento descreve o conceito, arquitetura e esqueleto técnico do SaaS **Fiel?**  
> Destina-se ao agente de desenvolvimento (OpenCode) que irá implementar o projeto.  
> Não contém código — apenas decisões de produto, fluxos, estrutura e contratos entre os módulos.

---

## 1. Visão Geral do Produto

**Fiel?** é um SaaS B2C onde usuárias contratam uma IA para testar a fidelidade de seus parceiros via Instagram.

A IA assume o perfil de uma jovem fictícia, inicia interação orgânica com o perfil indicado pela cliente e reporta o comportamento em tempo real através de um painel e via WhatsApp.

**Proposta de valor central:**  
A cliente insere o `@` do Instagram do parceiro, paga, e recebe um relatório com o comportamento dele durante a operação de 24h.

---

## 2. Diagrama de Módulos

```
┌─────────────────────────────────────────────────────────┐
│                     FRONT-END ESTÁTICO                  │
│                                                         │
│  fiel.top/              → Landing Page (HTML puro)      │
│  fiel.top/afiliados     → LP Afiliados  (HTML puro)     │
│                                                         │
│  Todos os CTAs apontam para → app.fiel.top              │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  APP — Next.js + TypeScript              │
│                  Hospedagem: Vercel                      │
│                                                         │
│  app.fiel.top/login             → Auth                  │
│  app.fiel.top/login?affiliate   → Auth c/ flag afiliado (vindo da LP) │
│  app.fiel.top/dashboard         → Painel da cliente     │
│  app.fiel.top/admin             → Painel operacional    │
│  app.fiel.top/checkout          → Seleção de plano      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                      BACKEND / API                      │
│           Node.js (ESM — type: module) + Mongoose       │
│                  Hospedagem: Render                      │
│                                                         │
│  /api/auth          → Autenticação (NextAuth — Google OAuth) │
│  /api/payments      → AbacatePay webhooks + checkout    │
│  /api/operations    → CRUD das operações (testes)       │
│  /api/timeline      → Atualização dos passos            │
│  /api/affiliates    → Gestão de afiliados               │
│  /api/notify        → Disparo WhatsApp (Z-API) + E-mail (Resend) │
└─────────────────────────┬───────────────────────────────┘
                          │
              ┌───────────┴────────────────┐
              ▼                            ▼
  ┌─────────────────┐     ┌───────────────────────────┐
  │    MongoDB      │     │  Mensageria                │
  │  (via Mongoose) │     │                            │
  │  MongoDB Atlas  │     │  WhatsApp → Z-API          │
  └─────────────────┘     │  E-mail   → Resend         │
                          └───────────────────────────┘
```

---

## 3. Módulos Detalhados

---

### 3.1 Landing Pages (HTML Estático)

**Arquivos:** `fiel-lp.html` e `fiel-afiliados.html`  
**Hospedagem sugerida:** Cloudflare Pages, Netlify ou GitHub Pages  
**Domínio:** `fiel.top`

Não há dependência de framework. São arquivos HTML/CSS/JS puros já desenvolvidos.

**Regra de CTA:**  
Todos os botões e links de ação das LPs devem apontar para:

| CTA | Destino |
|---|---|
| "Quero testar agora" | `https://app.fiel.top/login` |
| "Começar meu teste" | `https://app.fiel.top/login` |
| "Quero ser afiliado" | `https://fiel.top/?affiliate=true` |
| "Receber kit de afiliado" | `https://fiel.top/?affiliate=true` |

---

### 3.2 App — Next.js com TypeScript

**Stack:** Next.js 14+ (App Router), TypeScript, Tailwind CSS  
**Hospedagem:** Vercel  
**Domínio:** `app.fiel.top`

#### Rotas da aplicação

```
app/
├── (auth)/
│   ├── login/               → Login via Google OAuth
│   │                          Lê query param ?affiliate=true
│   │                          Se presente, marca usuário como candidato a afiliado
│   └── callback/            → Callback Google OAuth (NextAuth)
│
├── (dashboard)/
│   ├── dashboard/           → Painel da cliente logada
│   │   ├── page.tsx         → Lista de operações + status
│   │   └── nova/            → Formulário: inserir @ + escolher plano
│   │
│   ├── checkout/            → Tela de pagamento (AbacatePay — PIX)
│   │
│   └── operacao/[id]/       → Detalhe de uma operação
│       └── page.tsx         → Timeline dos 4 passos + relatório final
│
└── (admin)/
    └── admin/               → Acesso restrito (role: ADMIN)
        ├── page.tsx         → Lista de todas as operações ativas
        ├── operacao/[id]/   → Gerenciamento de uma operação
        │   └── page.tsx     → Atualizar passos da timeline manualmente
        ├── usuarios/        → Lista de clientes
        └── afiliados/       → Lista de candidatos a afiliado + aprovação
```

---

---

### 3.2b Backend — Node.js (API REST)

**Runtime:** Node.js  
**Hospedagem:** Render  
**Domínio:** `api.fiel.top` (ou subpath interno do Next.js conforme evolução)

#### ⚠️ Módulos ES — obrigatório usar `type: "module"`

O backend **deve** usar ES Modules (ESM), não CommonJS. Isso significa:

- O `package.json` do backend deve conter `"type": "module"`
- Usar `import` / `export` em todos os arquivos — **nunca** `require()` ou `module.exports`
- Extensões de arquivo: `.js` com sintaxe ESM, ou `.ts` compilado para ESM
- Ao usar TypeScript, configurar o `tsconfig.json` com `"module": "NodeNext"` e `"moduleResolution": "NodeNext"`

```json
// package.json — backend
{
  "name": "fiel-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

```json
// tsconfig.json — backend
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

> Atenção ao validar o webhook da AbacatePay: o `rawBody` deve ser lido **antes** de qualquer middleware de parsing. Em ESM com Express, usar `express.raw({ type: '*/*' })` especificamente na rota do webhook antes de `express.json()`.

---

### 3.3 Fluxo de Autenticação

**Solução:** NextAuth.js com provider Google OAuth  
**Sem senha** — o único método de autenticação é o Google. Não há cadastro por e-mail/senha.

1. Usuária acessa `app.fiel.top/login`
2. Clica em "Entrar com Google" → redirecionada para consentimento Google
3. Retorna via callback `/api/auth/callback/google`
4. NextAuth cria ou recupera o documento `User` no MongoDB com os dados do perfil Google (`name`, `email`, `image`)
5. Se veio com `?affiliate=true`, flag `isAffiliateCandidate: true` é salva no perfil
6. Após login, redireciona para `/dashboard`
7. Se não tiver créditos nem plano ativo → redireciona para `/checkout`

**Roles de usuário:**

| Role | Acesso |
|---|---|
| `USER` | `/dashboard` e `/operacao/[id]` |
| `ADMIN` | Tudo acima + `/admin/*` |
| `AFFILIATE` | Futuro: painel de afiliado com métricas |

---

### 3.4 Fluxo de Pagamento

**Solução:** AbacatePay (PIX)  
**Método disponível no MVP:** Apenas PIX  
**Cartão de crédito:** Fora do escopo do MVP  
**Documentação oficial:** https://docs.abacatepay.com

> **Atenção:** Valores monetários na API AbacatePay são sempre em **centavos**.  
> R$ 34,90 = `3490` · R$ 79,90 = `7990` · R$ 89,90 = `8990`

---

#### Produtos / valores:

| Produto | Tipo | Preço | Centavos |
|---|---|---|---|
| Crédito Avulso | PIX QRCode único | R$ 34,90 | `3490` |
| Pack 3 Créditos | PIX QRCode único | R$ 79,90 | `7990` |
| Plano Vigia | PIX QRCode mensal | R$ 89,90/mês | `8990` |

> **Nota sobre recorrência:** A AbacatePay gera cobranças PIX pontuais. Para o Plano Vigia, o backend deve controlar a data de expiração (`subscriptionExpiry`) e gerar uma nova cobrança automaticamente quando o vencimento se aproximar, notificando a cliente via WhatsApp com o QR Code de renovação.

---

#### Objeto PIX QRCode (retorno da API):

```json
{
  "id": "pix_char_mXTWdj6sABWnc4uL2Rh1r6tb",
  "amount": 3490,
  "status": "PENDING",
  "devMode": false,
  "method": "PIX",
  "brCode": "...",
  "brCodeBase64": "...",
  "platformFee": 80,
  "description": "Crédito Avulso — Fiel?",
  "metadata": {
    "userId": "abc123",
    "product": "CREDITO_AVULSO"
  },
  "createdAt": "2024-12-06T18:56:15.538Z",
  "updatedAt": "2024-12-06T18:56:15.538Z",
  "expiresAt": "2024-12-06T19:26:15.538Z"
}
```

**Campos relevantes para o frontend:**
- `brCode` → string Copia e Cola exibida para o usuário
- `brCodeBase64` → imagem do QR Code (renderizar como `<img src="data:image/png;base64,{brCodeBase64}">`)
- `expiresAt` → exibir countdown de expiração na tela de checkout

**Status possíveis:**

| Status | Significado |
|---|---|
| `PENDING` | Aguardando pagamento |
| `PAID` | Pago com sucesso |
| `EXPIRED` | PIX não pago dentro do prazo |
| `CANCELLED` | Cancelado manualmente |
| `REFUNDED` | Valor estornado ao cliente |

---

#### Fluxo de criação do PIX:

```
Cliente seleciona plano em /checkout
        ↓
Backend cria PIX QRCode via AbacatePay API
  POST https://api.abacatepay.com/v1/pixQrCode/create
  Headers: { Authorization: "Bearer ABACATEPAY_API_KEY" }
  Body: {
    amount: 3490,
    description: "Crédito Avulso — Fiel?",
    metadata: { userId: "...", product: "CREDITO_AVULSO" }
  }
        ↓
Retorno: { brCode, brCodeBase64, id, expiresAt }
        ↓
Frontend exibe QR Code + Copia e Cola + countdown de expiração
Backend salva Payment no MongoDB com status PENDING e pixQrCodeId
        ↓
Cliente paga o PIX
        ↓
AbacatePay dispara webhook → POST /api/payments/webhook
        ↓
Backend valida a requisição (dupla camada — ver abaixo)
        ↓
Lê metadata.userId e metadata.product do payload
        ↓
Backend credita na conta:
  CREDITO_AVULSO  → user.credits += 1
  PACK_3          → user.credits += 3
  VIGIA_MENSAL    → user.subscriptionStatus = "ACTIVE"
                    user.subscriptionExpiry = now + 30 dias
        ↓
Atualiza Payment no MongoDB para status PAID
        ↓
Dispara e-mail via Resend (payment_confirmed ou subscription_active)
Redireciona para /dashboard
```

---

#### Webhook — evento e validação:

O único evento relevante para pagamentos é **`billing.paid`**.

**Payload recebido (PIX QRCode):**
```json
{
  "id": "log_12345abcdef",
  "event": "billing.paid",
  "devMode": false,
  "data": {
    "payment": {
      "amount": 3490,
      "fee": 80,
      "method": "PIX"
    },
    "pixQrCode": {
      "id": "pix_char_mXTWdj6sABWnc4uL2Rh1r6tb",
      "amount": 3490,
      "kind": "PIX",
      "status": "PAID"
    }
  }
}
```

Para recuperar `userId` e `product`, buscar o `Payment` no MongoDB pelo campo `pixQrCodeId` correspondente ao `data.pixQrCode.id`.

**Validação de segurança — duas camadas obrigatórias:**

Camada 1 — Secret na query string (autenticação simples):
```typescript
// A URL do webhook deve ser configurada como:
// https://api.fiel.top/api/payments/webhook?webhookSecret=SEU_SECRET

app.post('/api/payments/webhook', (req, res) => {
  const webhookSecret = req.query.webhookSecret;
  if (webhookSecret !== process.env.ABACATEPAY_WEBHOOK_SECRET) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  // ...
});
```

Camada 2 — Assinatura HMAC-SHA256 no header `X-Webhook-Signature`:
```typescript
import crypto from 'node:crypto';

// Chave pública fixa fornecida pela AbacatePay
const ABACATEPAY_PUBLIC_KEY = "t9dXRhHHo3yDEj5pVDYz0frf7q6bMKyMRmxxCPIPp3RCplBfXRxqlC6ZpiWmOqj4L63qEaeUOtrCI8P0VMUgo6iIga2ri9ogaHFs0WIIywSMg0q7RmBfybe1E5XJcfC4IW3alNqym0tXoAKkzvfEjZxV6bE0oG2zJrNNYmUCKZyV0KZ3JS8Votf9EAWWYdiDkMkpbMdPggfh1EqHlVkMiTady6jOR3hyzGEHrIz2Ret0xHKMbiqkr9HS1JhNHDX9";

function verifyAbacateSignature(rawBody: string, signatureFromHeader: string): boolean {
  const expected = crypto
    .createHmac('sha256', ABACATEPAY_PUBLIC_KEY)
    .update(Buffer.from(rawBody, 'utf8'))
    .digest('base64');

  const A = Buffer.from(expected);
  const B = Buffer.from(signatureFromHeader);
  return A.length === B.length && crypto.timingSafeEqual(A, B);
}

// IMPORTANTE: ler o rawBody antes de qualquer middleware de parsing JSON
```

> Ambas as validações devem passar antes de processar qualquer evento. Registrar e retornar `401` em caso de falha.

**Outros eventos suportados (não relevantes para o MVP, mas existem):**
- `withdraw.done` — saque concluído
- `withdraw.failed` — saque falhou

---

#### Ambientes:

| Ambiente | Comportamento |
|---|---|
| Dev Mode | Webhooks e cobranças só recebem eventos do ambiente de testes. Usar chave API de dev. |
| Produção | Dados reais. Webhooks configurados em produção só recebem eventos reais. |

Testar pagamentos em dev mode usando o endpoint `POST /pixQrCode/simulate-payment` fornecido pela AbacatePay.

---

### 3.5 Fluxo de Operação (Teste de Fidelidade)

Uma **Operação** é a unidade central do produto. Cada teste iniciado é uma operação.

#### Estados da Operação:

```
AGUARDANDO_PAGAMENTO
        ↓
PAGA (crédito debitado)
        ↓
EM_ANDAMENTO
        ↓
CONCLUIDA | CANCELADA
```

#### Timeline — 4 Passos:

Cada operação possui uma timeline com 4 passos que o operador (admin) atualiza manualmente no painel:

```
Passo 1 — SEGUINDO
  Status: pendente | ativo | concluido
  Descrição: "IA seguiu o perfil e iniciou conexão"

Passo 2 — INTERAGINDO
  Status: pendente | ativo | concluido
  Descrição: "Curtidas e respostas de stories realizadas"

Passo 3 — CONVERSANDO
  Status: pendente | ativo | concluido
  Descrição: "Conversa no direct iniciada — buscando red flags"

Passo 4 — RELATORIO
  Status: pendente | ativo | concluido
  Descrição: "Relatório gerado e enviado"
  Campo extra: resultado (SEM_RED_FLAGS | RED_FLAG_LEVE | RED_FLAG_GRAVE)
```

Cada vez que o admin atualiza um passo, o sistema:
1. Atualiza o banco de dados
2. Envia notificação via WhatsApp para a cliente

---

### 3.6 Painel da Cliente (Dashboard)

**Rota:** `/dashboard`

**Componentes principais:**

- **Header** com saldo de créditos ou badge "Plano Vigia Ativo"
- **Botão "Novo teste"** → abre formulário para inserir `@Instagram` + confirma débito de crédito
- **Lista de operações** com card para cada uma:
  - @ da pessoa consultada
  - Status atual (badge colorido)
  - Data de início
  - Link "Ver detalhes"
- **Tela de detalhe da operação** com:
  - Timeline visual dos 4 passos (estilo stepper)
  - Cada passo com ícone de status (pendente / ativo / concluído)
  - Área de relatório final (visível quando Passo 4 concluído)

---

### 3.7 Painel Admin

**Rota:** `/admin` — protegida por role `ADMIN`

**Funcionalidades:**

#### Visão geral
- Contadores: operações ativas hoje, total de clientes, receita do mês
- Lista de todas as operações ordenadas por data

#### Gerenciamento de operação (`/admin/operacao/[id]`)
- Dados da operação: cliente, @ consultado, data
- **Controles da timeline:** botões para avançar cada passo manualmente
- Campo de observações internas (não visível para a cliente)
- Seletor de resultado final (Sem red flags / Red flag leve / Red flag grave)
- Botão "Enviar notificação manual" via WhatsApp

#### Gestão de clientes
- Lista com nome, e-mail, plano, créditos, data de cadastro
- Ação manual: adicionar créditos (para suporte/reembolso)

#### Gestão de afiliados
- Lista de candidatos vindos com `?affiliate=true`
- Ação: aprovar afiliado (futura integração com sistema de comissões)

---

### 3.8 Mensageria — WhatsApp + E-mail

O sistema usa dois canais de comunicação com a cliente, cada um com um papel distinto.

---

#### Canal 1 — WhatsApp (Z-API)

Canal operacional em tempo real. Usado durante e após a operação.

**Fluxo de disparo:**  
`Admin atualiza passo → /api/notify → Z-API → WhatsApp entregue`

| Gatilho | Mensagem |
|---|---|
| Operação criada | "Olá! Sua operação foi iniciada. Acompanhe em tempo real: app.fiel.top/dashboard" |
| Passo 1 concluído | "✅ Nossa IA já seguiu o perfil e iniciou a aproximação." |
| Passo 2 concluído | "✅ Interações realizadas. A conversa está sendo aberta." |
| Passo 3 concluído | "✅ Conversa em andamento. Estamos mapeando o comportamento." |
| Passo 4 — Sem red flags | "✅ Operação concluída. Nenhuma red flag detectada. Relatório disponível no painel." |
| Passo 4 — Red flag detectada | "🚨 Operação concluída. Red flag detectada. Acesse o relatório completo no painel." |
| Renovação Vigia | "Olá! Sua assinatura Vigia vence em 3 dias. Pague o PIX para continuar: [link]" |

---

#### Canal 2 — E-mail (Resend)

Canal transacional. Usado em momentos chave do ciclo de vida da cliente.  
**Solução:** Resend (`resend.com`) com templates em React Email.

> Templates a definir pelo produto. Os abaixo são referência inicial simples para o MVP.

| Gatilho | Template | Assunto sugerido |
|---|---|---|
| Cadastro realizado | `welcome` | "Bem-vinda ao Fiel? — sua conta está pronta" |
| Pagamento aprovado (crédito/pack) | `payment_confirmed` | "Pagamento confirmado — seu crédito está disponível" |
| Assinatura Vigia ativada | `subscription_active` | "Plano Vigia ativo — você pode testar quando quiser" |
| Assinatura Vigia renovada | `subscription_renewed` | "Assinatura renovada — mais um mês de paz de espírito" |
| Assinatura cancelada | `subscription_canceled` | "Sua assinatura foi cancelada" |

**Estrutura dos templates (MVP — simples):**  
Cada template contém: logo do Fiel?, título, texto principal, CTA com botão e rodapé com link de suporte. Sem HTML complexo — texto + botão + branding mínimo.

**Fluxo de disparo:**  
`Evento ocorre (pagamento, cadastro) → /api/notify → Resend API → E-mail entregue`

---

## 4. Modelo de Dados (Mongoose — referência)

**Banco:** MongoDB Atlas  
**ODM:** Mongoose  

```typescript
// models/User.ts
const UserSchema = new Schema({
  email:                { type: String, required: true, unique: true },
  name:                 { type: String },
  image:                { type: String },       // avatar do Google
  googleId:             { type: String, unique: true }, // sub do Google OAuth
  phone:                { type: String },       // WhatsApp para notificações (preenchido após cadastro)
  role:                 { type: String, enum: ['USER', 'ADMIN', 'AFFILIATE'], default: 'USER' },
  credits:              { type: Number, default: 0 },
  isAffiliateCandidate: { type: Boolean, default: false },

  // AbacatePay
  abacateCustomerId:    { type: String },
  subscriptionStatus:   { type: String, enum: ['ACTIVE', 'CANCELED', 'SUSPENDED'] },
  subscriptionExpiry:   { type: Date },

  createdAt: { type: Date, default: Date.now },
})

// models/Operation.ts
const OperationSchema = new Schema({
  userId:        { type: Schema.Types.ObjectId, ref: 'User', required: true },
  targetHandle:  { type: String, required: true },  // @ do Instagram consultado
  status:        {
    type: String,
    enum: ['AGUARDANDO_PAGAMENTO', 'PAGA', 'EM_ANDAMENTO', 'CONCLUIDA', 'CANCELADA'],
    default: 'PAGA'
  },
  result:        { type: String, enum: ['SEM_RED_FLAGS', 'RED_FLAG_LEVE', 'RED_FLAG_GRAVE'] },
  adminNotes:    { type: String },
  steps: [
    {
      number:      { type: Number, required: true },  // 1, 2, 3 ou 4
      status:      { type: String, enum: ['PENDENTE', 'ATIVO', 'CONCLUIDO'], default: 'PENDENTE' },
      completedAt: { type: Date },
    }
  ],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
})

// models/Payment.ts
const PaymentSchema = new Schema({
  userId:          { type: Schema.Types.ObjectId, ref: 'User', required: true },
  pixQrCodeId:     { type: String, required: true, unique: true }, // id retornado pela AbacatePay (pix_char_...)
  product:         { type: String, enum: ['CREDITO_AVULSO', 'PACK_3', 'VIGIA_MENSAL'] },
  amount:          { type: Number },      // em centavos (ex: 3490)
  platformFee:     { type: Number },      // taxa AbacatePay em centavos
  status:          { type: String, enum: ['PENDING', 'PAID', 'EXPIRED', 'CANCELLED', 'REFUNDED'], default: 'PENDING' },
  brCode:          { type: String },      // copia e cola
  brCodeBase64:    { type: String },      // base64 do QR Code para exibição
  expiresAt:       { type: Date },
  paidAt:          { type: Date },
  createdAt:       { type: Date, default: Date.now },
})
```

---

## 5. Variáveis de Ambiente Necessárias

```env
# App
NEXT_PUBLIC_APP_URL=https://app.fiel.top
NEXTAUTH_URL=https://app.fiel.top
NEXTAUTH_SECRET=

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# MongoDB
MONGODB_URI=mongodb+srv://...

# AbacatePay
ABACATEPAY_API_KEY=             # chave da API (dev ou produção)
ABACATEPAY_WEBHOOK_SECRET=      # secret configurado no dashboard da AbacatePay
# ABACATEPAY_PUBLIC_KEY é fixo e hardcoded no código de validação HMAC (ver seção 3.4)

# WhatsApp (Z-API)
ZAPI_INSTANCE_ID=
ZAPI_TOKEN=
ZAPI_BASE_URL=

# E-mail (Resend)
RESEND_API_KEY=
RESEND_FROM_EMAIL=noreply@fiel.top

# Admin
ADMIN_EMAIL=
```

---

## 6. Ordem de Implementação Sugerida (MVP)

A ordem abaixo prioriza ter o fluxo de dinheiro funcionando o mais rápido possível.

```
Fase 1 — Fundação
  [ ] Setup Next.js + TypeScript + Tailwind
  [ ] Conectar MongoDB Atlas via Mongoose
  [ ] Implementar autenticação Google OAuth com NextAuth.js
  [ ] Criar models: User, Operation, Payment

Fase 2 — Pagamentos
  [ ] Integrar AbacatePay API
  [ ] Criar rota de geração de cobrança PIX
  [ ] Exibir QR Code e Copia e Cola na tela de checkout
  [ ] Implementar webhook de confirmação de pagamento
  [ ] Creditar conta do usuário após pagamento confirmado
  [ ] Enviar QR Code via WhatsApp ao criar cobrança

Fase 3 — Core do Produto
  [ ] Formulário de nova operação (inserir @ + debitar crédito)
  [ ] Tela de dashboard com lista de operações
  [ ] Tela de detalhe com timeline dos 4 passos
  [ ] Painel admin com controle manual dos passos

Fase 4 — Notificações
  [ ] Integrar Z-API para WhatsApp
  [ ] Disparar mensagem WhatsApp ao criar operação
  [ ] Disparar mensagem WhatsApp a cada passo concluído pelo admin
  [ ] Integrar Resend para e-mail transacional
  [ ] Criar template `welcome` (boas-vindas ao cadastro)
  [ ] Criar template `payment_confirmed` (pagamento aprovado)
  [ ] Criar template `subscription_active` (Plano Vigia ativado)
  [ ] Criar template `subscription_renewed` (renovação mensal)

Fase 5 — Polimentos
  [ ] Lógica de expiração de assinatura Vigia (suspender acesso após vencimento)
  [ ] Recobrança mensal automática via AbacatePay + notificação WhatsApp
  [ ] Painel de afiliados (candidatos + aprovação)
  [ ] Relatório final formatado com resultado
```

---

## 7. Decisões Técnicas Abertas

Estes pontos ainda não foram decididos e devem ser resolvidos antes ou durante a implementação:

| Ponto | Opções | Decisão |
|---|---|---|
| Auth | Clerk vs NextAuth | **NextAuth.js + Google OAuth** — sem senha, sem Clerk |
| WhatsApp | Z-API vs Evolution API | **Z-API** (mais simples de integrar) |
| E-mail | Resend vs SendGrid | **Resend** + React Email para templates |
| Banco | MongoDB Atlas | **MongoDB Atlas** (free tier generoso, sem migrations) |
| Pagamento | AbacatePay PIX | **AbacatePay** — único método no MVP |
| Deploy Next.js | Vercel | **Vercel** para o app Next.js |
| Deploy Backend | Render | **Render** para o backend Node.js |

---

## 8. Fora do Escopo do MVP

As funcionalidades abaixo **não devem ser implementadas agora** para não atrasar a validação:

- **Login por e-mail e senha** — autenticação exclusivamente via Google OAuth
- **Pagamento via cartão de crédito** — apenas PIX no MVP (AbacatePay)
- Painel dedicado para afiliados com tracking de cliques e comissões automáticas
- IA autônoma operando o Instagram (MVP é operado manualmente pelo admin)
- App mobile
- Relatório em PDF gerado automaticamente
- Sistema de indicação com código personalizado
- Multi-idioma

---

## 9. Casos de Uso e Testes de Comportamento

Esta seção define os comportamentos esperados do sistema em cenários felizes e de borda. Tanto o backend quanto o frontend **devem cumprir todos os casos abaixo** antes de considerar um módulo funcional.

> Convenção: ✅ = comportamento esperado obrigatório · ❌ = comportamento proibido · ⚠️ = validação necessária

---

### UC-01 — Autenticação com Google OAuth

#### UC-01.1 — Primeiro acesso (cadastro)
**Ator:** Usuária sem conta  
**Pré-condição:** Nunca acessou o sistema

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Acessa `app.fiel.top/login` | ✅ Exibe botão "Entrar com Google" sem campos de senha |
| 2 | Clica em "Entrar com Google" | ✅ Redireciona para consentimento Google |
| 3 | Cancela consentimento no Google | ✅ Retorna para `/login` com mensagem de erro amigável, sem quebrar a tela |
| 4 | Autoriza e retorna | ✅ Conta criada no MongoDB com `name`, `email`, `image`, `googleId` |
| 5 | Após criação | ✅ E-mail `welcome` enviado via Resend |
| 6 | Redirecionamento | ✅ Sem créditos/plano → vai para `/checkout`. Com créditos → vai para `/dashboard` |

#### UC-01.2 — Retorno (login existente)
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Faz login com mesmo e-mail Google | ✅ Conta existente reconhecida, **não cria duplicata** |
| 2 | Redirecionamento | ✅ Vai direto para `/dashboard` |

#### UC-01.3 — Acesso via link de afiliado
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Acessa `fiel.top/?affiliate=true` e clica no CTA → chega em `app.fiel.top/login?affiliate=true` | ✅ Flag `isAffiliateCandidate: true` salva no perfil após login |
| 2 | Acessa sem o parâmetro | ✅ Flag permanece `false`, não sobrescreve valor existente |

#### UC-01.4 — Sessão expirada
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Sessão expira durante uso | ✅ Próxima ação protegida redireciona para `/login` |
| 2 | Após novo login | ✅ Retorna para a rota que tentou acessar (redirect preservado) |

---

### UC-02 — Pagamento PIX (AbacatePay)

#### UC-02.1 — Compra de Crédito Avulso (fluxo feliz)
**Pré-condição:** Usuária logada, sem créditos

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Acessa `/checkout` e seleciona Crédito Avulso | ✅ Backend cria QR Code via AbacatePay (`amount: 3490`) |
| 2 | Frontend exibe QR Code | ✅ `brCodeBase64` renderizado como imagem + `brCode` copiável |
| 3 | Frontend exibe countdown | ✅ Contador regressivo baseado em `expiresAt` |
| 4 | Usuária paga o PIX | ✅ AbacatePay dispara webhook `billing.paid` |
| 5 | Backend valida webhook | ✅ Camada 1: `webhookSecret` na query string · Camada 2: HMAC-SHA256 no header |
| 6 | Validação falha | ❌ Nunca processar crédito com assinatura inválida — retornar `401` |
| 7 | Crédito creditado | ✅ `user.credits += 1` via `findOneAndUpdate` atômico |
| 8 | E-mail disparado | ✅ Template `payment_confirmed` enviado via Resend |
| 9 | Redirecionamento | ✅ Usuária vai para `/dashboard` com crédito visível |

#### UC-02.2 — Compra do Pack 3 Créditos
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Seleciona Pack 3 (`amount: 7990`) | ✅ QR Code gerado corretamente |
| 2 | Paga e webhook processado | ✅ `user.credits += 3` — exatamente 3, nem mais nem menos |

#### UC-02.3 — Assinatura Plano Vigia
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Seleciona Vigia (`amount: 8990`) | ✅ QR Code gerado |
| 2 | Webhook processado | ✅ `subscriptionStatus: "ACTIVE"` + `subscriptionExpiry = now + 30 dias` |
| 3 | E-mail disparado | ✅ Template `subscription_active` enviado |

#### UC-02.4 — Webhook duplicado (mesmo `pixQrCode.id`)
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Webhook `billing.paid` chega pela segunda vez com mesmo `pixQrCode.id` | ✅ `findOneAndUpdate({ pixQrCodeId, status: 'PENDING' })` retorna `null` |
| 2 | Resultado | ✅ Crédito **não** adicionado novamente · Retorna `200` silenciosamente |
| 3 | Proibido | ❌ Nunca processar dois créditos para o mesmo `pixQrCode.id` |

#### UC-02.5 — PIX expirado sem pagamento
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuária não paga dentro do prazo | ✅ `expiresAt` passa, status do Payment atualizado para `EXPIRED` |
| 2 | Usuária tenta usar o mesmo QR Code | ✅ Frontend exibe mensagem "QR Code expirado" com opção de gerar novo |
| 3 | Crédito | ✅ Nenhum crédito adicionado |

#### UC-02.6 — Usuária fecha tela de checkout antes de pagar
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Fecha ou navega antes do pagamento | ✅ Payment permanece `PENDING` no banco |
| 2 | Retorna ao checkout | ✅ Sistema verifica se há Payment `PENDING` ativo e rexibe o mesmo QR Code (se não expirado) |

---

### UC-03 — Criação de Operação (Teste de Fidelidade)

#### UC-03.1 — Criação com crédito disponível (fluxo feliz)
**Pré-condição:** Usuária logada com `credits >= 1` ou `subscriptionStatus: ACTIVE`

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Clica em "Novo Teste" | ✅ Formulário exibe campo para `@Instagram` |
| 2 | Submete com `@` válido | ✅ Frontend desabilita botão imediatamente após clique |
| 3 | Backend recebe requisição | ✅ Valida `credits >= 1` antes de debitar |
| 4 | Crédito debitado | ✅ `user.credits -= 1` atomicamente |
| 5 | Operação criada | ✅ 4 passos criados com status `PENDENTE` |
| 6 | Notificação | ✅ WhatsApp disparado: "Sua operação foi iniciada" |
| 7 | Redirecionamento | ✅ Usuária vai para detalhe da operação com timeline |

#### UC-03.2 — Tentativa sem créditos
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Clica em "Novo Teste" sem créditos | ✅ Frontend bloqueia e exibe "Você não tem créditos disponíveis" |
| 2 | CTA | ✅ Botão "Recarregar créditos" leva para `/checkout` |
| 3 | Proibido | ❌ Backend nunca cria operação com `credits = 0` mesmo que frontend falhe |

#### UC-03.3 — @ do Instagram inválido
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Campo vazio | ✅ Frontend bloqueia submit com mensagem de validação |
| 2 | `@` com caracteres inválidos | ✅ Frontend valida regex de handle do Instagram antes de enviar |
| 3 | Backend recebe `@` inválido | ✅ Backend também valida e retorna `400` — crédito não debitado |
| 4 | Operação já criada mas `@` invalidado pelo admin após início | ✅ Admin cancela operação → crédito devolvido automaticamente (`user.credits += 1`) |

#### UC-03.4 — Race condition (duplo clique / dupla requisição)
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuária clica duas vezes rapidamente | ✅ Frontend desabilita botão após primeiro clique — segunda requisição não enviada |
| 2 | Duas requisições chegam ao backend mesmo assim | ✅ Apenas a primeira com `credits >= 1` é processada — a segunda retorna `402 Payment Required` |

---

### UC-04 — Atualização da Timeline (Admin)

#### UC-04.1 — Avanço de passo (fluxo feliz)
**Pré-condição:** Admin logado, operação com status `EM_ANDAMENTO`

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Admin avança Passo 1 | ✅ `step.status = 'CONCLUIDO'`, Passo 2 vai para `ATIVO` |
| 2 | WhatsApp disparado | ✅ Mensagem correspondente ao passo enviada à cliente |
| 3 | Frontend da cliente atualiza | ✅ Timeline reflete novo status (polling ou refresh) |

#### UC-04.2 — Tentativa de pular passo
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Admin tenta avançar Passo 3 sem Passo 2 concluído | ✅ Backend retorna `400` — passos devem ser sequenciais |
| 2 | Frontend | ✅ Botões de passos futuros ficam desabilitados enquanto anterior não está `CONCLUIDO` |

#### UC-04.3 — Conclusão de operação (Passo 4)
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Admin avança para Passo 4 sem definir `result` | ✅ Backend retorna `400` — campo `result` obrigatório no Passo 4 |
| 2 | Admin define `result` e conclui | ✅ `operation.status = 'CONCLUIDA'`, `operation.result` salvo |
| 3 | WhatsApp disparado | ✅ Mensagem de acordo com resultado (sem red flags ou red flag detectada) |
| 4 | Tentativa de reeditar operação concluída | ✅ Backend retorna `403` — operação `CONCLUIDA` é imutável |

#### UC-04.4 — Acesso não autorizado ao painel admin
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuário com role `USER` acessa `/admin` | ✅ Redirecionado para `/dashboard` com erro `403` |
| 2 | Requisição direta à API admin sem role `ADMIN` | ✅ Backend retorna `403` em qualquer rota `/admin/*` |

---

### UC-05 — Painel da Cliente

#### UC-05.1 — Visualização de operação de outro usuário
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuária acessa `/operacao/[id]` de outra usuária via URL | ✅ Backend verifica `operation.userId === session.userId` |
| 2 | IDs não batem | ✅ Retorna `403` — frontend exibe "Operação não encontrada" |

#### UC-05.2 — Estados vazios
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Dashboard sem nenhuma operação | ✅ Exibe estado vazio com CTA claro "Iniciar primeiro teste" |
| 2 | Erro de rede ao carregar | ✅ Exibe mensagem de erro com botão "Tentar novamente" — não quebra a UI |

---

### UC-06 — Mensageria

#### UC-06.1 — Falha no WhatsApp
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Z-API retorna erro ao disparar mensagem | ✅ Erro logado no servidor |
| 2 | Operação | ✅ Continua normalmente — falha de WhatsApp não bloqueia nenhum fluxo |
| 3 | Admin notificado | ✅ Painel admin exibe flag de "Falha na notificação WhatsApp" na operação |
| 4 | Usuária sem número cadastrado | ✅ WhatsApp pulado silenciosamente, apenas e-mail é enviado |

#### UC-06.2 — Falha no Resend (e-mail)
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Resend retorna erro | ✅ Erro logado no servidor |
| 2 | Fluxo principal | ✅ Não é bloqueado — pagamento e crédito já foram processados antes do e-mail |
| 3 | Admin notificado | ✅ Painel admin exibe flag de "Falha no envio de e-mail" quando aplicável |

---

---

## 10. Módulo de Afiliados

### 10.1 Modelos de Dados — Afiliado

Além dos modelos existentes, o módulo de afiliados requer os seguintes documentos no MongoDB:

```typescript
// models/AffiliateSession.ts
// Registra cada clique no link de afiliado (sem deduplicação)
const AffiliateSessionSchema = new Schema({
  affiliateUserId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  refCode:         { type: String, required: true },   // userId do afiliado usado no ?ref=
  ip:              { type: String },
  userAgent:       { type: String },
  referrer:        { type: String },                   // de onde veio o clique
  createdAt:       { type: Date, default: Date.now },
})

// models/AffiliateConversion.ts
// Registra cada usuário que se cadastrou e/ou pagou via link de afiliado
const AffiliateConversionSchema = new Schema({
  affiliateUserId:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  convertedUserId:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  refCode:          { type: String, required: true },
  type:             { type: String, enum: ['SIGNUP', 'PAYMENT'] }, // cadastro ou pagamento
  paymentId:        { type: Schema.Types.ObjectId, ref: 'Payment' }, // preenchido se type=PAYMENT
  commissionAmount: { type: Number },  // em centavos (30% do valor pago)
  createdAt:        { type: Date, default: Date.now },
})

// models/AffiliatePayout.ts
// Registra cada saque processado pelo admin
const AffiliatePayoutSchema = new Schema({
  affiliateUserId:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  amount:           { type: Number, required: true },  // valor em centavos
  status:           { type: String, enum: ['PENDING', 'PAID'], default: 'PENDING' },
  pixKey:           { type: String },       // chave PIX do afiliado
  processedAt:      { type: Date },         // data que o admin marcou como pago
  processedBy:      { type: String },       // email do admin que processou
  referenceMonth:   { type: String },       // ex: "2025-01" — mês de referência
  createdAt:        { type: Date, default: Date.now },
})
```

**Campos adicionais no modelo `User` para afiliados:**

```typescript
// Adicionar ao UserSchema existente:
refCode:              { type: String, unique: true, sparse: true }, // gerado na aprovação do admin
pixKey:               { type: String },      // chave PIX para receber comissões
pendingCommission:    { type: Number, default: 0 }, // saldo de comissão acumulado em centavos
totalCommissionPaid:  { type: Number, default: 0 }, // total já sacado historicamente
```

---

### 10.2 Rotas de API — Afiliados

```
/api/affiliates/
  GET  /panel              → KPIs do afiliado logado (sessões, conversões, faturamento, próximo saque)
  GET  /link               → Retorna o link e refCode do afiliado logado
  POST /track              → Registra sessão ao clicar no link (chamado pela LP)

/api/admin/affiliates/
  GET  /candidates         → Lista usuários com isAffiliateCandidate: true aguardando aprovação
  POST /approve/:userId    → Aprova afiliado: gera refCode, muda role para AFFILIATE
  GET  /payouts            → Lista saques pendentes do mês
  POST /payouts/:userId    → Admin marca saque como processado (PAID)
```

---

### 10.3 Painel do Afiliado — Especificação de Tela

**Rota:** `app.fiel.top/affiliate`  
**Acesso:** exclusivo para usuários com `role: AFFILIATE`  
**Tela separada** do dashboard de operações — menu/rota diferente

#### Seções da tela:

**1. Meu link de afiliado**
- Exibe o link completo: `fiel.top/?ref={refCode}`
- Botão "Copiar link" com feedback visual (copiado!)
- Badge de status: `Ativo` / `Pendente de aprovação`

**2. KPIs — cards no topo**

| KPI | Fonte | Descrição |
|---|---|---|
| Sessões | `AffiliateSession.count({ affiliateUserId })` | Total de cliques no link (sem dedup) |
| Convertidos | `AffiliateConversion.count({ type: 'PAYMENT' })` | Usuários que pagaram via link |
| Taxa de conversão | `convertidos / sessões * 100` | Calculado no frontend |
| Faturamento gerado | Soma dos `commissionAmount` de todas as conversões | Total em R$ |
| Comissão acumulada | `user.pendingCommission` | Saldo disponível para saque |
| Total já sacado | `user.totalCommissionPaid` | Histórico |

**3. Próximo saque**
- Data fixa: **dia 10 do próximo mês**
- Valor estimado: `user.pendingCommission` em R$
- Campo para cadastrar/editar chave PIX
- Status do último saque (PENDING ou PAID)

**4. Histórico de saques**
- Tabela: mês de referência · valor · status · data de processamento

---

### UC-08 — Cadastro e Aprovação de Afiliado

#### UC-08.1 — Cadastro via link de afiliado (fluxo feliz)
**Pré-condição:** Criador de conteúdo acessa `fiel.top/?affiliate=true` (LP principal)

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Acessa LP com `?affiliate=true` | ✅ LP carrega normalmente; parâmetro salvo em cookie ou propagado via CTA |
| 2 | Clica em CTA da LP | ✅ Redirecionado para `app.fiel.top/login?affiliate=true` com parâmetro preservado |
| 3 | Conclui login com Google | ✅ Conta criada com `isAffiliateCandidate: true` e `role: USER` |
| 4 | Crédito de boas-vindas | ✅ `user.credits = 1` adicionado imediatamente no cadastro |
| 5 | E-mail disparado | ✅ Template `welcome` enviado (mencionar que candidatura de afiliado está em análise) |
| 6 | Redirecionamento | ✅ Vai para `/dashboard` como usuário normal — painel afiliado ainda não disponível |
| 7 | Proibido | ❌ `role: AFFILIATE` nunca atribuído automaticamente no cadastro — exige aprovação admin |

#### UC-08.2 — Cadastro com `?affiliate=true` em conta já existente
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuária já cadastrada acessa link de afiliado | ✅ Login normal, `isAffiliateCandidate` atualizado para `true` |
| 2 | Crédito extra | ❌ Crédito de boas-vindas **não** dado novamente — apenas para novos cadastros |

#### UC-08.3 — Aprovação pelo admin
**Pré-condição:** Admin acessa `/admin/afiliados` e vê candidato

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Admin visualiza candidatos | ✅ Lista com nome, e-mail, data de cadastro, seguidores informados |
| 2 | Admin clica "Aprovar" | ✅ Backend: `role = 'AFFILIATE'`, `refCode` único gerado (ex: hash curto do userId) |
| 3 | `refCode` já existe | ✅ Gerar novo até encontrar um único — colisão tratada |
| 4 | Link gerado | ✅ `fiel.top/?ref={refCode}` disponível no painel do afiliado |
| 5 | E-mail disparado | ✅ Notificar afiliado aprovado com o link e instruções |
| 6 | Painel liberado | ✅ Afiliado passa a enxergar rota `/affiliate` no menu |

---

### UC-09 — Rastreamento de Sessão e Conversão

#### UC-09.1 — Clique no link de afiliado
**Pré-condição:** Afiliado aprovado com `refCode` válido

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Visitante acessa `fiel.top/?ref=ABC123` | ✅ LP carrega normalmente |
| 2 | Rastreamento | ✅ Requisição `POST /api/affiliates/track` disparada com `refCode`, IP e user-agent |
| 3 | Sessão registrada | ✅ Documento `AffiliateSession` criado — sem deduplicação, cada clique conta |
| 4 | Cookie salvo | ✅ `ref=ABC123` salvo em cookie com validade de 90 dias no browser do visitante |
| 5 | `refCode` inválido ou inexistente | ✅ LP carrega normalmente — sessão **não** registrada, sem erro visível |

#### UC-09.2 — Visitante se cadastra após clicar no link
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Visitante acessa login via LP com cookie `ref` ativo | ✅ `?ref=ABC123` propagado para a URL de login |
| 2 | Conclui cadastro com Google | ✅ `AffiliateConversion` criado com `type: 'SIGNUP'` |
| 3 | `refCode` não mais no cookie mas usuário já convertido | ✅ Conversão de pagamento ainda atribuída ao afiliado pelo `convertedUserId` |
| 4 | Afiliado tenta indicar a si mesmo | ✅ Backend detecta `refCode === user.refCode` — conversão **não** registrada |

#### UC-09.3 — Usuário convertido realiza pagamento
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuário que veio via link de afiliado paga qualquer plano | ✅ `AffiliateConversion` criado com `type: 'PAYMENT'` e `commissionAmount = amount * 0.30` |
| 2 | Comissão creditada | ✅ `affiliateUser.pendingCommission += commissionAmount` |
| 3 | Usuário faz segundo pagamento (Pack ou novo Vigia) | ✅ Nova `AffiliateConversion` gerada e comissão acumulada novamente |
| 4 | Pagamento estornado (`REFUNDED`) | ✅ `pendingCommission` decrementado pelo valor da comissão correspondente |

---

### UC-10 — Painel do Afiliado

#### UC-10.1 — Acesso ao painel
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Usuário com `role: USER` acessa `app.fiel.top/affiliate` | ✅ Redirecionado para `/dashboard` com erro 403 |
| 2 | Usuário com `role: AFFILIATE` acessa | ✅ Painel carrega com KPIs e link |
| 3 | Painel sem nenhuma sessão ainda | ✅ KPIs zerados com estado vazio amigável — não quebra a UI |

#### UC-10.2 — Cópia do link de afiliado
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Clica em "Copiar link" | ✅ `fiel.top/?ref={refCode}` copiado para clipboard via `navigator.clipboard` |
| 2 | Feedback visual | ✅ Botão muda para "Copiado!" por 2 segundos, depois volta ao estado original |
| 3 | Browser sem suporte a clipboard API | ✅ Fallback: exibe o link em campo de texto selecionável |

#### UC-10.3 — Visualização de KPIs
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Afiliado carrega o painel | ✅ Sessões, convertidos e faturamento carregados via `GET /api/affiliates/panel` |
| 2 | Taxa de conversão | ✅ Calculada no frontend: `(convertidos / sessões * 100).toFixed(1)%` |
| 3 | Sessões = 0 | ✅ Taxa de conversão exibe `—` (não divide por zero) |
| 4 | Erro de rede | ✅ KPIs exibem `—` com ícone de erro — não quebra a tela |

#### UC-10.4 — Cadastro de chave PIX
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Afiliado acessa campo de chave PIX | ✅ Campo editável com tipos: CPF, e-mail, telefone ou chave aleatória |
| 2 | Salva chave PIX | ✅ `user.pixKey` atualizado no backend |
| 3 | Chave não cadastrada no dia 10 | ✅ Admin visualiza alerta de "Chave PIX não cadastrada" no painel de saques |

---

### UC-11 — Processamento de Saque

#### UC-11.1 — Visibilidade do próximo saque
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Afiliado acessa painel | ✅ "Próximo saque: dia 10 de {mês seguinte}" exibido com valor de `pendingCommission` |
| 2 | `pendingCommission = 0` | ✅ Exibe "Nenhum saldo acumulado ainda" — sem confundir com saque zerado |

#### UC-11.2 — Processamento manual pelo admin (dia 10)
**Pré-condição:** Admin acessa `/admin/afiliados/saques`

| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Admin visualiza lista de saques do mês | ✅ Todos afiliados com `pendingCommission > 0` listados com nome, valor e chave PIX |
| 2 | Afiliado sem chave PIX cadastrada | ✅ Exibir alerta — admin não consegue processar sem chave cadastrada |
| 3 | Admin marca saque como processado | ✅ `AffiliatePayout` criado com `status: PAID` e `processedAt: now` |
| 4 | Após marcar como pago | ✅ `user.pendingCommission = 0` · `user.totalCommissionPaid += amount` |
| 5 | E-mail para o afiliado | ✅ Notificação de "Seu saque foi processado — R$ X via PIX" |
| 6 | Saque processado duas vezes pelo admin (erro humano) | ✅ Sistema verifica se já existe `AffiliatePayout` com mesmo `referenceMonth` e `affiliateUserId` com `status: PAID` — bloqueia com aviso |

#### UC-11.3 — Histórico de saques no painel do afiliado
| # | Ação | Comportamento esperado |
|---|---|---|
| 1 | Afiliado visualiza histórico | ✅ Tabela com: mês de referência, valor, status (Pago / Pendente), data de processamento |
| 2 | Saque pendente do mês atual | ✅ Exibido com status `Pendente` — data de processamento vazia |
| 3 | Nenhum saque ainda | ✅ Estado vazio com mensagem "Seus saques aparecerão aqui" |

---

### UC-07 atualizado — Resumo completo de decisões de produto

| Decisão | Comportamento definido |
|---|---|
| Webhook duplicado | `findOneAndUpdate({ pixQrCodeId, status: 'PENDING' })` — atômico, sem duplo crédito |
| @ inválido após criação | Crédito devolvido automaticamente pelo admin ao cancelar a operação |
| Race condition no botão | Frontend desabilita após primeiro clique; backend valida crédito antes de debitar |
| Falha de mensageria | Erro silencioso logado + flag de alerta visível no painel admin |
| Crédito de afiliado | 1 crédito dado imediatamente no cadastro com `?affiliate=true` — apenas para novos usuários |
| Role de afiliado | Nunca atribuído automaticamente — exige aprovação manual do admin |
| Link de afiliado | `fiel.top/?ref={refCode}` — aponta para LP com cookie de 90 dias |
| Sessões | Cada clique contabilizado sem deduplicação |
| Comissão | 30% de cada pagamento do usuário convertido, acumulada em `pendingCommission` |
| Saque | Admin processa manualmente todo dia 10 via PIX — protegido contra duplicata por mês |
| Auto-indicação | Afiliado não pode converter a si mesmo — detectado no backend pelo `refCode` |





*Documento gerado como guia de arquitetura para implementação com OpenCode.*  
*Versão: 1.0 — MVP*
