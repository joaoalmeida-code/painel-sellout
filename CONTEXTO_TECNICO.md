# Contexto Técnico — Painel Sell-Out Bravir

> Documento de onboarding para novos assistentes Claude Code.
> Leia este arquivo inteiro antes de fazer qualquer alteração no projeto.

---

## 1. Arquitetura geral

O projeto inteiro está em **um único arquivo**: `index.html` (~8.200 linhas).
Não há build system, bundler, framework ou backend. Basta abrir o arquivo no navegador.

Dependências via CDN (sem instalação):
- **Chart.js 4.4.0** — todos os gráficos
- **SheetJS (xlsx 0.18.5)** — leitura do arquivo Excel
- **Firebase Compat 10.12.0** — sincronização de previsões entre usuários (Firestore)
- **Anthropic SDK** (browser) — chat IA embutido no painel

O arquivo de dados `base_sellout.xlsx` **nunca vai para o repositório** (`.gitignore`).
O usuário o seleciona manualmente via `<input type="file">` em cada sessão.

---

## 2. Fluxo de dados

```
base_sellout.xlsx
  → FileReader (readAsArrayBuffer)
  → SheetJS XLSX.read()
  → preprocessar()        ← normaliza, calcula colunas derivadas, popula dadosGlobais[]
  → atualizarGrafico()    ← chama filtrarDados() e distribui para todos os renderizar*()
  → renderizar*(dados)    ← cada função recebe o array já filtrado
  → new Chart(canvas, config)
```

**`dadosGlobais`** é a fonte única de verdade — nunca modificado após `preprocessar()`.

---

## 3. Leitura da planilha — detalhe crítico

SheetJS é chamado com `header: 1`, que retorna **arrays**, não objetos:

```js
const rows = XLSX.utils.sheet_to_json(ws, { header: 1 });
```

Por isso `preprocessar()` constrói um mapa de índices a partir do cabeçalho:

```js
const colIdx = {};
rows[0].forEach((nome, i) => { colIdx[nome.trim()] = i; });
// acesso: rows[n][colIdx['Valor (R$) Sell Out']]
```

**Armadilha:** se o nome de uma coluna na planilha mudar (espaço extra, acento errado,
maiúscula diferente), o dado fica `undefined` silenciosamente — causa mais comum de
"painel não mostra nada" após atualizar a planilha.

As colunas que o código acessa por nome (verifique sempre):

| Coluna na planilha | Uso |
|---|---|
| `Ano`, `Mês` | Chave temporal `"ANO\|NomeMes"` |
| `SKU`, `COD SKU` | Identificação de produto |
| `Bravir / Terceiros` | Segmentação Bravir vs Terceiros |
| `Marca`, `Categoria`, `Cliente` | Dimensões de análise |
| `# PDVs Cliente` | Cálculo de venda por PDV |
| `Valor (R$) Sell Out` | Faturamento sell-out |
| `Quantidade Sell Out` | Volume sell-out |
| `Preço Sell Out` | Preço médio sell-out |
| `Valor (R$) Sell in` | Faturamento sell-in |
| `Quantidade Sell in` | Volume sell-in |
| `Preço Sell in` | Preço médio sell-in |
| `Margem Cliente (R$)` | Margem do varejista em R$ |
| `Margem Cliente %` | Margem do varejista em % |
| `Margem Contribuição Bravir (R$)` | Margem interna Bravir em R$ |
| `Estoque CD + Loja (último dia do mês)` | Estoque físico do varejista (total do cliente, repetido por SKU — requer deduplicação) |
| `Venda Por PDV Total` | Venda por PDV (pré-calculada) |
| `Margem Contribuição Bravir (R$)` | Margem interna Bravir (sobre sell-in) — usada em Evolução % Margem |
| `Vendedor` | Responsável comercial pela conta — popula filtro de Vendedor |
| `Estoque Bravir` | Estoque no CD/armazém da Bravir, por SKU, último dia do mês |
| `Custo de Trade (R$)` | Custo de trade marketing por SKU × Cliente × Mês |

---

## 4. Sistema de filtros

### `filtrarDados()`
Aplica todos os filtros ativos a `dadosGlobais` e retorna um novo array.
Filtros aplicados em ordem (todos são ANDs; array vazio = sem restrição):

1. **Período** — compara chave cronológica `ANO*100 + MESES_ORDEM[Mês]` com `keyDe`/`keyAte`
2. **Categoria** — array `catsSel`
3. **Marca** — array `marcasSel`
4. **Bravir/Terceiros** — string `filtroMarca` (`'Bravir'`, `'Terceiros'` ou `''`)
5. **Cliente** — array `clientesSel`
6. **SKU** — array `skusSel`
7. **Vendedor** — array `vendedoresSel`

### `dadosFiltradosSemPeriodo()`
Igual a `filtrarDados()` mas sem o filtro de período (etapa 1).
Usada onde o histórico completo importa: Top 15 SKUs (YTD), lista de SKUs disponíveis, elasticidade.

### `aplicarFiltro*(valor)`
Cada função atualiza a variável global correspondente e chama `atualizarGrafico()`.
`atualizarGrafico()` chama `filtrarDados()` uma única vez e distribui o resultado para
todos os `renderizar*(dados)`.

---

## 5. Convenções de renderização

### Regra geral
Todas as funções `renderizar*` recebem **`dados`** como único parâmetro (array já filtrado).
**Nenhuma** chama `filtrarDados()` internamente — essa responsabilidade é de `atualizarGrafico()`.

Exceções documentadas:
- `renderizarTop15SKUs` chama `dadosFiltradosSemPeriodo()` adicionalmente para calcular o % YTD
- `renderizarTemperatura` é `async` (faz fetch para a API Open-Meteo)
- `popularSKUs` chama `dadosFiltradosSemPeriodo()` para listar todos os SKUs sem corte de período
- `renderizarEstoque` e `renderizarCobertura` chamam `filtrarDados()` internamente (exceções históricas)
- `renderizarEstoqueBravir`, `renderizarCoberturaBravir` chamam `filtrarDados()` internamente
- `renderizarCustoTrade`, `renderizarROITrade` recebem `dados` como parâmetro (padrão correto)

### Destruição de instâncias
Antes de criar um gráfico, destrua a instância anterior:

```js
if (graficoXyz) graficoXyz.destroy();
graficoXyz = new Chart(canvas, config);
```

Se esquecer o `.destroy()`, o Chart.js sobrepõe uma instância sobre outra e o gráfico
fica sem resposta.

---

## 6. Chart.js 4 — padrões usados neste projeto

### Versão e carregamento
Chart.js 4.4.0 via CDN. **Nenhum plugin é registrado globalmente** com `Chart.register()`.

### Plugins inline
Sempre declarados como array na raiz do objeto de configuração (não dentro de `options`):

```js
new Chart(canvas, {
  type: 'bar',
  plugins: [{
    id: 'meuPlugin',          // id é obrigatório
    afterDatasetsDraw(chart) {
      const { ctx, chartArea: { top, bottom, left, right }, scales: { x, y } } = chart;
      // desenho customizado
    }
  }],
  data: { ... },
  options: { ... }
});
```

### API de coordenadas (Chart.js 4 — diferente das versões 2/3)
```js
// CORRETO (v4):
const px = scale.getPixelForValue(valorNumerico);

// ERRADO (v2/v3 — não funciona na v4):
const px = scale.getPixelForTick(índice);
```

Quando copiar código de exemplos da internet, verifique a versão do Chart.js.
A maioria dos exemplos online é v2/v3 e as APIs de escala são diferentes.

### Plugin reutilizável `_pluginRotulosKBT`
Constante global (declarada uma vez, usada em múltiplos gráficos).
Desenha rótulos em `k` (÷1000) acima das barras. Não pode carregar estado interno —
todas as informações vêm do objeto `chart` recebido no callback.

### Barras empilhadas com separador branco
Para distinguir segmentos em barras empilhadas:

```js
datasets: [{
  ...,
  borderWidth: { top: 2, right: 0, bottom: 0, left: 0 },
  borderColor: '#fff',
  barPercentage: 0.7,
  categoryPercentage: 0.75,
}]
```

### Multi-eixo (ex.: barras SO + linha temperatura)
```js
datasets: [
  { yAxisID: 'y',  ... },  // barras no eixo principal
  { yAxisID: 'y2', ... },  // linha no eixo secundário
],
options: {
  scales: {
    y:  { ... },
    y2: { position: 'right', grid: { drawOnChartArea: false }, ... }
  }
}
```

---

## 7. Armadilhas e bugs já resolvidos

### 7.1 Estoque inflado — RESOLVIDO
`Estoque CD + Loja` é um total do cliente repetido em cada linha de SKU.
Somar ingenuamente multiplica o valor por N (número de SKUs do cliente naquele mês).

**Fix aplicado** em `renderizarEstoque` e `renderizarCobertura`:
```js
// Só registra a primeira ocorrência por (cliente, mês)
if (!porMesCli[chave][cli]) {
  porMesCli[chave][cli] = num(est);
  totCli[cli] = (totCli[cli] || 0) + num(est);
}
```

### 7.2 Estoque Bravir ≠ Estoque CD + Loja — não confundir
`Estoque CD + Loja` = estoque do **varejista** (total do cliente, repetido por SKU — exige deduplicação).
`Estoque Bravir` = estoque no **armazém da Bravir** (por SKU — soma direta, sem deduplicação).
São perspectivas opostas da cadeia. `Estoque Bravir` baixo = risco de não abastecer o varejista.
`Estoque CD + Loja` baixo = risco de ruptura na gôndola do consumidor.

### 7.3 Cobertura ignora filtro de SKU — intencional
A cobertura de estoque representa o inventário físico total do cliente.
Não faz sentido dividir por SKU individual. A função aplica todos os filtros
**exceto** `skusSel`. Não "corrija" esse comportamento — seria um bug de subestimação.

### 7.4 YoY por offset de array, não subtração de ano
```js
const idx = todasChavesGlobais.indexOf(chAtual);
const chAnoAnterior = todasChavesGlobais[idx - 12]; // mesmo mês, ano anterior
```
Robusto para dados contínuos. Com gaps mensais (meses sem nenhuma venda),
o offset pode apontar para o mês errado. Não há correção aplicada — comportamento esperado.

### 7.5 Campo 'Versão' → 'Responsável' no Firestore
Previsões antigas salvas no Firestore usavam o campo `'Versão'`.
A função `_migrarPrevisoes(hist)` converte automaticamente ao carregar.
Qualquer consulta a dados históricos de previsão deve passar por esse helper.

### 7.6 Temperatura: archive API tem lag — RESOLVIDO
`archive-api.open-meteo.com` (ERA5) pode ter atraso de dias no processamento de meses recentes.
Fix aplicado: busca em paralelo nos dois endpoints e usa forecast como fallback:

```js
const [rArch, rFore] = await Promise.all([
  fetch('https://archive-api.open-meteo.com/...', { cache: 'no-cache' }).catch(() => null),
  fetch('https://api.open-meteo.com/v1/forecast?past_days=92', { cache: 'no-cache' }).catch(() => null),
]);
// Para cada mês: prefere archive (mais preciso histórico); se não tem dados usa forecast
```

### 7.7 Gráfico temperatura: julho aparecia vazio — RESOLVIDO
`MESES_TEMP` incluía o mês atual (julho) mesmo sem dados de sell-out.
Fix em `renderizarTemperatura`: corta os meses no último com sell-out > 0.

```js
const ultimoComSO = MESES_TEMP.reduce((last, m, i) => (porMes[m] > 0 ? i : last), -1);
const mesesEfetivos = MESES_TEMP.slice(0, ultimoComSO + 1);
```

### 7.8 Top 15 SKUs não seguia filtros — RESOLVIDO
O cálculo YTD usava `dadosGlobais` em vez de `dadosFiltradosSemPeriodo()`.
Corrigido: o loop de YTD agora respeita todos os filtros ativos exceto período.

### 7.9 Simulador: bloco 'scall' depende dos filhos
O bloco "Todos os Clientes" (`ns = 'scall'`) soma `siProj` e `estFim` dos blocos individuais.
`recalcularSimulador('scall')` só faz sentido **após** todos os blocos filhos recalculados.
O código já garante isso — não quebrar essa ordem ao modificar o simulador.

### 7.10 `ocultarCardsIrrelevantes()` usa `dataset.filtroOculto`
Cards ocultados programaticamente recebem `element.dataset.filtroOculto = 'true'`.
O sistema de toggle só atua nesses elementos — não em cards com `display:none` no HTML estático.

---

## 8. Constantes e helpers globais importantes

| Símbolo | Tipo | O que faz |
|---|---|---|
| `dadosGlobais` | `Array` | Todas as linhas da planilha pós-`preprocessar()`. Fonte única de verdade. |
| `MESES_ORDEM` | `Object` | `{ Janeiro: 1, ..., Dezembro: 12 }` — para comparações cronológicas. |
| `MESES_TEMP` | `Array` | Chaves `"ANO\|NomeMes"` de jan/2025 até o mês atual. Base para os gráficos de temperatura. |
| `_todasChavesGlobais()` | `Function` | Retorna array cronológico de todas as chaves `"ANO\|NomeMes"` presentes nos dados. Usado para YoY (offset –12). |
| `_mesAvancar(ch, n)` | `Function` | Avança/recua uma chave por `n` meses, respeitando virada de ano. Ex.: `_mesAvancar('2024\|Dezembro', 1)` → `'2025\|Janeiro'`. |
| `_lblCh(ch)` | `Function` | `"2024\|Janeiro"` → `"Jan/24"`. Para rótulos de eixo X. |
| `CORES` | `Array` | Paleta principal de cores para gráficos multi-série. |
| `CORES_5` | `Array` | Subconjunto de 5 cores para gráficos de até 5 séries. |
| `BRL(v)` | `Function` | Formata número como `R$ 1.234,56`. Retorna `'—'` para null/undefined. |
| `num(v)` | `Function` | `parseFloat(v) \|\| 0` — normaliza valores da planilha. |
| `vazio(v)` | `Function` | `true` se null, undefined ou string vazia. |
| `isMarcaBravir(m)` | `Function` | `true` para marcas próprias Bravir (Bendita Cânfora, Bravir Tradicional, Laby, Alivik). |
| `prioridadeMarca(m)` | `Function` | Retorna ordem de exibição: Laby=0, Bendita=1, Bravir=2, Alivik=3, Terceiros=4. |
| `aplicarFiltroCliente(c)` | `Function` | Atualiza filtro de cliente e chama `atualizarGrafico()`. |
| `aplicarFiltroMarca(m)` | `Function` | Idem para marca. |
| `aplicarFiltroSKU(s)` | `Function` | Idem para SKU. |
| `aplicarFiltroClienteMarca(c, m)` | `Function` | Aplica filtro de cliente + marca simultaneamente (clique em Mix Marcas). |

---

## 9. Firebase — uso e estrutura

Biblioteca: Firebase Compat 10.12.0 (modo compat, não modular).
Apenas **Firestore** é utilizado — sem Auth, Storage ou Realtime Database.

**Coleção:** `previsoes`

Cada documento representa uma previsão salva por um responsável para um cliente/período.
Campos: `cliente`, `responsavel`, `meses` (array de chaves), `soProj` (objeto mês→valor),
`siProj`, `estFim`, `cobDias`, `timestamp`.

Sincronização em tempo real via `onSnapshot()` — múltiplos usuários veem as mesmas previsões
sem recarregar a página.

O campo de responsável é selecionado em um `<select id="sel-responsavel">`. Para adicionar
novos nomes, basta adicionar um `<option>` nesse elemento.

---

## 10. Seções do painel e gráficos implementados

O painel tem **9 seções** navegáveis. Cada seção é um `div.secao-header` com `display:none`
que é exibido quando o dado tem informação suficiente para justificar a seção.

### Seção 1 — Visão Geral
- KPIs de sell-out (faturamento, quantidade, ticket médio, variação YoY)
- **Evolução Mensal** — barras SO + barras SI + linha Margem Bravir (função: `renderizarEvolucao`)
- **Top 15 SKUs — Sell-out Últimos 12 Meses** — tabela com 12 colunas mensais + Δ YTD (função: `renderizarTop15SKUs`)
- Tabela de sell-out detalhada por mês

### Seção 2 — Categorias
- **Donut Mix de Categorias** — rótulos externos com leader lines (função: `renderizarDonutCategorias`)
- **Evolução por Categoria** — barras mensais (função: `renderizarEvolucaoPorCategoria`)

### Seção 3 — Clientes / PDVs
- **Resumo por Cliente** — tabela unificada com 11 colunas, clicável, ordenável (função: `renderizarTabelaResumoClientes`)
- **Venda por PDV por Cliente** — barras horizontais + rótulo "Mix: N SKUs" (função: `renderizarVendaPorPDV`)
- **Evolução Sell-In por Cliente** — barras empilhadas com borda branca entre segmentos
- **Evolução Sell-In por Marca** — idem por marca

### Seção 4 — SKUs / Produtos
- **Top 10 SKUs por Faturamento** — barras horizontais, nomes quebrados em 2 linhas, fonte 9px (função: `renderizarTopSKUs`)
- **Evolução Top 5 SKUs** — linhas com checkboxes para toggle (função: `renderizarTop5SKUsEvolucao`)
- **Ranking Margem por SKU** — scatter + tabela
- **Scatter Margem × Sell-Out** — bolhas por SKU

### Seção 5 — Marcas
- **Heatmap Marcas × Meses** — intensidade por faturamento
- **Mix de Marcas por Cliente** — barras empilhadas 100%, com barra "▶ Total geral" no topo (função: `renderizarMixMarcaCliente`)
- **Evolução de Marcas** — linhas multi-série com checkboxes
- **Margem por Marca** — barras horizontais
- **Evolução Sell-In por Marca** — barras empilhadas
- **Evolução SO — Bravir vs Terceiros por Cliente** — barras agrupadas, checkboxes, rótulos em k (função: `renderizarEvoluBravirTercCliente`)
- **Evolução SO — Bravir vs Terceiros por Categoria** — idem por categoria (função: `renderizarEvoluBravirTercCategoria`)
- **Mix Bravir × Terceiros por Cliente** — barras empilhadas 100%, barra "▶ Total geral", rótulos % dentro das barras (função: `renderizarMixBravirTercCliente`)
- **Evolução % Margem — Bravir vs Terceiros** — 4 linhas: margem interna e margem cliente, para Bravir e Terceiros (função: `renderizarMargemBravirTerc`)

### Seção 6 — Estoque & Cobertura
- **Evolução Estoque por Cliente** — linhas com checkboxes (função: `renderizarEstoque`)
- **Cobertura de Estoque (dias)** — barras + linha meta (função: `renderizarCobertura`)

### Seção 7 — Precificação
- Markup por Cliente
- Radar de Precificação
- Oportunidade de Repricing
- Ranking Variação de Preço
- Scatter Preço × PDV
- Preço vs Volume
- Evolução de Preço por Categoria e por Cliente

### Seção 8 — Análises Específicas
- **Elasticidade Preço × Demanda** — scatter com regressão log-log (função: `renderizarElasticidade`)
- **Sell Out × Temperatura Real** — barras de quantidade + linhas de temperatura média e mínima, dropdown de cliente, dados Open-Meteo (função: `renderizarTemperatura`)

### Seção 9 — Previsão de Vendas
- Simulador dinâmico por cliente: tabela de entrada de SO projetado, cálculo automático de SI e cobertura
- KPIs YoY do simulador
- Gráfico combinado histórico + projeção com linha separadora
- Sincronização via Firebase (múltiplos responsáveis podem salvar previsões)
- Responsáveis configurados: João Almeida, Vitor Amaral, Mariana Gomes, João Wanderley, Rafael Júnior
- Dashboard de acurácia de previsões salvas vs realizado

---

## 11. Como fazer alterações com segurança

1. **Leia `CLAUDE.md` primeiro** — contém as regras de trabalho do projeto.
2. **Apresente o plano antes de editar** — o usuário quer aprovar antes de qualquer mudança.
3. **Uma mudança por vez** — cada commit deve ser atômico e descritivo.
4. **Antes de criar qualquer novo gráfico**, verifique se a variável `grafico*` correspondente está declarada no bloco de globais (~linha 1200 do arquivo) e se a chamada à função está em `atualizarGrafico()`.
5. **Sempre destrua o gráfico anterior** com `.destroy()` antes de criar um novo.
6. **Plugins inline** devem ter `id` único — Chart.js usa esse id internamente.
7. **Ao mexer em filtros**, teste com todos os filtros ativos simultaneamente para verificar que não há conflito.
8. **Ao mexer no simulador**, não altere a ordem de recálculo dos blocos (filhos antes de 'scall').
9. **Commit e push** ao final de cada alteração aprovada — o painel está publicado em GitHub Pages.

---

## 12. Repositório e publicação

- **GitHub:** `https://github.com/joaoalmeida-code/painel-sellout`
- **GitHub Pages (URL pública):** `https://joaoalmeida-code.github.io/painel-sellout/`
- Branch principal: `master`
- Cada `git push origin master` atualiza o painel publicado automaticamente (GitHub Pages via branch master).
- O arquivo `base_sellout.xlsx` está no `.gitignore` — nunca commitar dados reais.
