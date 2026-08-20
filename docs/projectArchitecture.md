# Arquitetura do Projeto — Comparador de Preços (Salvador, BA)

## 1. Visão Geral

O aplicativo é uma plataforma mobile de comparação de preços de produtos de supermercado, com ingestão de dados via leitura de QR Code de notas fiscais eletrônicas (NFC-e) através do portal da SEFAZ-BA. O diferencial competitivo em relação ao Preço da Hora BA está na experiência de uso (UX/UI) e, nas fases seguintes, em uma camada social/comunitária — algo estruturalmente difícil de um app governamental entregar.

O sistema é dividido em três grandes blocos:

- **Mobile App (React Native)** — interface do usuário, captura de QR Code, busca e comparação de preços.
- **Backend serverless (Firebase)** — autenticação, banco de dados, processamento assíncrono via Cloud Functions.
- **Fonte de dados externa (Portal SEFAZ-BA)** — origem oficial dos dados fiscais que alimentam o catálogo de preços.

## 2. Stack Tecnológica

| Camada | Tecnologia | Motivo |
|---|---|---|
| App mobile | React Native | Familiaridade da equipe com JavaScript |
| Autenticação | Firebase Auth | Integração nativa, baixo overhead de setup |
| Banco de dados | Firestore | NoSQL, tempo real, escala bem para leitura de catálogo |
| Processamento backend | Cloud Functions (Node.js) | Processa QR Code/NFC-e de forma assíncrona e desacoplada do app |
| Notificações (Fase 2) | Cloud Messaging | Reservado para engajamento social/comunitário |
| Fonte de dados | Portal SEFAZ-BA | Fonte oficial e gratuita de dados fiscais |

## 3. Fluxo de Dados Principal (Fase 1)

```
[Usuário escaneia QR Code da nota fiscal]
              │
              ▼
   [App RN decodifica a URL da NFC-e]
              │
              ▼
[Cloud Function: parseReceipt] ──► [Consulta ao portal SEFAZ-BA]
              │
              ▼
[Cloud Function: validateReceipt] ─► valida estrutura e autenticidade
              │
              ▼
[Normalização de produtos e preços] ─► matching com catálogo existente
              │
              ▼
        [Gravação no Firestore]
     (produtos, preços, estabelecimentos)
              │
              ▼
[App consulta Firestore em tempo real]
   → exibe comparação de preços ao usuário
```

**Por que processar no backend e não no app:** o parsing da NFC-e envolve scraping/consulta ao portal da SEFAZ, validação de dados e possível normalização de nomes de produtos (ex.: "ARROZ TP1 5KG" e "Arroz Tipo 1 5kg" precisam ser tratados como o mesmo item). Isso é responsabilidade de backend para manter o app leve, consistente e para permitir reprocessamento sem exigir atualização do app.

## 4. Modelo de Dados (alto nível — Firestore)

| Coleção | Descrição | Campos-chave |
|---|---|---|
| `users` | Dados do usuário autenticado | uid, nome, cidade, criadoEm |
| `receipts` | Notas fiscais processadas | id, userId, storeId, dataCompra, status |
| `products` | Catálogo normalizado de produtos | id, nomeNormalizado, categoria, unidade |
| `prices` | Histórico de preços por produto/loja | productId, storeId, valor, dataColeta |
| `stores` | Estabelecimentos comerciais | id, nome, endereço, geolocalização |

> Este modelo é uma base inicial — deve ser refinado durante o Sprint 0/1 conforme a modelagem real do NFC-e for validada.

## 5. Arquitetura do App Mobile

- **Navegação:** estrutura em stacks/tabs, com fluxos isolados por domínio (Auth, Scanner, ProductSearch, PriceComparison, Profile).
- **Camada de serviços:** toda comunicação com Firebase e com a lógica de SEFAZ fica isolada em `services/`, nunca chamada diretamente das telas — isso facilita testes e eventual troca de provedor.
- **Estado:** hooks customizados + camada de store para dados compartilhados entre telas (ex.: usuário logado, carrinho de comparação).
- **Design system:** cores, tipografia e espaçamentos centralizados em `theme/`, para manter consistência visual — peça central da estratégia de diferenciação por UX/UI.

## 6. Segurança e Regras de Acesso

- Autenticação obrigatória via Firebase Auth para envio de notas fiscais (evita spam/dados falsos no catálogo).
- Regras do Firestore restringindo escrita direta pelo cliente — toda gravação de preços/produtos passa obrigatoriamente pelas Cloud Functions, nunca diretamente do app.
- Leitura pública (ou por usuários autenticados) do catálogo de preços, já que esse é o valor central do produto.

## 7. Escalabilidade por Fases

| Fase | Foco | Impacto arquitetural |
|---|---|---|
| **Fase 1** | UX/UI superior, ingestão via QR Code | Base descrita acima |
| **Fase 2** | Camada social/comunitária | Novas coleções (posts, avaliações, seguidores), uso de Cloud Messaging, possível necessidade de moderação de conteúdo |
| **Fase 3** | Expansão multi-estado | Dados passam a ter dimensão geográfica mais forte (estado/região), possível particionamento de dados por UF, integração com múltiplos portais de SEFAZ estaduais |

## 8. Riscos Técnicos Conhecidos

- **Dependência do portal SEFAZ-BA:** mudanças na estrutura do portal podem quebrar o parsing — a lógica deve ficar isolada (`services/sefaz/` e `functions/src/receipts/`) para facilitar manutenção.
- **Normalização de produtos:** nomes de produtos em notas fiscais são inconsistentes; a qualidade da comparação de preços depende diretamente da qualidade desse matching.
- **Multi-estado (Fase 3):** cada estado pode ter portal e formato de NFC-e diferentes, exigindo abstração desde já na camada de ingestão.

---

*Este documento reflete o estado de planejamento atual (Fase 1) e deve ser atualizado conforme decisões técnicas forem validadas durante os sprints.*