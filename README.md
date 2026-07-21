# pdfReader — sua biblioteca de PDFs

Frontend estático (um único `index.html`) que sincroniza seus PDFs entre PC e celular usando **Supabase Auth + Storage privado**. Sem backend próprio, sem banco de metadados: as pastas do Storage são as categorias.

Interface minimalista e moderna, com tema claro/escuro, busca por arquivo e leitor de PDF embutido (PDF.js) que reabre na última página lida.

O que fica no dispositivo (localStorage): tema, última pasta aberta e última página lida de cada PDF. Todo o resto vem do Supabase.

---

## 1. Criar o projeto no Supabase

1. Acesse [supabase.com](https://supabase.com) → **New project** (plano gratuito serve).
2. Em **Authentication → Users → Add user**, crie seu usuário com e-mail e senha (marque *Auto confirm*). Não há tela de cadastro no app — é uma biblioteca pessoal.
3. Em **Authentication → Sign In / Providers → Email**, deixe habilitado. Opcional: desative *Allow new users to sign up* para ninguém mais conseguir criar conta.

## 2. Criar o bucket privado

Em **Storage → New bucket**:

- Nome: `pdfs`
- **Public bucket: DESLIGADO** (privado)
- Opcional: limite de tamanho por arquivo (ex. 50 MB) e MIME `application/pdf`

## 3. Políticas de acesso (RLS)

Em **SQL Editor**, rode o script abaixo. Ele garante que cada usuário autenticado só enxerga e manipula arquivos dentro da pasta com o **próprio ID** (`{user_id}/categoria/arquivo.pdf` — é assim que o app grava):

```sql
-- Leitura
create policy "ler os proprios pdfs"
on storage.objects for select to authenticated
using ( bucket_id = 'pdfs' and (storage.foldername(name))[1] = auth.uid()::text );

-- Upload
create policy "enviar os proprios pdfs"
on storage.objects for insert to authenticated
with check ( bucket_id = 'pdfs' and (storage.foldername(name))[1] = auth.uid()::text );

-- Renomear / mover
create policy "mover os proprios pdfs"
on storage.objects for update to authenticated
using ( bucket_id = 'pdfs' and (storage.foldername(name))[1] = auth.uid()::text )
with check ( bucket_id = 'pdfs' and (storage.foldername(name))[1] = auth.uid()::text );

-- Excluir
create policy "excluir os proprios pdfs"
on storage.objects for delete to authenticated
using ( bucket_id = 'pdfs' and (storage.foldername(name))[1] = auth.uid()::text );
```

Sem sessão válida, nada é acessível — o bucket é privado e não há política para `anon`.

## 4. Configurar as credenciais no app

Em **Project Settings → API Keys**, copie a **Project URL** e a **publishable key** (`sb_publishable_...`). Abra o `index.html` e edite o bloco `CONFIG`:

```js
const SUPABASE_URL = 'https://xxxx.supabase.co';
const SUPABASE_PUBLISHABLE_KEY = 'sb_publishable_...';
```

> A *publishable key* é feita para ficar no frontend — ela não dá acesso a nada por si só. Quem protege os arquivos são as políticas RLS acima. Nunca use a `secret key` (`sb_secret_...`) nem a `service_role` no frontend.

## 5. Publicar no GitHub Pages

1. Envie o `index.html` (e este README) para a branch `main`.
2. No GitHub: **Settings → Pages → Source: Deploy from a branch → main / root → Save**.
3. Em 1–2 minutos o app estará em `https://SEU-USUARIO.github.io/SEU-REPO/`.
4. (Recomendado) No Supabase, em **Authentication → URL Configuration**, adicione essa URL em *Site URL*.

Abra a mesma URL no PC e no celular, entre com o mesmo e-mail/senha e a biblioteca fica sincronizada.

---

## Uso

| Ação | Como |
|---|---|
| Criar pasta (categoria) | **Nova pasta** na barra lateral |
| Criar subpasta | Botão **Subpasta** dentro de uma pasta (navegue pelo *breadcrumb*) |
| Enviar PDF | Botão **Enviar PDF** (arrastar e soltar funciona) |
| Ver em lista ou catálogo | Alternador no canto direito da busca |
| Selecionar vários | Botão **Selecionar** → mover/excluir em massa |
| Buscar | Campo de busca acima da lista |
| Ler | Toque no arquivo — leitor de **rolagem contínua** (como um PDF comum), reabre na última página lida |
| Notas | Painel lateral no leitor (ícone de balão) — notas, resumos e citações, salvos automaticamente |
| Renomear / Mover / Excluir | Ícones que aparecem ao passar o mouse na linha do arquivo |
| Histórico | Aba **Histórico** — registro de leitura e ações (salvo no dispositivo) |
| Insights | Aba **Insights** — progresso de leitura, PDFs por pasta, armazenamento |
| Tema claro/escuro | Ícone de sol no topo (salvo no dispositivo) |

> **Subpastas & RLS:** as subpastas são apenas prefixos mais profundos
> (`{user_id}/pasta/subpasta/arquivo.pdf`). As políticas do README continuam
> válidas, pois checam sempre o **primeiro** segmento do caminho (o seu ID).
>
> O histórico, o progresso de leitura, as **notas** e a preferência de layout
> ficam no dispositivo (localStorage); os arquivos e pastas vêm do Supabase.
>
> **PDFs grandes:** o leitor abre por *streaming* (URL assinada + range
> requests do PDF.js) e renderiza as páginas sob demanda conforme você rola —
> por isso arquivos pesados abrem rápido, sem baixar tudo antes.

---

## Stack

- **HTML/CSS/JS puro** — sem build, sem framework.
- **[Supabase](https://supabase.com)** — Auth + Storage privado.
- **[PDF.js](https://mozilla.github.io/pdf.js/)** — renderização do PDF no navegador.
- **[Inter](https://rsms.me/inter/)** — tipografia.
