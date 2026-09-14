# Divino Frango — Pedidos

Sistema **independente** do painel de vendas/compras (aquele projeto continua existindo separado, sem nenhuma ligação com este). Este projeto tem só uma função: **clientes   fazem pedidos pela internet, e você aceita ou recusa em tempo real.**

Dois sites em um projeto:

- **`/` (raiz)** — site público do cliente: cardápio, carri nho, formulário de pedido. É esse link que você compartilha.
- **`/admin`** — painel privado onde você vê os pedidos chegando (com som de aviso) e aceita, recusa, conclui ou imprime.

---

## 1. Criar o projeto no Supabase (novo, separado do outro)

1. Em [supabase.com](https://supabase.com), clique em **New project**.
2. Dê um nome diferente do outro, tipo `divino-frango-pedidos`, escolha uma senha e a região **South America (São Paulo)**.
3. Espere o projeto ficar pronto (1–2 min).
4. Vá em **SQL Editor → New query**, cole todo o conteúdo de [`supabase/schema.sql`](./supabase/schema.sql) e clique em **Run**. Isso cria as tabelas `cardapio` e `pedidos`, ativa o Realtime (para o painel avisar na hora) e cria o espaço de armazenamento das fotos.
5. Vá em **Project Settings → API** e copie a **Project URL** e a **anon public key** — vai precisar delas no passo 3.

---

## 2. Testar localmente (opcional)

```bash
npm install
cp .env.example .env
# edite o .env com a URL/chave do NOVO projeto Supabase
npm run dev
```

- Site do cliente: `http://localhost:5173/`
- Painel do dono: `http://localhost:5173/admin`

---

## 3. Publicar na Vercel

1. Suba esta pasta para um repositório novo no GitHub (ex: `divino-frango-pedidos`).
2. Na Vercel → **Add New… → Project** → selecione o repositório.
3. Zero-config (Vite é detectado automaticamente).
4. Em **Environment Variables**, adicione:
   | Nome | Valor |
   |---|---|
   | `VITE_SUPABASE_URL` | URL do **novo** projeto Supabase |
   | `VITE_SUPABASE_ANON_KEY` | anon key do **novo** projeto Supabase |
5. **Deploy**.
6. Depois de publicado: `https://SEU-SITE.vercel.app/` é o site do cliente, e `https://SEU-SITE.vercel.app/admin` é o seu painel.

## Publicar na Netlify (alternativa)

Mesmo processo do outro projeto: importar o repositório, adicionar as duas variáveis de ambiente em **Site settings → Environment variables**, e publicar. O `netlify.toml` já cuida do resto.

---

## Novidades: segurança com login, relatórios, aviso persistente, estoque e cupons

### 1. Rode isso no SQL Editor do Supabase (tudo de uma vez)

```sql
-- Caixa/PDV (se ainda não tiver rodado antes)
create table if not exists caixas (
  id uuid primary key default gen_random_uuid(),
  aberto_em timestamptz not null default now(),
  fechado_em timestamptz,
  valor_abertura numeric(12,2) not null default 0,
  valor_fechamento_informado numeric(12,2),
  status text not null default 'aberto',
  observacao text,
  created_at timestamptz not null default now()
);
create index if not exists caixas_status_idx on caixas (status);

create table if not exists movimentacoes_caixa (
  id uuid primary key default gen_random_uuid(),
  caixa_id uuid not null references caixas(id) on delete cascade,
  tipo text not null,
  valor numeric(12,2) not null,
  forma_pagamento text,
  descricao text,
  created_at timestamptz not null default now()
);
create index if not exists movimentacoes_caixa_caixa_idx on movimentacoes_caixa (caixa_id);

-- Estoque no cardápio
alter table cardapio add column if not exists estoque integer;

-- Cupom + desconto no pedido
alter table pedidos add column if not exists cupom_codigo text;
alter table pedidos add column if not exists desconto numeric(12,2) not null default 0;

-- Cupons de desconto
create table if not exists cupons (
  id uuid primary key default gen_random_uuid(),
  codigo text not null unique,
  tipo text not null,
  valor numeric(12,2) not null,
  ativo boolean not null default true,
  validade date,
  usos_maximos integer,
  usos_atual integer not null default 0,
  created_at timestamptz not null default now()
);
alter table cupons enable row level security;
create policy "cupons leitura publica" on cupons for select using (true);
create policy "cupons escrita autenticada" on cupons for insert with check (auth.role() = 'authenticated');
create policy "cupons update autenticada" on cupons for update using (auth.role() = 'authenticated');
create policy "cupons delete autenticada" on cupons for delete using (auth.role() = 'authenticated');

-- Funções seguras chamadas pelo site público (baixar estoque e contar uso de cupom)
create or replace function decrementar_estoque(p_item_id uuid, p_quantidade int)
returns void language plpgsql security definer set search_path = public as $$
begin
  update cardapio set estoque = greatest(estoque - p_quantidade, 0) where id = p_item_id and estoque is not null;
end; $$;
grant execute on function decrementar_estoque(uuid, int) to anon, authenticated;

create or replace function usar_cupom(p_codigo text)
returns void language plpgsql security definer set search_path = public as $$
begin
  update cupons set usos_atual = usos_atual + 1 where lower(codigo) = lower(p_codigo);
end; $$;
grant execute on function usar_cupom(text) to anon, authenticated;

-- Segurança: troca as políticas "acesso total" por políticas que exigem login
-- para qualquer ação de escrita (menos criar pedido e ler cardápio, que continuam públicas)
drop policy if exists "acesso total cardapio" on cardapio;
create policy "cardapio leitura publica" on cardapio for select using (true);
create policy "cardapio escrita autenticada" on cardapio for insert with check (auth.role() = 'authenticated');
create policy "cardapio update autenticada" on cardapio for update using (auth.role() = 'authenticated');
create policy "cardapio delete autenticada" on cardapio for delete using (auth.role() = 'authenticated');

drop policy if exists "acesso total pedidos" on pedidos;
create policy "pedidos leitura publica" on pedidos for select using (true);
create policy "pedidos insercao publica" on pedidos for insert with check (true);
create policy "pedidos update autenticada" on pedidos for update using (auth.role() = 'authenticated');

drop policy if exists "acesso total configuracoes" on configuracoes;
create policy "configuracoes leitura publica" on configuracoes for select using (true);
create policy "configuracoes update autenticada" on configuracoes for update using (auth.role() = 'authenticated');

drop policy if exists "acesso total caixas" on caixas;
alter table caixas enable row level security;
create policy "caixas autenticada" on caixas for all using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');

drop policy if exists "acesso total movimentacoes_caixa" on movimentacoes_caixa;
alter table movimentacoes_caixa enable row level security;
create policy "movimentacoes_caixa autenticada" on movimentacoes_caixa for all using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');

drop policy if exists "cardapio foto upload" on storage.objects;
drop policy if exists "cardapio foto update" on storage.objects;
drop policy if exists "cardapio foto delete" on storage.objects;
create policy "cardapio foto upload autenticada" on storage.objects for insert with check (bucket_id = 'cardapio' and auth.role() = 'authenticated');
create policy "cardapio foto update autenticada" on storage.objects for update using (bucket_id = 'cardapio' and auth.role() = 'authenticated');
create policy "cardapio foto delete autenticada" on storage.objects for delete using (bucket_id = 'cardapio' and auth.role() = 'authenticated');
```

### 2. Criar o login do admin (você)

1. No painel do Supabase, vá em **Authentication → Users → Add user**.
2. Preencha seu e-mail e uma senha forte. Marque **"Auto Confirm User"** (assim não precisa confirmar por e-mail).
3. Clique em **Create user**.
4. Ainda em Authentication, vá em **Providers → Email** e **desative "Allow new users to sign up"** — isso impede que qualquer pessoa crie uma conta nova sozinha; só o usuário que você acabou de criar (e outros que você criar manualmente) conseguem entrar.

Pronto — agora `/admin` pede e-mail e senha antes de mostrar qualquer coisa.

### O que mudou

- **Segurança**: `/admin` agora exige login. Sem a senha, ninguém consegue aceitar pedidos, mexer no caixa ou editar o cardápio — o cliente continua conseguindo ver o cardápio e fazer pedidos normalmente, sem precisar de login.
- **Relatórios**: nova aba com faturamento de hoje (delivery + balcão separados), total dos últimos 30 dias, gráfico dos últimos 14 dias, e produtos mais vendidos.
- **Aviso persistente**: enquanto houver pedido pendente sem resposta, o som repete a cada 20 segundos e o título da aba do navegador pisca ("🔴 1 pedido aguardando!") — difícil de não perceber.
- **Estoque**: ao cadastrar um item no cardápio, o campo "Estoque" é opcional (vazio = ilimitado). Quando chega a zero, o item aparece como "Esgotado" pro cliente automaticamente.
- **Cupons de desconto**: crie cupons (percentual ou valor fixo, com validade e limite de usos opcionais) na tela de Configurações. O cliente aplica o código no checkout e o desconto entra no cálculo do total.

## Domínio próprio

Em vez de `SEU-SITE.vercel.app`, você pode usar um domínio seu (ex: `pedidos.divinofrango.com.br` ou `divinofrangopedidos.com`).

1. **Compre um domínio** — em [registro.br](https://registro.br) (para `.com.br`, mais barato e é o registro oficial no Brasil) ou em [Namecheap](https://namecheap.com)/GoDaddy (para `.com`).
2. Na Vercel, entra no projeto → **Settings → Domains**.
3. Digita o domínio que você comprou e clica em **Add**.
4. A Vercel mostra um ou dois registros de DNS pra você configurar (geralmente um **CNAME** apontando para `cname.vercel-dns.com`, ou um **A record** com um IP). 
5. Vai até o painel do lugar onde você comprou o domínio (registro.br, Namecheap, etc.) → área de **DNS** → adiciona esses registros exatamente como a Vercel mostrou.
6. Espera propagar (de alguns minutos até algumas horas). A Vercel confirma automaticamente quando estiver certo.

Se quiser fazer essa parte, me chama que eu te guio passo a passo igual fizemos com o Supabase e a Vercel.

## Estrutura

```
├── index.html              # site do cliente (raiz)
├── admin.html               # painel do dono (/admin)
├── src/
│   ├── main.jsx              # entrada do site do cliente
│   ├── admin-main.jsx         # entrada do painel do dono
│   ├── PedidoApp.jsx           # lógica/telas do site do cliente
│   ├── AdminApp.jsx             # lógica/telas do painel (pedidos + cardápio)
│   ├── supabaseClient.js         # conexão com o Supabase deste projeto
│   └── logo.js                    # logo do Divino Frango em base64
├── supabase/schema.sql       # script único para criar tudo no Supabase
├── vercel.json / netlify.toml  # URLs amigáveis (/admin)
├── package.json
└── vite.config.js
```

## O que já foi verificado

Sintaxe de todo o JSX/JS validada (parsing completo, sem erros).

## O que só dá pra confirmar depois do deploy

Este ambiente não tem acesso à internet, então não foi possível rodar `npm install`/`npm run build` de verdade aqui. As dependências são padrão e estáveis — o teste real acontece no primeiro `npm install` + `npm run build`, seja no seu computador ou automaticamente na Vercel/Netlify. Qualquer erro nesse momento, me manda a mensagem que eu ajusto.
