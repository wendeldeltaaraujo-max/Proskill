# ProSkill — Plataforma (site público + área do aluno + ADM MASTER)

Arquitetura: **Cloudflare Workers** (API + site estático) · **Neon PostgreSQL** (dados) · **Cloudflare R2** (arquivos dos cursos e capas) · **Stripe Checkout** (pagamentos).

```
public/            site (HTML/CSS/JS), /admin/ (painel)
src/worker.js      entrada: roteia site, API e a ORIGEM ISOLADA de conteúdo
src/routes/        api.js (auth, catálogo, aluno, admin, Stripe) · content.js (serve os cursos HTML)
src/lib/           zip.js (validação), crypto, tokens, stripe, http, auth, courses, mime
src/repo/          neon.js (SQL) · memory.js (somente testes/dev)
migrations/        0001_init.sql
test/              testes automatizados (node --test) e test/e2e (navegador)
```

## O que VOCÊ precisa configurar (nada disso foi feito/ testado com contas reais)
1. **Neon**: crie um projeto, copie a connection string → segredo `DATABASE_URL`. Rode `DATABASE_URL=... npm run migrate`.
2. **R2**: `npx wrangler r2 bucket create proskill-courses` (bucket **privado**, sem domínio público).
3. **Dois domínios** apontando para o MESMO Worker: o do site (`PUBLIC_ORIGIN`) e um **domínio separado** só para os cursos (`CONTENT_ORIGIN`, ex.: `cursos-proskill.com.br` — de preferência outro domínio registrável, não subdomínio com cookies compartilhados). Ajuste `wrangler.toml`.
4. **Segredos** (`npx wrangler secret put NOME`): `DATABASE_URL`, `CONTENT_TOKEN_SECRET` (aleatório, `openssl rand -base64 48`), `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`.
5. **Stripe**: no painel, crie o endpoint de webhook `https://SEU-SITE/api/stripe/webhook` com os eventos `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`; copie o *signing secret* para `STRIPE_WEBHOOK_SECRET`. Teste primeiro em modo de teste (cartão 4242 4242 4242 4242).
6. `npm install` e `npm run deploy`.
7. **Primeiro administrador**: o e-mail definido em `ADMIN_EMAIL` (no `wrangler.toml`, já preenchido com wendel.delta.araujo@gmail.com) vira ADM MASTER automaticamente ao se cadastrar no site — só se ainda não existir nenhum admin. **Cadastre-se logo após publicar o site, antes de divulgá-lo**: como não há verificação de e-mail, quem se cadastrasse antes com esse e-mail ficaria com o acesso. Se já houver conta com esse e-mail, use `DATABASE_URL=... npm run make-admin -- email`. Acesse `/admin/`.
8. Opcional: `js/config.js` (e-mail de contato e redes sociais reais — hoje vazios de propósito).

Desenvolvimento local com os serviços reais: copie `.dev.vars.example` → `.dev.vars` e rode `npm run dev` (use `http://localhost:8787`, conteúdo em `http://content.localhost:8787`).
Sem nenhum serviço: `node scripts/dev-server.mjs` (banco em memória, R2 e Stripe falsos — **apenas para testes**).


## Rodar em QUALQUER lugar (servidor Node.js — recomendado)
O mesmo código roda como um servidor Node.js comum (`npm start`) ou em Docker. Serve o site, a API, o painel e o conteúdo dos cursos. Funciona em VPS, Render, Railway, Fly.io, Hetzner, DigitalOcean, no seu computador, etc.

**Variáveis de ambiente** (veja `.env.example`): `PUBLIC_ORIGIN`, `CONTENT_ORIGIN`, `ADMIN_EMAIL`, `DATABASE_URL` (Neon), `CONTENT_TOKEN_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `TRUST_PROXY=1` (atrás de proxy) e o armazenamento dos cursos:
- **Compatível com S3** (`S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`): Cloudflare R2, AWS S3, Backblaze B2, MinIO… Necessário em hospedagens cujo disco apaga a cada deploy (Render grátis, Railway, Fly sem volume).
- **Pasta local** (`STORAGE_DIR`): somente em VPS/servidor com disco persistente.

**Passos**
1. `npm install` e `npm start` (ou `docker build -f deploy/docker/Dockerfile -t proskill . && docker run -p 8787:8787 --env-file .env proskill`).
2. Aponte o domínio do site **e um segundo domínio** (para `CONTENT_ORIGIN`) para o mesmo servidor, com HTTPS.
3. O banco Neon já tem o esquema aplicado (ou rode `npm run migrate`).

**Não serve para isso:** a Vercel (funções serverless) — limite de ~4,5 MB por envio (os cursos têm até 50 MB) e sem disco. O site que está na Vercel hoje mostra só a parte visual.

## Como o ADM MASTER publica um curso HTML
`/admin/` → Gerenciar cursos → **Novo curso** → preencher dados → capa → **Conteúdo interativo do curso** (ZIP com `index.html` na raiz, ou HTML único; até 50 MB) → *Pré-visualizar* → **Salvar como rascunho** ou **Salvar e publicar**. Reenviar o ZIP cria uma **nova versão**, que só vai ao ar ao clicar em Publicar novamente. Despublicar/Arquivar tiram do catálogo e bloqueiam novos acessos.

**Progresso salvo (opcional, dentro do HTML do curso):**
```js
window.addEventListener('message', e => { if (e.data?.type === 'proskill:init') retomar(e.data.progress /* {percent, completed, state} | null */); });
parent.postMessage({ type: 'proskill:progress', percent: 40, completed: false, state: { aula: 3 } }, '*');
```

## Segurança implementada
- **Conteúdo de curso nunca roda na origem principal**: é servido por host separado, com `Content-Security-Policy: sandbox` (origem opaca) e `<iframe sandbox="allow-scripts …">` **sem** `allow-same-origin`. O curso não lê cookies/sessão/localStorage da ProSkill nem chama a API autenticada (testado no navegador).
- **ZIP**: rejeita `..`/caminhos absolutos/`\`/symlinks/criptografia/extensões fora da lista/duplicados/CRC inválido/mais de 1000 arquivos/>50 MB/>150 MB descompactado/razão de compressão suspeita; o limite é aplicado sobre os bytes **realmente descompactados** (tamanho declarado falso não passa). Tudo é validado antes de gravar no R2; só administradores enviam.
- **Autorização no servidor**: acesso ao conteúdo exige sessão + matrícula (checadas no backend); o bucket é privado; o navegador recebe um token HMAC com validade de 4 h (alterar `CONTENT_TTL_SEC` em `api.js`). *Limitação:* uma matrícula revogada só perde o acesso a arquivos já liberados ao expirar o token.
- **Pagamento**: preço sempre do banco; webhook com assinatura HMAC + tolerância de 5 min, idempotência (`webhook_events`), conferência de valor/moeda; a página de sucesso não libera nada.
- **Auth**: senhas PBKDF2-SHA256 (100.000 iterações, teto do Workers), sessões aleatórias com hash no banco, cookie `HttpOnly; Secure; SameSite=Lax`, proteção CSRF (Origin + cabeçalho), bloqueio após 5 falhas/15 min. CSP e cabeçalhos de segurança no site.

## Estado real (honesto)
| Item | Situação |
|---|---|
| Código de ZIP, senha, tokens, assinatura Stripe | **Testado** (18 testes unitários) |
| Servidor Node (site+API+upload em pasta local+conteúdo isolado), assinatura S3 (confere com o vetor oficial da AWS) | **Testado** (processo real e testes automatizados). S3 real (R2/AWS/B2): **não testado** com conta real |
| Esquema e consultas principais no Neon | **Testado** em banco Neon real (migration, busca sem acento, idempotência do webhook, e-mail único). O código rodando conectado ao Neon: **não testado** |
| Fluxos da API (admin → upload → publicar → catálogo → matrícula → webhook → acesso → progresso) | **Testado** (13 testes de integração, com banco em memória e R2 falso) |
| Fluxo completo no navegador (34 verificações: painel, upload, prévia sandbox, catálogo, compra simulada, área do aluno, progresso, celular) | **Testado** com `scripts/dev-server.mjs` |
| Cloudflare R2 real, `wrangler dev/deploy`, domínio isolado em produção | **NÃO testado** |
| Stripe real (checkout e webhook com a Stripe) | **NÃO testado** — só com assinatura gerada localmente e Stripe simulado |
| Certificados | **NÃO implementado** (tabela reservada; a área do aluno informa "em breve") |
| Módulos/aulas em tabelas, recuperação de senha, verificação de e-mail, reembolsos | **NÃO implementados** |
Rode `npm test` (e `python3 test/e2e/e2e.py` com o dev-server ligado) para repetir os testes.


## Vercel (somente a parte visual)
O `Dockerfile` fica em `deploy/docker/` de propósito: na raiz, a Vercel o detecta e tenta construir/enviar uma imagem de contêiner (erro `BUILD_FAILED` / `buildah`). `vercel.json` publica só a pasta `public/` como site estático (sem login, compra ou admin).
