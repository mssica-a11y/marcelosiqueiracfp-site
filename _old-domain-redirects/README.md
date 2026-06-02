# 301 — Migração vectis.com.br → marcelosiqueiracfp.com.br

## O que este redirect faz
Move permanentemente (301) **todas** as URLs do domínio antigo para o novo, preservando o caminho. Isso transfere para o novo domínio a autoridade/links que o antigo já acumulou — essencial para não perder o pouco de SEO existente e para a IA reconhecer o novo domínio como a fonte.

Estratégia (a ordem das regras importa — Netlify usa a **primeira** que casar):
1. **Overrides de slug que mudou** — `/publicacoes`, `/blog`, `/publications` → `/insights` (a nova seção editorial).
2. **Páginas institucionais** — mapeadas explicitamente (documenta a intenção).
3. **Catch-all path-preserving** — `/* → marcelosiqueiracfp.com.br/:splat` cobre raiz, landings ITCMD, lead-magnet e qualquer URL com slug idêntico.

O `!` em `301!` força o redirect mesmo se existir arquivo naquele caminho — correto numa migração, em que o site antigo não deve mais servir nada.

## Onde publicar  ⚠️ ponto crítico
O `_redirects` precisa estar na **raiz do deploy do site ANTIGO (vectis.com.br)** — é lá que o redirect roda. Publicar no site novo não tem efeito sobre o tráfego que chega no domínio antigo.

- **Se vectis.com.br está no Netlify:** copie o arquivo `_redirects` para a raiz do publish desse site e faça redeploy. Pronto.
- **Se vectis.com.br está no Claude Design / outro host:** esse host pode não suportar `_redirects`. Duas opções:
  1. Apontar o DNS de vectis.com.br para um site Netlify mínimo (só com este `_redirects`) — solução limpa e definitiva.
  2. Usar o mecanismo de redirect do host atual (registrar.com, Cloudflare Rules, etc.) replicando as regras abaixo.

## Alternativa: netlify.toml
Se preferir configurar via `netlify.toml` em vez de `_redirects`, o equivalente é:

```toml
[[redirects]]
  from = "/publicacoes/*"
  to = "https://marcelosiqueiracfp.com.br/insights/:splat"
  status = 301
  force = true

[[redirects]]
  from = "/blog/*"
  to = "https://marcelosiqueiracfp.com.br/insights/:splat"
  status = 301
  force = true

[[redirects]]
  from = "/*"
  to = "https://marcelosiqueiracfp.com.br/:splat"
  status = 301
  force = true
```
Use **um** dos dois (`_redirects` OU `netlify.toml`), não os dois.

## Depois de publicar — verificar
```bash
curl -sI https://vectis.com.br/                | grep -i "location\|HTTP/"
curl -sI https://vectis.com.br/blog/qualquer   | grep -i "location\|HTTP/"
```
Esperado: `HTTP/.. 301` + `location: https://marcelosiqueiracfp.com.br/...`

## Escopo / ressalva
Este arquivo cobre a migração de **vectis.com.br**. Se alguma landing (ITCMD, lead-magnet) estiver publicada em **outro domínio**, esse domínio precisa do seu próprio `_redirects` apontando para o slug correspondente em marcelosiqueiracfp.com.br.
