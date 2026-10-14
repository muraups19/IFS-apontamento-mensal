# Painel de Horas (versão compartilhada com Supabase)

Site estático (um `index.html`) em que cada colega cria a própria conta e lança
suas horas. Os dados ficam no Supabase (plano gratuito) e cada pessoa enxerga
apenas o que é seu. Não há servidor para manter: basta hospedar os arquivos.

```
index.html                       o site
config.js                        URL e chave pública do seu Supabase
supabase/schema.sql              tabelas, regras de segurança e função keep-alive
supabase/seed_dados_planilha.sql importa os 3 períodos da planilha para a sua conta
.github/workflows/keep-alive.yml evita que o projeto gratuito seja pausado
```

## 1. Criar o projeto no Supabase
1. Em supabase.com, crie um projeto (plano Free).
2. Abra **SQL Editor**, cole o conteúdo de `supabase/schema.sql` e clique em **Run**.
   Pode rodar de novo sem problema.

## 2. Configurar o login
Em **Authentication**:
- **Sign In / Providers**: deixe *Email* ativo.
- **URL Configuration**: em *Site URL* coloque o endereço final do site (passo 4) e
  adicione o mesmo endereço em *Redirect URLs*. Isso vale para o link de confirmação
  de e-mail e para "Esqueci minha senha".
- **Confirmação de e-mail**: o envio de e-mails padrão do Supabase tem um limite
  baixo por hora. Para um grupo pequeno, o jeito mais simples é desligar
  *Confirm email*. Se preferir manter a confirmação, configure um SMTP próprio
  em *Authentication > SMTP Settings*.
- Para limitar o cadastro a um domínio (por exemplo `@suaempresa.com.br`), use o
  bloco "OPCIONAL 1" no final do `schema.sql`.

## 3. Conectar o site
Em **Project Settings > API**, copie a **Project URL** e a chave **anon** (ou
**publishable**) e cole em `config.js`.
A chave é pública por desenho: quem protege os dados são as regras de segurança
(RLS) do `schema.sql`. Nunca use a chave `service_role` no site.

## 4. Publicar o site
Qualquer hospedagem de arquivos estáticos serve (gratuitas: GitHub Pages,
Netlify, Cloudflare Pages, Vercel). Suba `index.html` e `config.js`.
Depois de publicado, volte ao passo 2 e informe o endereço no Supabase.

## 5. Manter o banco ativo (função keep-alive)
O plano gratuito pausa o projeto quando fica uma semana com pouca atividade.
Três proteções já estão prontas:
1. **GitHub Actions** (principal): `.github/workflows/keep-alive.yml` chama
   `public.keep_alive()` a cada 3 dias. Suba esta pasta para um repositório no
   GitHub e crie os secrets `SUPABASE_URL` e `SUPABASE_ANON_KEY`
   (Settings > Secrets and variables > Actions). Teste em
   Actions > *Supabase keep-alive* > Run workflow.
   Observação: o GitHub desativa fluxos agendados em repositórios sem atividade
   por 60 dias. Se isso acontecer, é só reativar em Actions.
2. **O próprio site**: cada abertura chama `keep_alive()` (no máximo 1 escrita por hora).
3. **Opcional**: agendamento interno com `pg_cron` (bloco "OPCIONAL 2" do `schema.sql`).

A função grava um único registro na tabela `keepalive` e não expõe nenhum dado.

## 6. Primeiro acesso e seus dados antigos
1. Abra o site, clique em **Criar conta** e informe nome, e-mail e senha.
2. O site pede o salário base na primeira entrada.
3. Para trazer os 3 períodos da planilha: abra `supabase/seed_dados_planilha.sql`,
   troque `SEU_EMAIL_AQUI` pelo e-mail cadastrado e rode no SQL Editor (uma única vez).

## Como os dados são protegidos
- Cada tabela tem RLS ligada: `user_id = auth.uid()`. Um colega nunca vê os dados de outro.
- Visitantes sem login não têm acesso a nenhuma tabela.
- O banco recusa períodos que se cruzam, horas fora de 0 a 24 e lançamentos fora do período.
- Excluir a conta no Supabase apaga todos os dados dela em cascata.

## Regras de cálculo (iguais às da planilha)
- Horas do período = dias úteis × 8 (sem sábados, domingos e feriados).
- Valor da hora = salário do período ÷ horas do período.
- Total = valor da hora × (horas normais + horas adicionais).
- Feriados nacionais (inclusive Carnaval, Sexta Santa e Corpus Christi) são calculados
  pelo site para qualquer ano. Feriados locais cada pessoa cadastra os seus.
