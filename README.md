# marcelosiqueiracfp.com.br — site

Site estático (HTML + CSS, sem build) de **Marcelo Siqueira CFP**. Deploy no Netlify apontando para `marcelosiqueiracfp.com.br`.

## Estrutura
```
index.html                Hub: problema comum + duas portas
executivo.html            Porta do executivo com equity (/executivo)
founder.html              Porta do founder e da família empresária (/founder)
na-midia.html             Entrevistas Times Brasil | CNBC + VideoObject (/na-midia)
insights.html             Hub /insights (servido por rewrite 200 no netlify.toml; filtro de trilhas sem JS)
teste-90-dias.html        Lead magnet; ?perfil=executivo|founder pula a pergunta de bifurcação
insights/
  <slug>.html             Artigos (cada um termina com a .porta para /executivo ou /founder)
  article-template.html   Template de artigo (noindex; JSON-LD Person→Article→FAQPage)
  styles.css              Folha do design system
  og/                     Cartões Open Graph 1200x630
llms.txt                  Resumo para LLMs (text/plain UTF-8 via netlify.toml)
sitemap.xml               Submeter no Google Search Console
robots.txt                Libera crawlers de IA (GPTBot, ClaudeBot, PerplexityBot, Applebot-Extended…)
favicon.* / icon-*.png    Favicon + site.webmanifest
_old-domain-redirects/    301 vectis.com.br → este domínio. NÃO é servido aqui;
                          deve ser deployado na RAIZ do site ANTIGO. Ver README de lá.
```

## Deploy (Netlify)
- Publish directory: raiz do repo.
- Domínio: `marcelosiqueiracfp.com.br` (apex) + redirect www→apex.

## Decisões técnicas
- **HTML estático, não SPA/React** — crawlers de IA não rodam JS; conteúdo + JSON-LD precisam estar no payload inicial.
- **Entidade `Person` + `ProfessionalService`** idênticas em todas as páginas (só CFP®; sameAs → LinkedIn, YouTube, Instagram), `author` de cada artigo por `@id`. Gate regulatório: nada de "consultor CVM", "consultoria de investimentos" ou frase de rebate até a migração sair.
- FAQ visível bate 1:1 com o `FAQPage` JSON-LD (senão o Google desconta).

> `article-template.html` é um **template** (corpo é esqueleto). Ao criar um artigo real, copie para `insights/<slug>.html`, preencha o corpo e adicione a URL no `sitemap.xml`.
