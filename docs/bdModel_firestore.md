# Modelo de Banco de Dados — Firestore

## 1. Visão Geral

Este documento detalha a modelagem de dados a ser implementada no Firestore para a Fase 1 do projeto. O Firestore é um banco NoSQL orientado a documentos, organizado em **coleções** (equivalentes a "tabelas") e **documentos** (equivalentes a "registros"), que podem conter subcoleções.

**Princípios de modelagem adotados:**

- Priorizar dados **desnormalizados** quando o ganho de performance de leitura compensar (padrão comum em Firestore, diferente de um banco relacional).
- Gravações no catálogo de produtos/preços acontecem **somente via Cloud Functions**, nunca diretamente do app — o app só lê.
- IDs de documentos devem ser gerados automaticamente pelo Firestore (`auto-id`), exceto quando um ID natural e estável já existir (ex.: CNPJ de loja).

## 2. Diagrama de Relacionamento (alto nível)

```
users ──────────┐
                 │ userId
                 ▼
             receipts ──────────► stores
                 │  storeId          ▲
                 │ receiptId         │ storeId
                 ▼                   │
          receiptItems ──────────────┘
                 │ productId
                 ▼
             products
                 │ productId
                 ▼
              prices ──────────► stores (storeId)
```

## 3. Coleções

### 3.1 `users`

Armazena os dados do usuário autenticado.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `uid` | string | ✅ | Mesmo ID do Firebase Auth (usado como ID do documento) |
| `nome` | string | ✅ | Nome do usuário |
| `email` | string | ✅ | E-mail (via Firebase Auth) |
| `cidade` | string | ❌ | Cidade do usuário (ex.: "Salvador") |
| `bairro` | string | ❌ | Usado futuramente para segmentação geográfica |
| `criadoEm` | timestamp | ✅ | Data de criação da conta |
| `ultimoAcessoEm` | timestamp | ❌ | Última vez que o usuário usou o app |

> **ID do documento:** usar o `uid` do Firebase Auth diretamente.

---

### 3.2 `stores`

Estabelecimentos comerciais identificados a partir das notas fiscais.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `cnpj` | string | ✅ | CNPJ do estabelecimento (usado como ID do documento) |
| `nomeFantasia` | string | ✅ | Nome comercial da loja |
| `razaoSocial` | string | ❌ | Nome jurídico completo |
| `endereco` | map | ✅ | `{ rua, numero, bairro, cidade, uf, cep }` |
| `geolocalizacao` | geopoint | ❌ | Latitude/longitude para busca por proximidade |
| `criadoEm` | timestamp | ✅ | Data do primeiro registro dessa loja no sistema |

> **ID do documento:** usar o CNPJ (normalizado, só números) como ID — evita duplicidade de lojas.

---

### 3.3 `products`

Catálogo normalizado de produtos (o coração da comparação de preços).

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `nomeNormalizado` | string | ✅ | Nome padronizado do produto (ex.: "Arroz Tipo 1 5kg") |
| `nomesOriginais` | array\<string\> | ❌ | Variações de nome vindas das notas fiscais, para melhorar matching futuro |
| `categoria` | string | ✅ | Categoria (ex.: "Grãos", "Higiene", "Limpeza") |
| `marca` | string | ❌ | Marca do produto, se identificável |
| `unidade` | string | ✅ | Unidade de medida (ex.: "kg", "un", "L") |
| `quantidade` | number | ❌ | Quantidade referente à unidade (ex.: 5 para "5kg") |
| `criadoEm` | timestamp | ✅ | Data de criação do registro |

> **Ponto de atenção:** este é o dado mais sensível do sistema em termos de qualidade — o algoritmo de normalização/matching (rodando nas Cloud Functions) determina se dois nomes de produtos diferentes serão tratados como o mesmo item.

---

### 3.4 `prices`

Histórico de preços coletados por produto e por loja. Volume alto de escrita — desenhado para consultas rápidas de comparação.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `productId` | reference | ✅ | Referência ao documento em `products` |
| `storeId` | reference | ✅ | Referência ao documento em `stores` |
| `valor` | number | ✅ | Preço pago, em reais |
| `dataColeta` | timestamp | ✅ | Data/hora da compra (extraída da nota fiscal) |
| `receiptId` | reference | ✅ | Referência à nota fiscal de origem (rastreabilidade) |
| `origem` | string | ✅ | Ex.: `"nfce_qrcode"` — permite adicionar outras origens no futuro |

> **Índice recomendado:** composto em `productId` + `dataColeta` (para buscar o histórico de preço de um produto ordenado por data) e `productId` + `storeId` (para comparar preços entre lojas de um mesmo produto).

---

### 3.5 `receipts`

Registro de cada nota fiscal (NFC-e) processada.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | reference | ✅ | Referência ao usuário que enviou |
| `storeId` | reference | ✅ | Referência à loja da compra |
| `chaveAcesso` | string | ✅ | Chave de acesso da NFC-e (44 dígitos) — usada como ID do documento para evitar duplicidade |
| `dataCompra` | timestamp | ✅ | Data da compra |
| `valorTotal` | number | ❌ | Valor total da nota |
| `status` | string | ✅ | `"pendente"` \| `"processada"` \| `"erro"` |
| `criadoEm` | timestamp | ✅ | Data de envio pelo usuário |
| `erroDetalhe` | string | ❌ | Preenchido apenas se `status = "erro"` |

> **ID do documento:** usar a `chaveAcesso` da nota fiscal — garante idempotência (a mesma nota não pode ser processada duas vezes).

**Subcoleção:** `receipts/{receiptId}/items`

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `productId` | reference | ✅ | Produto identificado nesse item |
| `nomeOriginal` | string | ✅ | Nome exatamente como veio na nota fiscal |
| `quantidade` | number | ✅ | Quantidade comprada |
| `valorUnitario` | number | ✅ | Preço unitário pago |

## 4. Regras de Segurança (Firestore Rules) — diretrizes

```
- users/{userId}: leitura e escrita apenas pelo próprio usuário autenticado.
- stores: leitura pública (ou por qualquer usuário autenticado); escrita bloqueada para o client (somente Cloud Functions).
- products: leitura pública; escrita bloqueada para o client.
- prices: leitura pública; escrita bloqueada para o client.
- receipts/{receiptId}: leitura e criação apenas pelo usuário dono (userId == request.auth.uid); atualização de status bloqueada para o client.
- receipts/{receiptId}/items: mesma regra da nota fiscal pai.
```

> Regra geral: **nenhuma escrita em `products`, `prices` ou `stores` deve ser permitida diretamente pelo app** — isso passa exclusivamente pelas Cloud Functions (`parseReceipt` → `validateReceipt` → gravação), garantindo integridade do catálogo.

## 5. Índices Compostos Necessários

| Coleção | Campos do índice | Motivo |
|---------|------------------|--------|
| `prices` | `productId` (Asc) + `dataColeta` (Desc) | Histórico de preço de um produto, mais recente primeiro |
| `prices` | `productId` (Asc) + `valor` (Asc) | Encontrar o menor preço atual de um produto |
| `receipts` | `userId` (Asc) + `criadoEm` (Desc) | Listar notas fiscais enviadas por um usuário |

## 6. Considerações para Fases Futuras

| Fase | Impacto no modelo |
|------|-------------------|
| **Fase 2 (social)** | Novas coleções: `posts`, `comments`, `follows`, `reviews` — provavelmente referenciando `users` e `products` |
| **Fase 3 (multi-estado)** | Adicionar campo `uf` em `stores` e possivelmente em `prices` (via desnormalização) para permitir filtragem eficiente por estado sem precisar de join |

---

*Este modelo deve ser validado na prática assim que os primeiros QR Codes reais forem processados (Sprint 0/1), especialmente os campos extraídos do XML/HTML da NFC-e retornado pelo portal SEFAZ-BA.*