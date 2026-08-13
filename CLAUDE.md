# Projeto: Painel de Sell-out Bravir (treino com dados reais)

## O que é
Um painel simples, em um único arquivo HTML, para visualizar o sell-out
da Bravir a partir de `base_sellout.xlsx`. Projeto de aprendizado, com
dados reais (já limpos) da empresa.

> Sugestão: renomeie o arquivo para `base_sellout.xlsx` (sem acentos nem
> espaços) para evitar problemas no código.

## Os dados
- Arquivo: `base_sellout.xlsx`, aba `Base de Dados`.
- 24 colunas. Período: jan/2025 a jun/2026 (18 meses).
- Granularidade: cada linha = um SKU, em um Cliente, em um Mês.

### Colunas (todas úteis — 24)
Identificação:
- `Data`, `Ano`, `Mês`, `COD SKU`, `SKU`
- `Bravir / Terceiros` — eixo marca própria (Bravir) × white label (Terceiros)
- `Marca`, `Categoria`, `Cliente`, `Segmento`, `# PDVs Cliente`
- `Vendedor` — responsável comercial pela conta

Sell out / Sell in:
- `Quantidade Sell Out`, `Valor (R$) Sell Out`, `Preço Sell Out`
- `Quantidade Sell in`, `Valor (R$) Sell in`, `Preço Sell in`

Margem e outros:
- `Margem Cliente (R$)`, `Margem Cliente %`
- `Margem Contribuição Bravir (R$)` — margem interna Bravir sobre o sell-in
- `Estoque CD + Loja (último dia do mês)`, `Venda Por PDV Total`

Novas colunas (jul/2026):
- `Estoque Bravir` — estoque no CD/armazém da Bravir, por SKU, último dia do mês
- `Custo de Trade (R$)` — custo de ações de trade marketing por SKU × Cliente × Mês

### Como entender as dimensões
- `Bravir / Terceiros`:
  - Bravir → marcas próprias: Bendita Cânfora, Bravir Tradicional, Laby, Alivik.
  - Terceiros (white label) → a coluna `Marca` traz o nome do varejista
    (Araújo, Assifarma, DPSP, Pague Menos, Panvel, Raia / Drogasil,
    São João, Venâncio).
- Categorias: Corporal, Cânfora multiuso, Manteiga de cacau,
  Protetor Labial, Protetor Solar.
- 9 clientes (redes de varejo). Segmento: só VAREJO.

### Atenção a dados faltantes
Algumas linhas têm vazio em `Preço Sell Out`, `Margem Cliente %`,
`Estoque CD + Loja` ou `Venda Por PDV Total`. Trate esses casos sem quebrar
(ex.: ignorar na conta ou mostrar "—"), e não invente valor.

## Como ler o arquivo no navegador
- NÃO leia o arquivo com fetch direto do disco — o navegador bloqueia
  arquivos locais (file://).
- Use um `<input type="file">` para EU selecionar o arquivo, e leia com a
  biblioteca SheetJS (xlsx) via CDN. Assim funciona só abrindo o HTML.

## Como quero trabalhar
- Antes de editar qualquer arquivo, me mostre o PLANO e espere eu aprovar.
- Mudanças pequenas e incrementais, uma de cada vez.
- Soluções simples, sem setup complicado. Roda abrindo o HTML no navegador.
- Explique o que você fez em linguagem simples — estou aprendendo.
- Valores em reais no formato brasileiro (R$ 1.234,56).

## Padrões
- Interface e comentários do código: em português.
- Privacidade: a base tem nomes de clientes e margem por cliente. Não subir
  o arquivo para repositório público (ver .gitignore na Missão de Git).

---

## Estado atual do painel (jul/2026)

> Para detalhes técnicos completos (funções, padrões, armadilhas), leia
> **`CONTEXTO_TECNICO.md`** antes de qualquer edição.

O painel está **em produção** em GitHub Pages. Tudo em `index.html` (~8.200 linhas).
Dependências via CDN: Chart.js 4.4.0, SheetJS 0.18.5, Firebase Compat 10.12.0.

### Seções implementadas (9)

1. **Visão Geral** — KPIs (SO, SI, margem, variação YoY), Evolução Mensal (barras SO+SI + linha margem), Top 15 SKUs (tabela 12 meses), tabela detalhada.
2. **Categorias** — Donut de mix por categoria, Evolução por Categoria.
3. **Clientes / PDVs** — Resumo por Cliente (tabela ordenável), Venda por PDV (barras com "Mix: N SKUs"), Evolução SI por Cliente, Evolução SI por Marca.
4. **SKUs / Produtos** — Top 10 SKUs (barras, nomes em 2 linhas, fonte 9px), Top 5 Evolução, Ranking Margem por SKU, Scatter Margem × Sell-Out.
5. **Marcas** — Heatmap, Mix de Marcas por Cliente (com barra ▶ Total geral), Evolução de Marcas, Margem por Marca, Evolução SI por Marca, Evolução SO Bravir vs Terceiros por Cliente e por Categoria (rótulos em k), Mix Bravir × Terceiros por Cliente (rótulos % + barra ▶ Total geral), Evolução % Margem Bravir vs Terceiros (4 linhas: margem interna e do cliente para cada segmento).
6. **Estoque & Cobertura** — Evolução de Estoque por Cliente, Cobertura em Dias, Evolução Estoque Bravir por Categoria, Cobertura Bravir (dias).
7. **Precificação** — Markup, Radar, Oportunidade de Repricing, Variação de Preço, Scatter Preço × PDV, Preço vs Volume, Evolução de Preço por Categoria e por Cliente.
8. **Análises Específicas** — Elasticidade Preço × Demanda, Sell Out × Temperatura Real (barras SO + linhas temperatura, API Open-Meteo com fallback dual archive+forecast), Custo de Trade por Cliente, ROI de Trade por Cliente.
9. **Previsão de Vendas** — Simulador por cliente (SO/SI/estoque projetados), sincronização Firebase, dashboard de acurácia. Responsáveis: João Almeida, Vitor Amaral, Mariana Gomes, João Wanderley, Rafael Júnior.

### Bugs resolvidos (não regredir)
- Estoque inflado: `Estoque CD + Loja` repetido por SKU — só conta a primeira ocorrência por (cliente, mês).
- Temperatura julho vazia: gráfico agora corta no último mês com sell-out > 0.
- Temperatura junho faltando: API ERA5 tem lag — faz fetch duplo (archive + forecast `past_days=92`) e mescla.
- Nomes de SKU ilegíveis no Top 10: callback de tick retorna array de strings para quebrar em 2 linhas.
- Campo `'Versão'` do Firestore migrado para `'Responsável'` via `_migrarPrevisoes()`.
