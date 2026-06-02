# marcelosiqueiracfp.com.br — site

Site estático (HTML + CSS, sem build) de **Marcelo Siqueira CFP**. Deploy no Netlify apontando para `marcelosiqueiracfp.com.br`.

## Estrutura
```
insights/
  index.html              Hub /insights (listagem dos artigos)
  article-template.html   Template de artigo (copiar por artigo; JSON-LD Person→Article→FAQPage)
  styles.css              Folha do design system
sitemap.xml               Submeter no Google Search Console
robots.txt                Libera crawlers de IA (GPTBot, ClaudeBot, PerplexityBot…)
_old-domain-redirects/    301 vectis.com.br → este domínio. NÃO é servido aqui;
                          deve ser deployado na RAIZ do site ANTIGO. Ver README de lá.
```

## Deploy (Netlify)
- Publish directory: raiz do repo.
- Domínio: `marcelosiqueiracfp.com.br` (apex) + redirect www→apex.

## Decisões técnicas
- **HTML estático, não SPA/React** — crawlers de IA não rodam JS; conteúdo + JSON-LD precisam estar no payload inicial.
- **Entidade `Person`** (Marcelo Siqueira, CFP, sameAs→LinkedIn) wired como `author` de cada artigo é a alavanca para a IA citar o nome.
- FAQ visível bate 1:1 com o `FAQPage` JSON-LD (senão o Google desconta).

> `article-template.html` é um **template** (corpo é esqueleto). Ao criar um artigo real, copie para `insights/<slug>.html`, preencha o corpo e adicione a URL no `sitemap.xml`.
