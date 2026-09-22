# Painel PRR — configuração

Painel de acompanhamento de projetos PRR: unidade, estado, taxa de execução do projeto, taxa de execução financeira, investimento, dependências, riscos e próximos passos. Página estática (`prr-dashboard.html`), sem build step, hospedada no GitHub Pages deste repositório (`prrstatus`).

Link público: `https://xanadudevs.github.io/prrstatus/prr-dashboard.html`

## Persistência dos dados

O painel guarda os dados numa base de dados **Supabase** (Postgres gratuito) — sem backend próprio, sem servidor a manter, e **sem nenhum passo manual para quem usa a página**: não há login nem botão para ligar. Qualquer gestor de projeto abre o link e já pode editar.

Já tentámos duas abordagens antes desta:
- **Ficheiro `projects.json` no GitHub, com um token pessoal embutido no código.** Funcionava para um único utilizador, mas não fazia sentido para vários gestores de projeto (implicava dar a cada um um token com acesso de escrita ao repositório inteiro) — e, na prática, o GitHub **revoga automaticamente tokens que deteta expostos em código publicado**, o que tornou a solução inviável mesmo para um único utilizador.
- Antes disso, um ecrã "Ligar ao GitHub" onde cada pessoa colava o seu próprio token — resolvia a exposição, mas continuava a exigir que cada gestor de projeto criasse e gerisse um token do GitHub, o que não é razoável pedir a quem só quer atualizar o estado de um projeto.

O Supabase resolve isto de forma diferente: a chave usada no código (`SUPABASE_ANON_KEY`) **é suposto ficar pública** — ao contrário de um token do GitHub, não dá acesso de escrita por si só. A segurança fica do lado da base de dados, através de regras de **Row Level Security (RLS)**.

### ⚠️ Sem login, qualquer pessoa com o link pode editar (e apagar) dados

Como não há autenticação de utilizadores, a política de RLS usada aqui (ver SQL abaixo) permite leitura e escrita públicas — **qualquer pessoa com o link da página consegue alterar ou apagar projetos**, não só os gestores. Se isto alguma vez deixar de ser aceitável (ex: o link for partilhado mais largamente), a evolução natural é adicionar autenticação do Supabase (ex: magic link por email) e restringir as políticas de RLS a utilizadores autenticados — não é algo que esteja implementado agora, mas o Supabase suporta isso sem mudar de base de dados.

### Como configurar (uma vez)

1. Cria uma conta gratuita em [supabase.com](https://supabase.com) e um novo projeto (região à tua escolha, password da BD à tua escolha — não é usada pelo painel).
2. No projeto, abre **SQL Editor** e corre:
   ```sql
   create table projects (
     id text primary key,
     nome text not null,
     unidade text,
     ficha text,
     estado text not null default 'em_execucao',
     gestor text,
     taxa_projeto numeric,
     taxa_financeira numeric,
     investimento_total numeric,
     dependencias text,
     riscos text,
     proximos_passos text,
     updated_at timestamptz not null default now()
   );

   alter table projects enable row level security;

   create policy "Leitura pública" on projects for select using (true);
   create policy "Escrita pública" on projects for insert with check (true);
   create policy "Atualização pública" on projects for update using (true);
   create policy "Remoção pública" on projects for delete using (true);

   create table project_snapshots (
     id bigint generated always as identity primary key,
     project_id text not null,
     week_start date not null,
     estado text,
     taxa_projeto numeric,
     taxa_financeira numeric,
     investimento_total numeric,
     captured_at timestamptz not null default now(),
     unique (project_id, week_start)
   );

   alter table project_snapshots enable row level security;

   create policy "Leitura pública" on project_snapshots for select using (true);
   create policy "Escrita pública" on project_snapshots for insert with check (true);
   create policy "Atualização pública" on project_snapshots for update using (true);
   create policy "Remoção pública" on project_snapshots for delete using (true);
   ```
   `project_snapshots` é opcional: só é usada para mostrar a variação face à semana anterior (ver secção "Evolução semanal" abaixo). Sem ela, o painel funciona na mesma, só sem essas variações. Nota que `project_id` não tem uma foreign key para `projects.id` de propósito — assim, reimportar um Excel (que apaga e recria a tabela `projects`) não arrasta consigo o histórico semanal já acumulado.
3. Vai a **Project Settings → API** e copia o **Project URL** e a chave **`anon` `public`**.
4. No `prr-dashboard.html`, procura estas duas linhas e substitui pelos valores copiados:
   ```js
   const SUPABASE_URL = 'COLOCAR_SUPABASE_URL_AQUI';
   const SUPABASE_ANON_KEY = 'COLOCAR_SUPABASE_ANON_KEY_AQUI';
   ```
5. Faz commit e push dessa alteração (ou pede para eu o fazer, colando-me os dois valores diretamente — ao contrário do token do GitHub, não há problema nenhum em estes ficarem no código nem no histórico do git).
6. Usa o botão **"Importar Excel"** uma vez para semear a tabela com os projetos atuais.

Enquanto `SUPABASE_URL`/`SUPABASE_ANON_KEY` não estiverem preenchidos, o painel mostra um aviso no topo e as edições ficam só em memória do browser (perdem-se ao recarregar).

## Fluxo de trabalho: importar em bloco ou gerir projeto a projeto

1. **Carregar os dados em bloco** — botão **"Importar Excel"**, que lê um ficheiro no formato "Ponto de Situação Projeto PRR" (folha `PDS PRR`, com cabeçalhos como "Unidade", "N. Ficha Projecto", "Nome do Projetos", "Estado", "Taxa de execução projeto", "Investimento total", "Taxa de execução financeira", "Dependências", "Riscos", "Próximos Passos", "Gestor de Projeto") e substitui todos os projetos atuais pelos do ficheiro.
   - As taxas podem vir em fração (`0.7`) ou já em percentagem (`70`) — o painel deteta automaticamente.
   - O campo "Estado" aceita as variações do Excel de origem ("em atraso", "em execução", "por iniciar"/"por inciar", "concluído") e mapeia para os quatro estados do painel.
   - Se o Supabase estiver configurado, a importação é logo gravada na base de dados; caso contrário fica só na sessão.
   - A operação pede confirmação antes de substituir os dados, porque é destrutiva — usa-se tipicamente para semear ou repor a lista completa (ex: no início de um novo período de reporte).
   - Antes de reimportar (substituir tudo), é boa prática guardar primeiro uma cópia dos dados atuais com o botão **"Exportar Excel"** — gera um `.xlsx` no mesmo formato, que serve de backup e pode ser reimportado se algo correr mal.
2. **Gerir projeto a projeto** — botão **"+ Novo projeto"** para criar um projeto individual, e botão "editar" em cada cartão para atualizar os campos de um projeto já existente (estado, taxas, investimento, dependências, riscos, próximos passos, gestor). Não é preciso voltar a importar Excel para isto. Não há, propositadamente, um botão para remover projetos na aplicação — para tirar um projeto da lista, volta a importar-se um Excel atualizado sem esse projeto.

## O que o painel mostra

- **Resumo no topo**: projetos em execução, concluídos, em atraso, taxa média de execução do projeto, investimento total, taxa média de execução financeira — recalculado consoante os filtros ativos (estado e coordenação), com a variação face à semana anterior ao lado de cada número (ver "Evolução semanal" abaixo).
- **Filtros por estado**: em atraso, em execução, por iniciar, concluído.
- **Filtros por coordenação**: gerados automaticamente a partir das unidades presentes nos dados (ex: UIA, UPACE, UID, URN). Combinam-se com o filtro de estado.
- **Resumo executivo / alertas**: texto gerado automaticamente a partir dos dados atuais (nível de execução, evolução semanal, heterogeneidade, desfasamento física vs. financeira, projetos que requerem atenção). Mostra "Alertas para a Direção" quando não há filtro de coordenação selecionado, ou "Resumo Executivo — [Coordenação]" quando se filtra por uma coordenação específica. É a mesma lógica usada no slide de sumário executivo do PowerPoint.
- **Cartões agrupados por unidade** (UIA, UPACE, UID, URN), com ponto de cor por estado, barras de execução do projeto e financeira (cada uma com a respetiva variação semanal), investimento total e caixa de riscos em destaque.
- **Exportar PowerPoint**: gera um `.pptx` (via PptxGenJS, no browser) com um slide de sumário executivo, um slide de visão global para a Direção e slides por unidade.

## Evolução semanal

Sempre que a página carrega os dados (ou sempre que se cria/edita um projeto ou se importa um Excel), o painel grava automaticamente, na tabela `project_snapshots`, uma "fotografia" da semana corrente (estado, taxas, investimento de cada projeto) — no máximo uma por projeto por semana ISO (segunda a domingo); voltar a carregar a página na mesma semana só atualiza essa fotografia, não cria linhas a mais.

Com isso, o painel consegue comparar os valores atuais com a fotografia mais recente **anterior** à semana corrente, e mostrar a diferença:
- Nos indicadores do topo (contagens e médias/somas).
- Nas barras de execução do projeto e financeira, dentro de cada cartão.
- Numa frase adicional no resumo executivo/alertas (ex: "a taxa média de execução subiu 3 pontos percentuais" ou "X entrou em atraso desde então").

Não há dados anteriores a este mês (a funcionalidade só começa a acumular histórico a partir de agora), por isso as variações só aparecem depois de a tabela `project_snapshots` ter pelo menos uma fotografia de uma semana anterior à atual — até lá, os números aparecem normalmente, sem variação ao lado.

### Captura fiável à sexta-feira, 23h (recomendado)

A fotografia feita pelo browser (acima) só acontece se alguém tiver a página aberta nessa altura — não é garantido que aconteça exatamente à sexta-feira às 23h, só que acontece "algures durante a semana". Para garantir que a comparação é sempre feita em relação ao **fim da semana anterior**, corre esta SQL uma vez no **SQL Editor** do Supabase — cria uma tarefa agendada (via `pg_cron`, incluído no Supabase) que grava a fotografia da semana diretamente na base de dados, todas as sextas-feiras às 23h, sem depender de ninguém ter a página aberta:

```sql
create extension if not exists pg_cron;

create or replace function capture_weekly_snapshot()
returns void
language plpgsql
as $$
begin
  insert into project_snapshots (project_id, week_start, estado, taxa_projeto, taxa_financeira, investimento_total, captured_at)
  select id, date_trunc('week', current_date)::date, estado, taxa_projeto, taxa_financeira, investimento_total, now()
  from projects
  on conflict (project_id, week_start) do update set
    estado = excluded.estado,
    taxa_projeto = excluded.taxa_projeto,
    taxa_financeira = excluded.taxa_financeira,
    investimento_total = excluded.investimento_total,
    captured_at = excluded.captured_at;
end;
$$;

select cron.unschedule(jobid) from cron.job where jobname = 'captura-semanal-sexta-23h';
select cron.schedule('captura-semanal-sexta-23h', '0 23 * * 5', 'select capture_weekly_snapshot();');
```

Notas:
- Se `create extension if not exists pg_cron;` der erro de permissões, ativa a extensão pelo **Database → Extensions** do Supabase (procurar "pg_cron" e ativar) e depois corre o resto do bloco.
- O `pg_cron` corre em UTC, não em hora de Lisboa — `'0 23 * * 5'` é "sexta-feira às 23h UTC", ou seja, 23h em Lisboa no horário de inverno e 00h (já sábado) no horário de verão (desfasamento de 1h, sem grande impacto numa fotografia semanal).
- Isto não substitui a captura feita pelo browser — as duas coexistem. A do browser serve de rascunho/reserva; a tarefa agendada é que garante a versão final de cada semana, à sexta-feira 23h, independentemente de alguém ter aberto a página.
- Podes confirmar que a tarefa ficou criada com `select * from cron.job;` no SQL Editor.

## Nota de segurança

Os dados dos projetos ficam na base de dados Supabase, não neste repositório. A chave `SUPABASE_ANON_KEY` escrita em `prr-dashboard.html` é segura por design — mas, como não há autenticação de utilizadores, qualquer pessoa com o link da página pode editar ou apagar dados (ver aviso na secção "Persistência dos dados" acima). `projects.json` deixou de ser usado; pode ser removido do repositório quando quiseres.
