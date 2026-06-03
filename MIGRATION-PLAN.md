# Plano de migração — esqueleto estático → Next.js 16 (stack Theanna)

> Gerado 2026-06-03. Alvo: portar o site `marcelosiqueiracfp-site` (HTML estático, 1 commit)
> para Next.js 16 / React 19 / Tailwind 4 / Vercel, com posts como **TypeScript
> ContentBlock arrays** e schemas SEO/GEO/AEO derivados do conteúdo.

---

## 0 · Decisão antes de escrever código (trade-off de tempo-em-índice)

A migração **não precisa** preceder o deploy. Se os slugs ficarem idênticos
(`/insights/<slug>`), trocar estático→Next.js depois é invisível para o Google —
mesma URL, re-crawl pega o HTML novo, sitemap e redirects seguem válidos.

| Caminho | Ganha | Perde |
|---|---|---|
| **Next.js-first** (este plano) | Fundação certa, paridade c/ Theanna, schema SSOT desde o artigo 1 | ~1–2 semanas de build antes de QUALQUER página indexar |
| **Estático agora → Next.js depois** | Artigo 1 indexando já; tempo-em-índice começa a contar | Reescrever o artigo 1 como ContentBlock na migração (custo baixo: 1 artigo) |

O recurso escasso numa jogada de AEO é **tempo-em-índice**, não código. Decisão é
do Marcelo — este plano assume **Next.js-first**, mas o caminho misto é defensável.

**Gate operacional (verificado 2026-06-03):** disco 63 GB livres ✅ · Node v24.14.0 ✅ · npm 11.9.0 ✅.
(O `ENOSPC` visto antes era a partição tmp do harness, não o disco.) Manter `TMPDIR`
apontado para o disco grande em qualquer `npm install`.

---

## 1 · Princípio espinha-dorsal: schema DERIVADO do conteúdo (fonte única)

O motivo de ContentBlock arrays baterem o template estático: **todo schema
(Article/FAQ/DefinedTermSet/HowTo) é derivado dos blocos tipados**, não escrito à mão.
Isso mata a regra frágil do arquivo atual ("manter o FAQ visível 1:1 com o FAQPage
JSON-LD na unha"). O bloco FAQ é renderizado **e** vira `FAQPage.mainEntity` da mesma
fonte → 1:1 por construção.

Se o plano autorar schema separado do conteúdo, recriamos a armadilha de dupla
manutenção com mais passos. **Esta é a decisão de design que carrega o resto.**

### Honestidade sobre os schemas (não vender o que não entrega)
- **ArticleSchema** — base de autoria/publicação. Vale para Google e LLM. ✅
- **FAQSchema / HowToSchema** — desde 2023 o Google **não gera mais rich snippet**
  desses para sites comuns. Continuam valiosos como **sinal legível por máquina para
  LLM/AEO** (o nosso objetivo real). Emitir sim — mas frame = sinal para IA, não rich result.
- **DefinedTermSetSchema** — marca glossário/termos; ótimo para LLM ancorar definições.
- **BreadcrumbSchema** — hierarquia; ainda exibido pelo Google.
- **Speakable** — limitado a notícias/inglês hoje; baixo retorno, custo zero — opcional.
- **Entity markup** — `Person`/`Organization` no `@graph` + `mentions`/`about` no Article
  (a alavanca para a IA citar o NOME). Prioridade máxima.

---

## 2 · Estrutura de pastas alvo (App Router)

```
marcelosiqueiracfp-site/
  app/
    layout.tsx              # <html lang="pt-BR">, fontes, <Nav/> <Footer/>, GA + Clarity
    page.tsx                # home (placeholder por ora)
    globals.css             # @import "tailwindcss"; @theme { ...tokens... }
    sitemap.ts              # GERADO do registry de posts (SSOT)
    robots.ts               # GERADO (libera GPTBot/ClaudeBot/PerplexityBot…)
    insights/
      page.tsx              # hub /insights — lista posts do registry
      [slug]/page.tsx       # artigo: generateStaticParams + generateMetadata + blocos + JSON-LD
  content/posts/
    _types.ts               # ContentBlock (union discriminada) + Post
    _registry.ts            # array de todos os posts — SSOT p/ sitemap, hub e rotas
    rsu-saida-empresa-imposto-de-renda.ts   # Post: metadados + blocks[]
  components/
    blocks/BlockRenderer.tsx # switch sobre block.type → Heading/Paragraph/Callout/Quote/FAQ/DefinedTerm/HowTo/CTA
    schema/JsonLd.tsx        # <script type="application/ld+json">
    schema/buildGraph.ts     # deriva @graph do Post (Person→Org→Article→FAQ→DefinedTermSet→HowTo→Breadcrumb)
    Nav.tsx  Footer.tsx
  lib/site.ts               # constantes: baseUrl, entidade Person, Organization
  public/insights/og/        # imagens OG
  next.config.ts            # redirects (www→apex), headers
  package.json  tsconfig.json
```

`_old-domain-redirects/` **permanece intocado** — pertence ao host do domínio ANTIGO
(vectis.com.br), não a este repo. A migração não o afeta.

---

## 3 · Modelo de dados (o coração)

```ts
// content/posts/_types.ts
export type ContentBlock =
  | { type: 'heading';     level: 2 | 3; text: string; id?: string }
  | { type: 'paragraph';   html: string }
  | { type: 'quote';       text: string }
  | { type: 'callout';     label: string; html: string }
  | { type: 'faq';         items: { q: string; a: string }[] }
  | { type: 'definedTerm'; term: string; definition: string }
  | { type: 'howTo';       name: string; steps: { name: string; text: string }[] }
  | { type: 'cta';         heading: string; body: string; href: string; label: string };

export type Post = {
  slug: string;
  metaTitle: string;        // <title> — intenção de busca, não brand
  metaDescription: string;
  title: string;            // <h1>
  eyebrow: string;          // "Tributação · Equity executivo"
  dek: string;
  datePublished: string;    // ISO
  dateModified: string;
  readingMinutes: number;
  ogImage: string;          // URL absoluta
  blocks: ContentBlock[];
};
```

`buildGraph(post)` percorre `blocks` UMA vez e monta o `@graph`:
- sempre: `Person` + `Organization` + `Article` (de `lib/site.ts` + metadados do post)
- blocos `faq`  → `FAQPage.mainEntity`
- blocos `definedTerm` → `DefinedTermSet`
- blocos `howTo` → `HowTo`
- `BreadcrumbList` (Início → Insights → título)

O componente FAQ visível renderiza dos **mesmos** blocos `faq` → consistência garantida.

---

## 4 · Mapeamento de migração (arquivo atual → novo)

| Estático hoje | Vira |
|---|---|
| `<title>`, meta description | `generateMetadata()` no `[slug]/page.tsx` |
| OG / Twitter tags | `metadata.openGraph` / `metadata.twitter` |
| `<link rel=canonical>` | `metadata.alternates.canonical` |
| JSON-LD `@graph` (Person→Org→Article→FAQPage) | `buildGraph(post)` + `<JsonLd>` (server component) |
| `<nav class=site-nav>` | `components/Nav.tsx` no `layout.tsx` |
| `<footer class=site-foot>` | `components/Footer.tsx` no `layout.tsx` |
| `styles.css` (`--vectis-*`) | `@theme` em `globals.css` (Tailwind 4 — ver §5) |
| corpo do artigo (esqueleto HTML) | `blocks[]` no `.ts` do post (ESCREVER o conteúdo real aqui) |
| FAQ visível `<details>` | bloco `faq` → componente (mesma fonte do schema) |
| `sitemap.xml` | `app/sitemap.ts` (gera do `_registry`) |
| `robots.txt` | `app/robots.ts` |
| `_old-domain-redirects/` | inalterado (host do domínio antigo) |

---

## 5 · Armadilhas técnicas (confirmadas)

- **Tailwind 4 é CSS-first.** Portar os tokens `--vectis-*` para um bloco `@theme` em
  `globals.css` com `@import "tailwindcss"` — NÃO um `tailwind.config.js` v3 com
  `theme.extend.colors`. Ex.:
  ```css
  @import "tailwindcss";
  @theme {
    --color-navy: #0A1E3D;  --color-brass: #C4A44A;  --color-graphite: #3D4F5F;
    --color-warm-white: #F7F6F3;  --color-linen: #EDECE8;  --color-stone: #E2E0DA;
    --font-display: "Instrument Serif", Georgia, serif;
    --font-sans: "DM Sans", system-ui, sans-serif;
    --font-mono: "DM Mono", monospace;
  }
  ```
- **Não inventar APIs específicas do Next 16 sem verificar.** Usar padrões estáveis
  desde App Router 13/14: `generateStaticParams`, `generateMetadata`,
  `app/sitemap.ts` + `app/robots.ts`, JSON-LD via `<script type="application/ld+json">`
  num **server component** (entra no payload inicial — requisito AEO).
- **ContentBlock vs MDX:** blocos dão schema-derivation (certo p/ AEO) mas são verbosos
  de escrever. MDX é mais rápido de autorar. Estamos espelhando a Theanna → blocos são
  defensáveis, mas **vigiar**: escrever 6 artigos como blocos não pode virar o gargalo.

---

## 6 · Fases de execução

- **Fase 0 — decisão + gate** ✅ (este doc; disco/node verificados)
- **Fase 1 — scaffold:** `create-next-app` (TS, App Router, Tailwind 4), `@theme` com
  tokens, `Nav`/`Footer`/`layout`, home placeholder. `next build` local verde.
- **Fase 2 — motor:** `_types.ts`, `BlockRenderer`, `buildGraph` (schema-derivation), `JsonLd`.
- **Fase 3 — artigo 1:** portar para `rsu-saida-empresa-imposto-de-renda.ts` **e escrever
  o corpo real** (hoje é esqueleto); wirear `/insights/[slug]` + hub `/insights`.
- **Fase 4 — descoberta:** `app/sitemap.ts` + `app/robots.ts` (do registry);
  `next.config.ts` (www→apex, headers de cache).
- **Fase 5 — telemetria:** Google Analytics + Microsoft Clarity no `layout`; imagem OG.
- **Fase 6 — deploy:** Vercel conectado ao mesmo repo GitHub (preset Next.js auto, build
  ~60s); domínio apex + www→apex; verificar domínio no GSC + submeter `/sitemap.xml`.

---

## 7 · Verificação por fase (como sei que funcionou)

- `next build` sem erro + `next start`, então `curl -s localhost:3000/insights/<slug> | grep ld+json`
  → o `@graph` está no **payload inicial** (não injetado por JS).
- Rich Results Test / Schema validator no HTML renderizado (não no JS).
- `view-source` da página: conteúdo do artigo presente sem rodar JS (prova de SSG/SSR).
- Lighthouse SEO ≥ 95.
- Pós-deploy: GSC indexando, `curl -sI` nos redirects do domínio antigo retornando 301.

---

## O que NÃO muda
- Slugs (`/insights/<slug>`) — idênticos, para a migração ser invisível ao Google.
- A entidade `Person` como `author` — alavanca central de citação por IA.
- `_old-domain-redirects/` — pertence ao host do domínio antigo.
- Deploy a cada `git push` — só troca Netlify→Vercel.
