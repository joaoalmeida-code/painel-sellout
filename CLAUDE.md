# Projeto: Painel de Sell-out Bravir (treino com dados reais)

## O que é
Um painel simples, em um único arquivo HTML, para visualizar o sell-out
da Bravir a partir de `base_sellout.xlsx`. Projeto de aprendizado, com
dados reais (já limpos) da empresa.

> Sugestão: renomeie o arquivo para `base_sellout.xlsx` (sem acentos nem
> espaços) para evitar problemas no código.

## Os dados
- Arquivo: `base_sellout.xlsx`, aba `Base de Dados`.
- 2.919 linhas, 21 colunas. Período: jan/2025 a mai/2026 (17 meses).
- Granularidade: cada linha = um SKU, em um Cliente, em um Mês.

### Colunas (todas úteis — 21)
Identificação:
- `Data`, `Ano`, `Mês`, `COD SKU`, `SKU`
- `Bravir / Terceiros` — eixo marca própria (Bravir) × white label (Terceiros)
- `Marca`, `Categoria`, `Cliente`, `Segmento`, `# PDVs Cliente`

Sell out / Sell in:
- `Quantidade Sell Out`, `Valor (R$) Sell Out`, `Preço Sell Out`
- `Quantidade Sell in`, `Valor (R$) Sell in`, `Preço Sell in`

Margem e outros:
- `Margem Cliente (R$)`, `Margem Cliente %`
- `Estoque CD + Loja (último dia do mês)`, `Venda Por PDV Total`

### Como entender as dimensões
- `Bravir / Terceiros`:
  - Bravir → marcas próprias: Bendita Cânfora, Bravir Tradicional, Laby.
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
