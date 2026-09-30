# Metas 2026 — ponto de situação a 30/09/2026

Aplicação independente do painel PRR Status (página própria, sem ligação ao painel). Link público: `https://xanadudevs.github.io/prrstatus/metas2026/`

Serve para preencher o ficheiro **CP2026_Tabela_Anexos** (folha `Metas 2026`):

- **Coluna AB — % Execução em 30/09/2026**: a preencher para cada meta.
- **Coluna AD — % Execução prevista em 30/11/2026**: vem pré-preenchida com a previsão já reportada; só se altera quando, no último mês, houve alterações suscetíveis de a afetar (atrasos, reprogramações, bloqueios, alterações de âmbito ou evolução relevante dos trabalhos). Quando é alterada, a página pede o **motivo** (lista fixa) e guarda também o valor original, para se ver "era X%".
- **Coluna AC — OBS**: ponto de situação / justificação em texto livre.

Guarda os dados no Supabase, numa tabela própria (`metas`) — não mexe nos dados do painel PRR. Corre uma vez no **SQL Editor**:

```sql
create table metas (
  id text primary key,              -- ID_Meta (coluna A)
  linha int,                        -- linha no Excel original
  dados jsonb not null default '{}'::jsonb, -- restantes colunas (projeto, meta, gestor, coordenação, datas, valores, % a 31/08, ...)
  exec_0930 numeric,                -- coluna AB, 0–100
  obs text,                         -- coluna AC
  prev_1130 numeric,                -- coluna AD (revista), 0–100
  prev_1130_original numeric,       -- coluna AD tal como veio no Excel importado
  motivo_revisao text,
  updated_at timestamptz,
  updated_by text
);

alter table metas enable row level security;

create policy "Leitura pública" on metas for select using (true);
create policy "Escrita pública" on metas for insert with check (true);
create policy "Atualização pública" on metas for update using (true);
create policy "Remoção pública" on metas for delete using (true);
```

Fluxo:
1. **Importar Excel** com o ficheiro CP2026_Tabela_Anexos (uma vez, para semear). Reimportar mais tarde é seguro: atualiza os dados de base (nomes, valores, datas, % a 31/08, previsão original) mas **mantém** o que já foi preenchido na página (30/09, OBS, previsão revista). Metas que deixem de constar do ficheiro são removidas (a confirmação diz quantas).
2. Cada gestor filtra pela sua coordenação/nome e preenche. Grava automaticamente ao sair de cada campo; o campo "Preenchido por" (guardado só no browser de cada pessoa) fica registado em cada alteração.
3. **Preencher Excel original**: escolhe-se o ficheiro original e a página devolve uma cópia com AB, AC e AD preenchidas. O ficheiro é editado diretamente (XML da folha), por isso formatação, outras folhas, tabela dinâmica e validações ficam intactas. Nas metas com previsão revista, a OBS exportada começa por "Previsão 30/11 revista de X% para Y% (motivo): ...". As fórmulas existentes na coluna AE (`=S*AD`) são recalculadas ao abrir.
4. **Exportar resumo**: `.xlsx` simples com as metas filtradas, previsão original vs. revista, motivo, alertas e quem preencheu — útil para a Direção.

Alertas que a página assinala em cada meta: execução a 30/09 por preencher; execução a 30/09 inferior à de 31/08; previsão a 30/11 inferior à execução já alcançada; previsão alterada sem motivo (ou sem OBS); data de fim já ultrapassada sem a meta estar a 100%; e previsão mantida mas que exigiria um ritmo muito superior ao do último mês.

Não há login: qualquer pessoa com o link pode editar. Os dados das metas ficam no Supabase, não no repositório — o ficheiro Excel não é publicado aqui.
