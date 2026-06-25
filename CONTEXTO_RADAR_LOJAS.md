# CONTEXTO — RADAR LOJAS (handoff técnico)

> Documento de continuidade. Permite retomar o desenvolvimento do `radar-lojas.html` em qualquer ambiente sem perder contexto. Última atualização: 15/06/2026.

---

## 1. O que é

**Radar Lojas** é uma aplicação standalone (arquivo HTML único, processamento 100% client-side no navegador) de **diagnóstico do canal loja** do Grupo Alcina Maria (rede O Boticário, Penedo/AL). O usuário arrasta relatórios brutos exportados do ERP e a app produz uma **análise, não uma prescrição**: leitura do canal, benchmark entre as lojas, curva ABC e mix por loja, achados por loja com materialidade em R$ e ressalvas, **uma aba de análise estruturada (Gini/Pareto/Lorenz, coef. de variação, fronteira peer), contexto de movimentação de equipe (§14), análise de sortimento por SKU cruzado entre lojas (§15), uma aba de Lucro (pools de lucro, lucro/cupom, bridge de drivers, mix por categoria, risco de lucro, gap de categoria, margem mix×taxa, estrutura de custo) + correlação entre alavancas (§16), uma aba de Apresentação (capa + plano, abre por padrão, §17), e uma aba de Categorias com a taxonomia oficial do usuário (§18)**. **Remodelada de motor de estratégias → ferramenta de diagnóstico em 15/06/2026 (ver §13) — saíram checklist de ações, "potencial total", waterfall e simulador.**

- **Stack:** HTML/CSS/JS vanilla + pdf.js 3.11.174 (cdnjs) + SheetJS 0.18.5 (cdnjs). Sem build, sem servidor, sem localStorage (estado só em memória).
- **Padrão da casa:** mesmo modelo das ferramentas Foco Atividade e painel-vendas (arquivo único que processa arquivos reais no browser).
- **Privacidade:** nenhum dado sai da máquina.

## 2. Contexto de negócio

- Rede com **6 lojas físicas** (canal loja) sob a operação ACQUA DISTRIBUIDORA DE PERFUMES E COSMETICOS LTDA. As 7 fontes de dados mapeiam para 6 lojas porque a **Coruripe migrou de código no período** (5905 antigo → 24670 atual) e o motor soma os dois.
- **De-para código → nome (confirmado pelo usuário em 13/06/2026):** 24303 = Loja São Sebastião · 24669 = Loja Penedo · 24668 = Loja Palmeira · 24617 = Loja Palmeira Sustentável · 24671 = Loja Teotônio · 24670 (+5905) = Loja Coruripe. Configurado em `CONFIG.nomesLojas` e `CONFIG.mergeLojas`.
- Período analisado: **26/12/2025 a 12/06/2026** (169 dias ≈ 5,55 meses).
- Motivação: o canal loja está fraco em faturamento; o dono quer estratégias acionáveis.

### Diagnóstico validado da loja 24303 (referência)
| Indicador | Valor |
|---|---|
| Faturamento (ABC) / Venda líquida | R$ 332.282,11 / R$ 328.564,51 |
| GMV (painel, **já líquido de descontos**) | R$ 321.502,17 → R$ 57,9k/mês |
| Lucro bruto / margem (ABC) | R$ 206.784,90 / 62,2% |
| Cupons (boletos) | 1.552 → **9,2/dia** (baixo) |
| Ticket médio | **R$ 207,15** (alto) · Itens/cupom **2,45** (baixo) |
| Preço de etiqueta (GMV+descontos) | R$ 435.036,94 |
| Descontos | R$ 113.534,77 = **26,1% da etiqueta** = R$ 73,15/cupom |
| Fidelidade | 94,48% (régua: ≥95 ótimo, ≥85 aceitável) |
| Trocas | 23 / R$ 3.717,60 = 1,12% (saudável) |
| Recarga cartão presente | **R$ 0** no período |
| B1 | R$ 61.047,22 (19% do GMV) · 669 un |
| Curva ABC | A=220 SKUs (80% fat), B=228, C=254 · 702 SKUs |
| Mix | Perfumaria 52% · C&B 15,5% · Kits 14,5% · Splash/spray 10,1% · **make+skin+capilar só 6,2%** (recorrência) |
| Cauda longa | 205 SKUs < R$100 no período (29%) · 160 SKUs com 1 un |

**Leitura central:** problema é fluxo (9 cupons/dia) + desconto + mix raso de recorrência; ticket, fidelidade e trocas são pontos fortes.

**Inconsistências conhecidas (não bloqueantes):** (a) Receita líquida do painel (330,8k) > GMV (321,5k) — em aberto, possivelmente GMV exclui alguma classe de item; (b) Fidelidade-Qtd de boletos (1.659) > boletos GMV (1.552) — bases distintas do sistema; usamos a penetração reportada.

## 3. Fontes de dados (formatos de entrada)

### 3.1 PDF "Relatório de ABC de Vendas" (RetaguardaGB 8.0.x) — 1 por loja
- Colunas por item: `Num. | Código (14 dígitos) | Produto | Quantidade (x,000) | Faturamento | Prc. Médio | Custo Total | Estadual | Federal | Lucro | Margem% | Markup% | Partic.% | Acum.%`.
- **Layout traiçoeiro:** nomes de produto quebram em 2–3 linhas visuais (parte do nome antes E/OU depois da linha numérica). A linha de filtros do cabeçalho se repete em toda página e quebra em 3 linhas (`Filtros: ...` / `[24303]; Tipo de Data: ...` / `Exibir Visão CMV: [Não];`).
- Metadados extraídos: código da loja (`Lojas: [NNNNN]`), período, razão social, e rodapé: `TOTAL GERAL` (qtd, fat, custo, lucro), `TOTAL DE ITENS DISTINTOS`, `TOTAL DE TROCAS` (qtd + R$), `TOTAL DE DEVOLUÇÕES`, `VENDA LIQUIDA`.
- Decimal brasileiro (1.234,56); quantidade com 3 decimais; margem/markup com ponto (42.56%).

### 3.2 Planilha de indicadores (.xlsx ou .csv) — exportação "Listar Por Lojas"
- A exportação real do painel vem em **.csv** (ANSI/cp1252, separador `;`, decimal brasileiro). O parser lê CSV como texto (`XLSX.read` com `type:'string', raw:true` + `decodeTexto` com fallback UTF-8→windows-1252) — **nunca** deixar o SheetJS coagir número em CSV ("207,15" viraria 20715).
- 1 linha por loja: `"24303 - RAZAO SOCIAL"` + colunas com header composto: `GMV-GMV`, `GMV-Boleto médio`, `GMV-Qtd de boletos`, `GMV-Itens por boleto`, `Meta receita-*` (vieram **vazias**), `Valores de vendas-Receita líquida`, `...(-) trocas`, `...B1`, `Fidelidade-Qtd de boletos`, `Fidelidade-Penetração boletos`, `Descontos-Total de descontos`, `Trocas-Trocas`, `Trocas-Qtd de trocas`, `Cartão presente-Recarga`, `Quantitativo-B1`.
- **Definição confirmada pelo usuário: o GMV já vem líquido de descontos.** Logo `etiqueta = gmv + descontos` e `descEtiqPct = descontos/etiqueta`.

## 4. Arquitetura do arquivo `radar-lojas.html` (~115 KB)

Ordem interna: `<head>+CSS` → `<body>` estático → CDN scripts → `<script>` com 4 blocos:

1. **CONFIG** — de-para `nomesLojas` (editável pelo usuário no próprio arquivo), `diasMes: 30.44`, réguas (`regua.*`), limites do simulador, `itemCrossFallback: 45`.
2. **Parsers (validados):**
   - `buildLines(items)` — reconstrói linhas do pdf.js agrupando text items por Y (tolerância 2.5) e ordenando por X; concatena **sem espaço** se o gap horizontal < 1.2 (evita quebrar o código de 14 dígitos).
   - `DATA_RE` — regex única que casa a linha numérica do item (num opcional + código 14 díg. + nome inline opcional + 7 valores + margem% + markup%).
   - `isSkipLine()` — descarta cabeçalhos/rodapés; o truque para a linha de filtros quebrada é `/\];/ && /:/` (nomes de produto nunca têm `];`).
   - Montagem do nome: se `nomeInline` vazio → formato quebrado → usa linha órfã anterior (`nomePre`) + seguinte (`nomePos`).
   - `validarABC()` — soma os itens e confere contra o rodapé (qtd, fat, custo, lucro, nº de itens). Cada arquivo recebe selo ✓/⚠ na UI. **Não confiar em parse sem esse selo.**
   - `parseIndicadores()` — SheetJS, localiza a linha de header por `listar por lojas`/`gmv-gmv`, mapeia colunas via `COLMAP` (chaves normalizadas sem acento), extrai código da loja por `/^(\d{3,})\s*-\s*(.+)$/`.
3. **Análise (`analisarABC`)** — por loja: curva ABC (A≤80%, B≤95%, C resto), mix por categoria e marca (`marcaDe`/`categoriaDe` por regex de prefixo/tokens — Floratta, Malbec, CBEM=Cuide-se Bem, NSPA=Nativa SPA, Botik etc.), recorrência = fat(Make+Skincare+Capilar)/fat, `precoItemCB` (preço médio de Corpo & Banho, usado como proxy do item de venda adicional), brindes/PRM (prc<5 ou `\bPRM\b`), lucro negativo, cauda (<R$100), `qtd=1`, kits por data (`NAT|MAES|AMOR|PAIS|CRIANC + /AA`), top20share, dias do período.
4. **Estado/merge** — `state.stores: Map(codigo → {pdfMeta, abc, ind})`; `derivar(s)` produz o objeto plano por loja (gmvMes, boletosDia, descEtiqPct, descPorBoleto, trocasPct, margem, recorrPct...); `canalAgregado(lojas)` faz somas e médias ponderadas (itens/cupom e fidelidade ponderados por boletos; margem e recorrência por faturamento ABC).
5. **Motor de estratégias (`gerarEstrategias`)** — 9 regras; cada uma retorna `{sev: critico|atencao|oportunidade, somavel, impacto (R$/mês), titulo, diag, meta, acoes[], fonte}`. Ordenação: severidade → impacto. **Somente impactos `somavel:true` entram no "potencial total"** (desconto, cesta, fluxo, giftcard, trocas); mix de recorrência mostra R$ "indireto" e fica fora da soma; fidelidade/cauda/heróis são qualitativas.
   - Desconto: alvo = média do canal (n>1) ou −3pp (mín. 18%); `impacto = Δpp/100 × etiquetaMes`.
   - Cesta: alvo = max(+0,3; média canal), teto 3,2; `impacto = Δitens × boletosMes × precoItemCB||45`.
   - Fluxo: alvo = média do canal (se abaixo de 90% dela) ou +15%; `impacto = Δcupons/dia × 30,44 × ticket`.
   - Giftcard (recarga=0): `impacto = 1% × gmvMes` (estimativa marcada).
   - Trocas (>1,5%): `impacto = pp acima da régua × receitaMes`.
   - Com 1 loja, réguas absolutas; com 2+ lojas o motor **recalibra automaticamente pela média do canal**.
6. **Render** — 4 abas (Visão do canal / Lojas / Estratégias / Simulador): KPI pills, ranking com heatmap por coluna (direção configurada: desconto/trocas menor=melhor), cards de loja com **réguas de semáforo** (assinatura visual; `SPEC` mapeia valor→posição no gradiente verde→âmbar→vermelho), donut SVG de mix, barras da curva ABC, top 10 SKUs, chips de marcas, alertas; aba **Estratégias** abre com **waterfall do GMV** no topo (ver §12) + cards com checklist; simulador com 3 sliders (Δdesconto pp, Δitens/cupom, Δcupons/dia) e conta aberta + projeção mensal/anual; botão **"Copiar resumo p/ Slack"**.
7. **SEED** — desde 12/06/2026 embute **as 7 lojas** (`SEED.pdfs[]` com meta do rodapé + itens minificados `[sku, nome, qtd, fat, custo, lucro]` de cada PDF, + `SEED.xlsx[]` com as 7 linhas de indicadores do CSV). App abre com o canal completo; re-upload de qualquer arquivo substitui a loja correspondente. Seed gerado pelo próprio parser do app via harness Node (`gen-seed.js`), com selo de rodapé exigido na geração.

## 5. Validações já executadas (não regredir)

1. **Parser Python de referência** (pdftotext -layout + regex): 702 itens, somas = rodapé com precisão de centavo.
2. **Parser pdf.js em Node** (mesma lógica do browser): validado em PDF sintético no layout RetaguardaGB (formatos A e B + rodapé) — metadados e somas 100%.
3. **Leitor SheetJS em Node**: 22 campos = planilha original.
4. **Smoke test do núcleo** (seed → derivar → canal → estratégias): dias=169, descEtiq=26,10%, margem=62,2%, curva A=220 SKUs, 7 estratégias, potencial R$ 15.202/mês.
5. **Render completo em jsdom**: zero erros; 8 pills, 1 card, 5 réguas, 7 estratégias, 3 sliders, projeção correta.

Critério permanente: **toda mudança no parser deve manter o selo ✓ de rodapé nos PDFs reais.**

## 6. Estratégias geradas para a 24303 (estado atual do motor, 1 loja)

| Sev | Estratégia | Impacto/mês |
|---|---|---|
| atenção | Plano de tráfego (9,2 → 10,6 cupons/dia) | R$ 8.686 ✚ |
| atenção | Venda adicional (2,45 → 2,75 itens/cupom) | R$ 3.587 ✚ |
| atenção | Disciplina de desconto (−3 p.p.) | R$ 2.351 ✚ |
| atenção | Mix de recorrência (6,2% → 10%) | ~R$ 1.092 (indireto) |
| oportun. | Cartão presente (recarga zero) | R$ 579 ✚ |
| oportun. | Fidelidade 94,5% → ≥95% | qualitativo |
| oportun. | Cauda longa (205 SKUs) — remanejar entre lojas | sem R$ (sem estoque no relatório) |

✚ somável → **potencial total R$ 15.202/mês** para a 24303.

## 7. Pendências / próximos passos

1. ~~Carregar as 6 lojas restantes~~ **feito em 12/06/2026** — as 7 lojas estão no SEED; réguas calibram pelo canal.
2. **De-para código → nome de loja** — usuário ainda não informou qual loja é a 24303. Preencher em `CONFIG.nomesLojas` ou pela UI.
3. **Metas vazias** na planilha (`Meta receita-*`) — se o usuário exportar com metas, o parser já captura (`metaReceita`, `metaAting`, `crescimento`); falta criar regra de estratégia de gap de meta no motor.
4. **Desconto por origem** (campanha vs manual de caixa): perguntado se o RetaguardaGB exporta; se vier, criar parser + regra dedicada (é a análise mais acionável do bloco de descontos).
5. **Versão offline** com pdf.js e SheetJS embutidos (inline) — prometida após validação com os dados completos (padrão painel-vendas).
6. Esclarecer **GMV < receita líquida** (item 2 acima) se o usuário descobrir a definição do painel.
7. Possíveis evoluções: comparativo com período do ano anterior; integração conceitual com Foco Atividade (reativação de inativos como tática do "Plano de tráfego"); export CSV do ranking.

## 8. Decisões de design (racional)

- **Validação empírica em primeiro lugar**: o usuário rejeita números especulativos; todo impacto exibe a fórmula (`fonte`) e o rodapé do ERP é a verdade de referência.
- Identidade: verde-floresta profundo (#0C1612/#142420) + verde O Boticário (#34C77B) + ouro de perfumaria (#D9B25F); régua de semáforo como assinatura (vocabulário do próprio usuário); números com `tabular-nums`.
- "Cupons" na UI (= boletos do painel) para não confundir com boleto bancário.
- Impactos sempre **ceteris paribus**, declarados como ordem de grandeza para priorização — nunca como meta contratual.
- Idioma: PT-BR em toda a UI e nos textos do motor.

## 9. Correções de 12/06/2026 (auditoria contra os 7 PDFs reais + CSV)

Bugs encontrados rodando os arquivos reais da rede e corrigidos no `radar-lojas.html`:

1. **Parser ignorava rotação de página.** Os PDFs reais do RetaguardaGB saem com `rotate=90` (paisagem); `buildLines` agrupava por `transform[5]` cru e lia cada coluna como uma "linha" → 0 itens, todos os PDFs rejeitados. Fix: `pdfjsLib.Util.transform(viewport.transform, item.transform)` antes de agrupar e ordenação por `y` crescente. O PDF sintético da validação original era retrato, por isso o bug passou.
2. **CSV não era aceito** (a exportação real é `.csv`, não `.xlsx`). Fix: extensão aceita + leitura como texto com `raw:true` (ver 3.2).
3. **Selo ✓ falso sem rodapé**: `validarABC` aprovava quando `TOTAL GERAL` não era encontrado (todas as checagens eram condicionais). Fix: rodapé ausente nega o selo. Contagem de itens ≠ "ITENS DISTINTOS" do ERP virou nota (não bloqueia) quando as 4 somas fecham — quirk visto na 5905 (1272 linhas, ERP diz 1270, somas batem ao centavo).
4. **Heatmap do ranking quebrado com 1 loja**: `heat()` devolvia HTML quando `min===max` e isso era injetado em `style="background:..."` → células com `—">` espúrio no estado seed. Fix: devolve `null` e a célula sai sem cor.
5. **Cross-check novo**: chip de alerta quando `fatPDF` destoa >15% da `receitaLiq` dos indicadores da mesma loja.
6. Donut do mix agora começa no topo (offset inicial estava se cancelando — cosmético).
7. **Razão social contaminava nomes de produto**: o cabeçalho de página repete a razão social e ela não era descartada — virava `nomePre` e era prepended ao nome do 1º item de cada página (≈1 SKU por página com marca/categoria errada; as somas batiam, então o selo não acusava). Fix: `parseABC` pula linhas iguais a `meta.razao`.
8. **Nome de loja sem escape nos diags**: o campo de renomear (aba Lojas) entrava sem `esc()` nos diagnósticos das estratégias, que são interpolados como HTML — um nome com `<` injetava markup. Fix: `esc(nomeLoja(l))` no motor.
9. **Chips de arquivo duplicavam**: re-arrastar o mesmo arquivo acumulava chips contraditórios. Fix: `addFile` substitui o chip pelo nome do arquivo.

**Validação de UI (12/06/2026):** render completo em jsdom com o seed das 7 lojas — 24 checks: boot sem erro, 8 pills, ranking 7 linhas, 7 cards, troca de abas, expandir detalhe, renomear com HTML malicioso (escapado), filtro de estratégias, 3 sliders recalculando, zero erros de JS após todas as interações.

**Validação re-executada (critério permanente mantido):** 7/7 PDFs reais com selo ✓ (somas do rodapé ao centavo; 5905 com nota de contagem), CSV real com 7 lojas e valores exatos, referência da 24303 intacta (169 dias, descEtiq 26,10%, margem 62,2%, curva A=220), pipeline completo com 7 lojas sem erro.

**✓ Resolvido (13/06/2026) — loja 5905/24670:** o usuário confirmou que **5905 e 24670 são a mesma loja física (Coruripe)**; 5905 é o código antigo. A divergência vinha de comparar fontes de códigos diferentes. Decisão do usuário: **somar os dois**. Implementado via `CONFIG.mergeLojas = {'5905':'24670'}` → `lojasEfetivas()`/`combinarABC`/`combinarInd` fundem ABC (união de SKUs, soma de qtd/fat/custo/lucro) e indicadores (soma dos somáveis; ticket = GMV/boletos; itens/cupom e fidelidade ponderados por boletos). Coruripe fundida: GMV R$ 828.025 · ABC R$ 1.054.943 · 3.605 boletos. O chip de divergência é suprimido para lojas fundidas (`mergedCods.length>1`).

> Nota: mesmo somadas, ABC (R$ 1,05M) e receita líquida (R$ 828k) da Coruripe divergem ~25% — porque dentro do **código 5905** o ABC (R$ 420,7k) já era ~2x o GMV do CSV (R$ 201,6k), enquanto no 24670 ABC≈receita. O usuário viu os números combinados e optou por somar mesmo assim.

## 11. Fusão de lojas (código atual × antigo)

- `CONFIG.mergeLojas` mapeia código antigo → atual; `lojasEfetivas()` (seção "Estado e merge") consolida `state.stores` em lojas efetivas antes de `derivar`. `state.efetivas` (Map) é populado por `todasLojas()` e lido por `renderLojas` e pela estratégia de cauda (mix/curva/cauda da loja já fundida).
- Para somar outra loja no futuro: só acrescentar uma entrada em `mergeLojas` (e o nome em `nomesLojas`). `CONFIG.totalLojas` controla o "X de N" do header.

## 10. Deploy (Render, static site)

- `render.yaml` na raiz (Blueprint): o build copia `radar-lojas.html` → `public/index.html` e publica só a pasta `public/` — este MD e quaisquer dados não vão para o site. Header `X-Robots-Tag: noindex` para não indexar em buscadores.
- Repositório: https://github.com/alcinadadosti-worspace/helpLoja.git (os relatórios *.pdf/*.csv ficam fora por `.gitignore`).
- **Atenção privacidade:** a app embute o SEED com os dados reais **das 7 lojas** (decisão do usuário em 12/06/2026); a URL do Render é pública para quem a tiver. O processamento de arquivos continua 100% client-side (nada é enviado a servidor).

## 12. Aba Estratégias — visual (15/06/2026)

> ⚠ **Superada pela §13.** A aba "Estratégias" virou "Benchmark e achados"; o **waterfall (12.2) foi removido** e a parte prescritiva da régua (12.1) foi reformulada de "agora → meta" para "agora → rede". A régua `svizHTML` continua viva, só com `alvoLabel` diferente.

O usuário pediu algo mais visual: a aba era só lista de cards-relatório (badge + diagnóstico + checklist). Dois acréscimos:

### 12.1 Régua "agora → meta" por estratégia (o que o usuário pediu)

**Cada card** ganha uma régua de semáforo (a assinatura da casa) mostrando onde a loja está e onde precisa chegar naquele indicador — não só texto.

- **`gerarEstrategias`**: cada uma das 9 regras agora carrega um objeto `viz: { label, fmt, best, worst, atual, alvo, marks }`. `best`/`worst` são os extremos do gradiente (verde no `best`); reaproveitam o `SPEC` da aba Lojas onde existe (desc, itens, fid, trocas, recorr) e são inline nas demais (fluxo, cartão presente, cauda, heróis). `alvo:null` ⇒ só o pino "agora" (caso dos heróis/concentração top20).
- **`svizHTML(z)`** (logo após `regua()`): posiciona `atual` (pino branco com anel da cor do estado) e `alvo` (bandeirinha dourada) no gradiente via `clamp((v-best)/(worst-best))`; faixa branca translúcida `.sviz-span` realça a distância a percorrer; cabeçalho `agora X → meta Y` (X colorido por posição: ≤34% verde, ≤67% âmbar, senão vermelho). Helper `vfmt(v,f)` formata por tipo (`pct1`/`pct0`/`n1`/`n2`). CSS no bloco `régua "agora → meta" por estratégia`. Render injeta `${svizHTML(s.viz)}` entre `.meta-l` e `.acoes`.
- **Validação:** render no Chromium (filtro Coruripe) — 4 cards / 4 réguas, valores corretos (desconto 28,1%→25,1%, recorrência 7,4%→10%, cauda 33%→20%, itens 2,91→3,20), zero erros de console.

### 12.2 Waterfall do GMV no topo da aba (resumo, commit `c9e2e75`)

Cascata acima dos cards: *GMV hoje → +cada alavanca somável → GMV potencial*.

- **`waterfallHTML(lojas, canal, strats, filtro)`** (perto de `renderEstrategias`): monta as linhas *GMV hoje → +cada alavanca somável → GMV potencial*. Cabeçalho mostra escopo + `+% vs hoje · R$/ano`; rodapé soma o ganho **indireto** (mix de recorrência, `somavel:false`) como nota **fora da cascata**. Helper `tituloCurto(t)` corta o título no `:` ou `(` para o rótulo.
- **Escala/posição:** barras posicionadas por offset cumulativo, `left%`/`width%` sobre o `total` (= base + Σ alavancas). Base é barra neutra, incrementos coloridos por severidade (crítico=verm / atenção=âmbar / oportunidade=azul), total em ouro. Animação `wfGrow` (scaleX) respeita `prefers-reduced-motion`. CSS no bloco `WATERFALL DO GMV`; responsivo em `max-width:560px`.
- **Respeita o filtro `#fStrat`:** *Todas as lojas* → base = `canal.gmvMes`, alavancas **agrupadas por título** (mantém ~4 barras limpas em vez de 29 estratégias); *uma loja* → base = `gmvMes` da loja, cada alavanca individual.
- **`renderEstrategias` agora recebe `canal`** (assinatura `(lojas, canal, strats)`; atualizados o `onchange` e a chamada em `renderAll`). O box "Potencial identificado" passou a **refletir o escopo filtrado** (antes era sempre o total do canal).
- **Validação:** `node --check` OK; teste isolado da matemática bate ao centavo (73.103 = 57.900 + 8.686 + 3.587 + 2.351 + 579 na 24303); render real no Chromium com o seed das 7 lojas sem erros de console — canal +25,9% (R$ 1.777.885/ano), filtro Coruripe R$ 149.130 → R$ 161.097 (+8,0%).

## 13. Remodelagem: de motor de estratégias para diagnóstico (15/06/2026)

Decisão do usuário: *"ao invés de me dar estratégias, remodule a app e faça uma análise do canal"*. Aprovado: **(1) virar diagnóstico** — Estratégias→Achados (mesma detecção, reapresentada como leitura de dados, sem checklist/meta/ações), + Benchmark entre lojas, "Visão do canal"→narrativa; **(2) remover o Simulador**. Materialidade em R$ mantida (é análise, não ordem). Esta seção é o **estado atual autoritativo** (supera §12 onde divergir).

**Antes de codar, extraí os números reais das 6 lojas via Playwright** (`page.evaluate` chamando `todasLojas()/canalAgregado()/gerarEstrategias()` no app já com o seed) — a narrativa e os achados são baseados nesses dados, não em achismo. Leituras-chave: canal **R$ 571k/mês (~R$ 6,85M/ano)**; Coruripe+Palmeira = **52%** do GMV; forças uniformes (ticket 204–230, margem 62–68%*, trocas <1,5%, fid 91–97%); variância de execução (desconto 18,5–28,1%, recorrência 3,6–10,2%, fluxo 3,1–23,8 cup/dia). **A manchete "R$ 103k/mês em fluxo" do motor antigo era enganosa** — dominada por exigir que a micro-loja Sustentável fizesse ~4,6× o próprio fluxo; agora vem com ressalva ("teto aritmético, não meta").

**Pipeline preservado intacto** (parsers, selo ✓, `derivar`/`canalAgregado`/merge, SEED). Mudou só apresentação + framing:

- **3 abas** (era 4): `Diagnóstico` (era "Visão do canal"), `Lojas`, `Benchmark e achados` (era "Estratégias"). Simulador removido — tab, `view-simulador` e a função `renderSimulador` excluídos.
- **Motor (`gerarEstrategias`, nome mantido; retorna "findings")**: detecção e materialidade (`impacto`) **intactas**; só os 9 `titulo` viraram observacionais ("Desconto acima da média da rede", "Fluxo abaixo da média da rede" etc.). O render usa `diag` como leitura e **ignora** `meta`/`acoes`/`fonte` (viram dead-data no objeto, não removidos do motor).
- **`renderCanal` → narrativa do Diagnóstico** (100% computada de `lojas`/`canal`/`findings`): KPIs (sem o pill "Potencial"); bloco "Leitura do canal" (tamanho, concentração top-2, micro-loja, forças uniformes via min–max, variância de execução); e "**Gargalo nº 1 de cada loja**" = o achado de maior materialidade por loja.
- **`renderEstrategias` → Benchmark + Achados**: `matrizHTML(lojas)` (heatmap loja×indicador, movido do canal) + cards reformulados — badge de relevância (alta/média/baixa), leitura (`diag`), materialidade `~R$/mês`, linha **"Melhor da rede: {loja} {valor} · canal {valor}"** (helpers `melhorRede` + `DIMKEY` + `canalVal`), régua `svizHTML` com `alvoLabel:'rede'`, e **ressalvas** automáticas: (a) Coruripe — indicadores de ABC (cauda/recorrência) marcados pelo merge 5905+24670; (b) micro-loja — materialidade de fluxo é teto aritmético.
- **Removido do render**: `waterfallHTML`/`tituloCurto`, box "Potencial identificado", checklist `.acoes`, simulador. CSS desses elementos ficou no arquivo (dead, inócuo). `resumoSlack` reframado para "Principais achados" (sem "potencial total").
- **`svizHTML`** ganhou `z.alvoLabel` (default `'meta'`; achados passam `'rede'`).
- **Aba Lojas**: ressalva de merge (Coruripe) no topo do card.

**Validação (critério permanente mantido):** `node --check` sem erro; render no Chromium com o seed das 7 lojas — 3 abas, 8 pills, narrativa (4 §), 6 gargalos, matriz 6 linhas, 29 achados, 24 linhas de benchmark, 3 ressalvas, **0 checklists**, 1 ressalva-merge na aba Lojas, **zero erros de console**. Selo ✓ dos PDFs e pipeline de dados inalterados.

**Constraint de fundo:** não há dados do ano anterior (YoY) — o eixo de comparação é **peer** (loja vs melhor loja da própria rede), que substitui o tempo. Ver memória `radar-lojas-sem-yoy`.

## 14. Aba Análise + contexto de equipe (15/06/2026)

Pedido: *"use de estratégia muito bem elaborada pra descobrir onde estamos indo ruim e onde podemos melhorar"* → esclarecido como **análise visual bem estruturada** (no rigor de Pareto/Gini). Nova aba **Análise** (`view-analise`, entre Diagnóstico e Lojas) — **4 abas agora**. 5 frameworks, todos computados client-side dos dados já parseados:

1. **Concentração do sortimento** — `gini()` do faturamento por SKU + `pareto80()` + `lorenzSVG()` (curva de Lorenz) por loja; veredito por `caudaPct`. Mede dependência de heróis × cauda morta. Ginis reais: 0,57–0,73.
2. **Concentração da rede** — `gini()` do GMV entre as 6 lojas + Pareto (barras + acumulado). Top-2 = 52%.
3. **Consistência de execução** — `cvar()` (coef. de variação) por indicador entre lojas, ordenado. Alto = disperso (oportunidade de alinhar); baixo = alinhado. Revela: ticket/margem/trocas consistentes; desconto/recorrência/fluxo dispersos.
4. **Fronteira de oportunidade** (peer/DEA-like) — cada loja vs a melhor da rede: desconto (front. Palmeira 18,5%) R$ 40,6k, cesta (front. Coruripe 2,91) R$ 37,4k, fluxo (exclui micro) R$ 35,1k, recorrência (indireto) R$ 9,4k. **Somável direto R$ 113k/mês = R$ 1,36M/ano = +19,8%**. Callout do **espelho Coruripe↔Palmeira** (cada uma é a melhor no que a outra é a pior; `melhorRede` sobre lojas não-micro — micro empatava e quebrava a detecção).
5. **Movimentação de equipe (contexto)** — const `MOVIMENTACOES`. Por loja: chips papel+data, nível de exposição (gerente ou ≥2 saídas cedo = alta; só tardia = baixa).

**Privacidade (decisão):** o usuário passou **nomes** de demitidos/transferidos. Como o app é **deployado público** (Render), **nomes NÃO foram embutidos** — `MOVIMENTACOES` guarda só `loja`/`data`/`tipo`/`cargo` (anonimizado). Sinal analítico preservado sem expor PII de terceiros (LGPD). Nomes ficaram na conversa + memória `radar-lojas-movimentacao-equipe`.

**Limitação honesta (dita na própria UI):** dados são **agregados do período** (1 nº/loja) — dá para *sinalizar* troca de equipe, não *isolar* o impacto. Para medir, precisa de vendas **por mês** (pré/pós cada data). **Hipótese forte:** Palmeira perdeu a **gerente em 02/02** e é a pior da rede em **cesta e identificação** — os indicadores que um gerente cobra; pode ser transitório (ramp-up), não estrutural.

**Helpers novos:** `gini`, `cvar`, `pareto80`, `lorenzSVG`, `renderAnalise`, const `MOVIMENTACOES`. CSS: bloco "ANÁLISE". **Validação:** `node --check` OK; Chromium (seed 7 lojas) — 5 seções, 6 Gini+Lorenz, 18 barras, espelho, 9 chips de equipe, zero erros de console.

## 15. Aba Sortimento — SKU cruzado entre lojas (15/06/2026)

Pedido aprovado: a maior análise possível **sem dados novos**, usando o per-SKU do ABC (cada item tem `sku`/`custo`/`lucro`/`prc`/`curva`/`marca` — `combinarABC` re-roda `analisarABC`, então a Coruripe fundida também). Nova aba **Sortimento** (entre Análise e Lojas) — **5 abas agora**. `crossSKU(lojas)` monta `Map(sku → {nome, marca, st:{loja:{fat,qtd,lucro,prc,curva}}, totFat/totQtd/totLucro})`. 3 blocos:

1. **Lacunas de distribuição** (seletor `#fGap`, estado `gapSel`) — SKUs **curva A (≥ R$ 500) em outra loja** mas **ausentes ou < R$ 100** na loja selecionada. Potencial est. = fat na loja-herói × (GMV da loja ÷ GMV da herói), mediana entre heróis. Coruripe: 41 lacunas ~R$ 70k; **Palmeira: 76 lacunas ~R$ 114k** (coerente com o buraco de gerência).
2. **Margem × giro** — matriz tipo BCG: cada SKU por giro (qtd vs mediana) × margem (vs `canal.margem`). 4 quadrantes (estrela / margem-vaza / jóia / cauda) com top SKUs. **Exclui itens de margem ≥ 98,5%** (brindes/custo-zero, que apareciam como "jóias" de R$ 44); jóias ordenadas por faturamento.
3. **Mesmo SKU, preço diferente** — SKUs em ≥ 2 lojas (qtd ≥ 5 em cada) com spread do **preço médio realizado** (`prc` = fat/qtd, pós-desconto) ≥ 15%, ordenado por spread × fat.

**Helpers:** `crossSKU`, `renderSortimento`, `let gapSel`. CSS: bloco "SORTIMENTO" (`.sku-tbl`, `.skuchip`, `.quad`). `#fGap` re-renderiza direto (select, sem corrida de blur do rename).

**Validação:** `node --check` OK; Chromium — 3 seções, 15 lacunas, 4 quadrantes, 12 preços, seletor troca loja; bugcheck completo (5 abas + rename + XSS) **zero erros**.

**PRM resolvido (15/06/2026):** o usuário confirmou **PRM = boleto promocional** (mecânica de campanha, não sortimento fixo). `crossSKU` agora **exclui itens com `\bPRM\b` no nome** das 3 análises do Sortimento — uma "lacuna" de PRM seria execução de campanha (a promo rodou numa loja e não na outra), não buraco de prateleira; e preço/margem de PRM é promocional, distorce a matriz. Lacunas após excluir PRM: Coruripe **37 (~R$ 54k)**, Palmeira **71 (~R$ 103k)**. (O parser principal segue tratando PRM como brinde em `a.brindes` — coerente.) Possível evolução futura: uma visão dedicada de **execução de campanha** (quais PRM rodaram em cada loja).

**Upgrade visual + analítico (15/06/2026):** a aba Sortimento ganhou — (a) **heatmap de cobertura** (bloco 0): top 26 SKUs × loja, célula verde/âmbar/vermelho por status (herói curva A / vende fraco / parado-ausente); (b) **scatter margem × giro** (`scatterSVG`): cada SKU um ponto (x = giro em log, y = margem, raio ∝ faturamento, cor = quadrante), com linhas de referência e `<title>` por ponto — cards de quadrante seguem abaixo; (c) **barra de amplitude de preço** (`rangeBarSVG`); (d) **estimativa de lacuna normalizada por tráfego** — penetração (un/cupom) da loja-herói × cupons desta loja × preço (usa `l.boletos`), em vez de escalar por GMV; (e) **roll-up das lacunas por marca**. Helpers: `scatterSVG`, `rangeBarSVG`.

## 16. Aba Lucro + correlação (15/06/2026)

Pedido: construir as **4 análises novas** propostas (lente de lucro, mix por categoria, bridge de drivers, correlação) — usuário pediu todas. Nova aba **Lucro** (após Sortimento) → **6 abas**. `aggProfit(lojas)` agrega fat/lucro/custo por loja / categoria / marca / cat×loja dos `abc.its` (exclui PRM). 8 blocos:

1. **Onde o lucro nasce** — lucro bruto total (~R$ 2,1M*), barras por loja/categoria/marca com margem%, + callout de divergência receita×lucro (categoria que mais muda de rank entre faturamento e lucro).
2. **Lucro por cupom** — lucro ÷ boletos por loja vs canal (qualidade da transação).
3. **Bridge de drivers** (seletor `#fBridge`, estado `bridgeSel`) — decompõe o GMV da loja vs a **loja-média do canal** em **cupons × itens/cupom × valor-do-item**. *Math:* decomposição sequencial; **`effVmi` usa `bItens` (não `canalItens`)** para absorver a interação cesta×valor — senão sobra um termo e não fecha. **Verificado: soma exata do total nas 6 lojas.** Tudo via CSV (sem asterisco do ABC).
4. **Mix por categoria cross-store** — heatmap loja × categoria; célula = % da categoria no faturamento da loja; cor = **índice vs canal** (verde ≥ 1,20× super-indexa / vermelho ≤ 0,80× subindexa).
5. **Concentração e risco de lucro** (add. da reinspeção) — Gini/Pareto sobre o **lucro** por SKU vs receita (via `crossSKU`, PRM fora): 80% do lucro vem de ~17% dos SKUs (≈ igual à receita 18% → **baixo risco de concentração de lucro**), top-20 = 20% do lucro; lista de motores de lucro + Gini lucro vs receita por loja. Nuance: **ARBO** (campeã de "margem vaza") é o **#2 motor de lucro absoluto** — por isso a lente de lucro importa.
6. **Gap de categoria** (add. da reinspeção) — por loja, soma das categorias onde fica **abaixo do peso do canal** × faturamento da loja = tamanho do desequilíbrio de mix (realocação, **não receita nova**; referência, não meta). Coruripe lidera (R$ 45,7k, falta Kits & Estojos) — **corrobora a matriz de datas** (subcaptura kits).
7. **Decomposição de margem: mix × taxa** (deep) — quebra o Δmargem de cada loja vs canal em **mix** (Σ(w_loja−w_canal)·margem_canal_categoria) + **taxa** (Σ w_loja·(margem_loja_cat−margem_canal_cat)); mix+taxa = Δ exato (verificado nas 6). **Achado forte:** o efeito **mix é ínfimo (≤0,2 pp) em todas** e a **taxa domina** → as lojas vendem o mesmo blend; a margem difere por **como precificam/compram dentro das categorias**, não pelo que vendem. Logo margem = disciplina de preço/desconto/custo, não sortimento (amarra com o desconto).
8. **Estrutura de custo** (deep) — stacked bar custo × margem bruta por categoria. **Descoberta de dado:** o **"Lucro" do ABC = faturamento − custo (margem bruta), SEM linha de imposto** neste export (imposto = fat−custo−lucro deu 0). Bloco reformulado de "contribuição custo/imposto/lucro" para "custo × margem" + nota; `stackBar` mostra resíduo vermelho só se um export futuro trouxer imposto. `aggProfit` passou a somar `custo`.

**Bloco 7 da aba Análise — Datas comemorativas (add. da reinspeção):** usa `s.abc.kits` (kits NAT/MAES/AMOR/PAIS/CRIANC por data, antes só mostrado por loja). Heatmap data × loja, célula = % do faturamento da loja na data, cor = índice vs canal. Total kits **R$ 319k (9,3%)**; janela cobre Natal/Mães/Namorados. Achado: **Coruripe subcaptura Mães (2,7% vs 4,6%) e Namorados** — vive de perfumaria, deixa data na mesa. **Fix cosmético:** helper `lojaCurta()` desambigua cabeçalhos de heatmap (Palmeira × Palmeira Sustentável, que truncavam igual) — aplicado nos 3 heatmaps + kits.

**Correlação (bloco 6 da aba Análise):** `pearson()`, matriz 7×7 entre indicadores das lojas (fluxo/ticket/cesta/desconto/fidelidade/recorrência/margem), cor por sinal×força. Callout: **desconto × fluxo = 0,18** (baixo → desconto NÃO compra movimento) vs **desconto × cesta = 0,80** (parece comprar cesta). Caveat forte: **n=6**, a micro-loja (outlier) pesa, confirmar com dado mensal.

**Achado do CSV (releitura):** a coluna **"Crescimento ano anterior" existe mas está vazia** — o ERP exporta YoY, só não veio populado neste arquivo. Um export populado destrava tendência.

**Helpers novos:** `aggProfit`, `renderLucro`, `pearson`, const `bridgeSel`. CSS: `.lucro-cols`, `.corr`. **Validação:** `node --check` OK; Chromium — bridge fecha exato nas 6 lojas, correlação 49 células, bugcheck completo (6 abas + rename + XSS) zero erros de console.

## 17. Aba Apresentação — capa + plano (16/06/2026)

O usuário vai **apresentar ao vivo pela app, para os gerentes de loja, com o objetivo de mostrar a oportunidade e aprovar um plano**. Nova aba **Apresentação** (1ª, `data-v="apresentacao"`, **abre por padrão** — `on` movido de canal para ela) → **7 abas**.

`renderApresentacao(lojas, canal, findings)`: recomputa a fronteira de oportunidade (opDesc+opItens+opFlux realista, ~R$ 113k/mês = **R$ 1,36M/ano, +20%**) e o espelho (cesta/desconto champ entre lojas não-micro). Layout estilo slide (CSS bloco `APRESENTAÇÃO`):
- **Hero** com a mensagem central ("vende bem, falta execução; a rede prova R$ X/ano").
- **3 KPIs grandes**: faturamento/ano, oportunidade/ano, +% sobre GMV.
- **O plano — 4 alavancas**, cada uma com a loja que prova: (1) desconto (Palmeira 18,5%), (2) cesta/2º item (Coruripe 2,91), (3) capturar datas/kits (R$ 319k), (4) campeão em todas (lacunas). Tom de gerente (operacional, acionável).
- Callout do **espelho** (Coruripe↔Palmeira se ensinam — "sem consultoria").

Tudo computado dos dados reais (atualiza se reupar). **Validação:** Chromium — boot abre na Apresentação, hero/3 KPIs/4 cards, inspeção completa (7 abas + 3 seletores + rename/XSS) zero erros.

**Roteiro de fala (gerentes, ao vivo):** abrir na capa → Benchmark (cada um vs a melhor da rede) → Lucro/mix×taxa (margem é execução) → Análise/fronteira (R$ provados + espelho) → Sortimento (o concreto por loja) → voltar à capa pro plano/aprovação.

## 18. Aba Categorias — taxonomia oficial do usuário (16/06/2026)

Início de uma remodelagem maior. O usuário passou a **taxonomia oficial dele** (15 categorias × palavras-chave) — mais granular que o `categoriaDe` heurístico antigo. Nova aba **Categorias** (após Sortimento) → **8 abas**.

**Classificador `catOB(nome)`** (const `CAT_OB_SPEC` + lista global `_KWOB`): regra **"palavra mais específica (mais longa) ganha"** entre TODAS as categorias (não só dentro de uma) — porque os conflitos eram cross-categoria (uma frase específica numa categoria de baixa prioridade perdia para um token curto numa de alta). Match: frase (com espaço) = substring de tokens consecutivos; token único = igualdade exata (evita "PR" casar com PRIMER etc.). Empate de tamanho → ordem da tabela. Adicionei algumas chaves óbvias pros "Outros" (COLONIA, SERUM/SRUM, CAIXA, LAP, CR, ESPUM BANHO, CR PENT) — "Outros" ficou ~1%.

**Decisão de negócio (confirmada pelo usuário):** **"DES COL" (desodorante colônia) = Perfumaria** — porque é o produto de perfume do dia a dia; "Desodorantes" fica só com antitranspirante (roll-on/aerossol). Isso vale **~60% do faturamento** (Perfumaria 60%, Cuidados com a Pele 11%, Gifts 11%, top-3 = 83%). **Validei a classificação rodando nos 1.908 SKUs reais ANTES de construir a UI** (passo essencial — a classificação errada quebraria tudo).

3 blocos (`aggCat` agrega fat/custo/lucro/SKUs por categoria e cat×loja, via `catOB`):
1. **Concentração por categoria** — Gini (deu **0,76** — muito concentrado em Perfumaria) + Pareto (barras por categoria).
2. **Mix por loja** — heatmap categoria × loja, índice vs canal (verde super / vermelho subindexa).
3. **Margem por categoria** — stacked custo × margem bruta por categoria.

**Validação:** `node --check` OK; Chromium — 3 seções, Gini 0,76, 14 barras, heatmap 10 linhas; inspeção (8 abas + 3 seletores + rename/XSS) zero erros. **Próximo:** o usuário sinalizou continuar a remodelagem — `catOB` pode futuramente substituir o `categoriaDe` heurístico em todo o app (Lucro, Diagnóstico recorrência, etc.).

## 19. catOB unificado em todo o app (16/06/2026)

O usuário notou a inconsistência: a aba Categorias usava `catOB` (taxonomia oficial) mas o resto (Lucro, donut da aba Lojas, recorrência) ainda usava o `categoriaDe` heurístico — "Perfumaria" significava coisas diferentes em abas diferentes. **Unificado em 3 trocas no `analisarABC`:**
1. `cat: categoriaDe(...)` → **`cat: catOB(...)`** — propaga a taxonomia oficial para tudo que lê `it.cat` (Lucro: pools/mix/gap/mix×taxa/estrutura de custo; donut da aba Lojas; crossSKU).
2. **Recorrência** remapeada: `['Maquiagem','Skincare','Capilar']` → `['Maquiagem','Cuidados Faciais','Cabelos']` (categorias de recompra rápida na taxonomia nova). Canal: 6,9% → **7,7%**.
3. **`precoItemCB`** (preço-base do "2º item" no cálculo de cesta): `byCat['Corpo & Banho']` → `byCat['Cuidados com a Pele']` (≈ R$ 34,35).

`categoriaDe` virou **código morto** (mantido, não chamado). Ordem OK: `catOB`/`_KWOB` (const ~linha 1300) são inicializados antes do `analisarABC` rodar (no `initSeed`, no fim do script). **Verificado:** Categorias e Lucro agora mostram a MESMA categoria top (Perfumaria) — `CONSISTENTE: true`; seed carrega com selo ✓; inspeção das 8 abas + seletores + rename/XSS zero erros. O selo do rodapé é imune (valida fat/custo/lucro/qtd, não categoria). Label "(make·skin·cap)" da recorrência ficou (ainda razoável: Maquiagem+Cuidados Faciais+Cabelos).

## 20. Remodelagem: 3 ajustes (16/06/2026)

Usuário pediu os 3 próximos passos de uma vez:

1. **Limpeza** — `categoriaDe` (heurístico, já morto desde §19) **removido**. "Outros" da classificação `catOB` caiu para **~1,0%** (chaves óbvias já tinham sido adicionadas em §18).
2. **Recorrência configurável** — nova `CONFIG.catRecorrencia` (default `['Maquiagem','Cuidados Faciais','Cabelos']`). **Decisão informada por dado:** testei incluir 'Cuidados com a Pele' + 'Sabonete Corpo' (a pergunta do usuário) → recorrência vai a **~22% uniforme em todas as lojas** (Penedo 25, Palmeira 21…), **matando o poder de discriminar**. Mantida a definição **estreita** (recompra de alto valor, ~7,7%, com Penedo destacando 10%). Régua `SPEC.recorr` inalterada. `analisarABC` usa `CONFIG.catRecorrencia.reduce(...)`.
3. **Marca × categoria** (novo bloco 4 da aba Categorias) — `aggCat` agora também agrega `catMarca[cat][marca]`. Por categoria: top 3 marcas + fatia da líder (vermelho ≥45% = dependência). **Achado:** categorias grandes (Perfumaria líder 13%, Gifts 21%, Desodorantes 23%) são diversificadas; as **de cuidado dependem de uma marca só** — Maquiagem **77% Make B**, Cuidados Faciais **74% Botik**, Cabelos **57% Match**, Sabonete **56% Cuide-se Bem**. Risco de concentração de marca + lacuna de marca.

**Validação:** `node --check` OK; Chromium — Categorias 4 seções, marca×cat 10 linhas; consistência Categorias↔Lucro mantida; inspeção (8 abas + 3 seletores + rename/XSS) **zero erros**.

## 21. Lacuna de marca por loja (16/06/2026)

Bloco 5 da aba Categorias (`aggCat` passou a agregar também `marcaTot` e `marcaLoja`). Para cada marca forte da rede, mostra em quais lojas ela é **subdesenvolvida** (índice da participação da marca na loja vs no canal). Heatmap marca × loja (top 14 marcas, cor por índice) + lista das maiores lacunas em R$ (potencial = trazer a marca ao peso do canal × faturamento da loja). `brandCat` = categoria principal de cada marca (via `catMarca`). Limiar de lacuna: índice < 0,6×.

**Achado:** poucas lacunas grandes (as lojas têm mix de marca parecido — mesmo catálogo), mas a **Palmeira** concentra as maiores: subdesenvolve **Botik** (líder de Cuidados Faciais, ~R$ 10,5k) e **Quasar** (perfumaria, ~R$ 9,5k) — coerente com a fraqueza de cesta/recorrência dela (e com o buraco de gerência). **Validação:** Categorias 5 seções, heatmap 14 marcas, inspeção (8 abas + seletores + rename/XSS) zero erros.

## 22. Resumo de Performance: YoY + meta PEF + aba Equipe (23/06/2026)

Nova fonte de dados: **`20260623_Resumo_de_Performance_Indicadores_Loja_*.xlsx`** (4 abas: FILTROS, CP, PDV, CONSULTOR). É a virada mais importante desde o início — **agora há ano anterior (YoY)**, derrubando o pressuposto "sem YoY, só peer" das §13/§14 (memória `radar-lojas-sem-yoy` fica **parcialmente superada**: o peer continua válido, mas o tempo voltou ao eixo). Período: **26/12/2025–14/06/2026** (2 dias a mais que o ABC/CSV, ends 14/06 vs 12/06) vs anterior **26/12/2024–12/06/2025**.

**O que o arquivo traz** (20 indicadores PDV; 16 no CONSULTOR): Receita, Boletos, Boleto Médio, Itens, Itens/Boleto, Preço Médio, Pen. Boleto Turbinado, Pen. Boleto **Promocional (PRM)**, Share BT/BP, Pen. Mobshop, Pen. Boletos **B1**, Pen. Boletos **Fidelidade**, **Resgate Fidelidade**, Conversão de Ação de Fluxo, Pen. **Cuidados Faciais**, **% Boletos ID Cliente**, Qtd Serviços em Loja, Clique & Retire, Loja Digital (% atendimento + TME 1ª resp.). Cada um com **Vs. Meta PEF** e **Vs. Período Anterior** (PDV) / **Vs. Ano Passado** (CP).

**Achados-manchete (reais, do arquivo):** canal **−17,6% receita / −24,8% cupons** YoY (queda de **fluxo**, não de ticket — boleto médio **+9,5%**); vs meta PEF **−14,6%**; **só a 24303 bate a meta de receita (+6,1%)** (24668 −6,7%, 24670 −20,1%, 24669 −24,9%, **24671 −26,6%** pior). **Mesmas-lojas ≈ −20%**: o −17,6% do canal é **diluído** porque **24303 e 24617 não têm base do ano anterior no sistema** (recodificadas/novas). Coruripe já vem **consolidada em 24670** no arquivo (sem 5905) → casa com `mergeLojas`.

**Reconciliação:** PDV soma **exato** ao CP (R$ 3.812.066,26 · 17.805 boletos). **CONSULTOR é crédito BRUTO** (soma > total da loja: 24671 +40%, 24668 +32%; total 21.665 vs 17.805 boletos) — um boleto creditado a +1 consultora (venda compartilhada). Logo ranking entre consultoras vale; "participação" é **entre consultoras creditadas**, não fatia exata da loja (dito na UI).

**Arquitetura (camada paralela, não mexe no motor ABC/CSV):**
- **`PERF_SEED`** — const JSON inserido logo após `const SEED` (gerado por `gen-perf-seed.js`/SheetJS; ~37KB). Estrutura `{meta, ind, cp:{k:{meta,real,vsMeta,vsAnt}}, pdv:{cod:{k:{v,vsMeta,vsAnt}}}, consultor:[{nome,pdv,vals:{k:{v,vsAnt}}}]}`.
- **`PERF_IND`** (const, fonte única de rótulo/escala/direção: brl|brl2|int|n2|pct|time) + `PERF_LABEL/SCALE/DIR`, `perfFmt(key,v)`.
- **`parsePerf(wb)`** + `isPerfWB(wb)` (detecta abas PDV+CONSULTOR). Espelha o gerador (sheet-aware). `handleFiles` separou CSV (antigo "Listar Por Lojas") de XLSX: XLSX → `isPerfWB`? parsePerf:parseIndicadores. **Re-upload valida: `parsePerf` reproduz o `PERF_SEED` célula a célula** (testado em Node com o XLSX real).
- **`state.perf`** (global; seedado no `initSeed`). `derivar` expõe **`d.perf`** = `state.perf.pdv[s.codigo]` (sem merge — arquivo já tem código atual). Canal lê `state.perf.cp`.

**Normalização (cuidado):** valores pct → **0-100** com escala **por aba** (NÃO por magnitude — Cuidados Faciais 2,78% viraria 278 com heurística): PDV/CONSULTOR penetrações são frações ×100; `resgateFid` e (no CONSULTOR) `idCliente` já são 0-100; CP já é 0-100. **Deltas:** PDV/CONSULTOR são **frações** (×100 = **%** p/ monetário/volume, **pp** p/ penetração); **CP vsAnt monetário vem em pp% (pct100)** → dividir por 100 (feito só p/ receita/boletos/ticket). `keyOf` dobra `ª/º` (o ordinal "1ª" sobrevive ao `norm`). **`% Boletos ID Cliente` é RBV e instável**: estoura >100% por loja (93–131%) e explode por consultora (ex.: 1.343%); CP vsAnt = −1003,4 (artefato). **Decisão: fora dos views de consultor** (block 3 usa Fidelidade no lugar; coaching usa itens/boleto+PRM+Fidelidade); mostrada só por loja (aba Lojas) com ressalva; deltas absurdos (|·|>600%) viram "—".

**Mudanças de UI (9 abas agora; Equipe entre Lojas e Benchmark):**
- **Aba Equipe** (`renderEquipe`, `view-equipe`): (1) contexto + ressalva de crédito bruto; (2) ranking da rede por nome (Gini 0,52 — concentração média) + multi-loja (9 consultoras); (3) ranking por loja (receita/part./boletos/ticket/itens/PRM/Fidelidade + Gini por loja); (4) **gap de execução intra-loja** (melhor×pior na mesma loja, ≥50 boletos, itens/PRM/Fidelidade = sinal de coaching).
- **Diagnóstico** (`renderCanal`): banda de **delta-pills** YoY/meta no topo + parágrafo **"tendência"** liderando a leitura (canal encolhendo, fluxo; mesmas-lojas −20%; meta).
- **Lojas** (`renderLojas`): **scorecard PEF por loja** (`perfPanel`) — 15 indicadores × atual/vs meta/vs ano ant.
- **Benchmark** (`renderEstrategias`): heatmap **`perfMatrizHTML`** (penetrações por loja, cor relativa + linhas receita vs meta/ano por sinal); nota corrigida (não é mais "sem ano anterior").
- **Apresentação** (`renderApresentacao`): hero reformulado para abrir pela **queda YoY** (urgência) + KPI "vs ano passado" e gap em R$/ano; mantém o plano de 4 alavancas e o espelho. CSS: só `.tag.bad` adicionado.

**Privacidade — MUDANÇA de decisão:** o usuário **autorizou expor os 45 nomes reais** de consultoras no seed ("pode expor, será uma reunião privada"), **contrariando** a regra das §10/§14 (PII fora do deploy). **Ressalva registrada:** a URL do Render é pública — os nomes ficam acessíveis lá também, não só na reunião. Toggle de anonimização no deploy não foi feito (oferecido).

**Validação (critério permanente mantido):** `node --check` OK; `parsePerf` == `PERF_SEED` célula a célula no XLSX real; jsdom (seed) — 9 abas trocam, Equipe 4 seções/9 tabelas, 6 scorecards PEF, matriz PEF no Benchmark, 3 seletores, rename/XSS escapado, **zero erros de console**. Pipeline ABC/CSV (selo ✓, mix, Gini, Lucro) intacto (views inalteradas no boot).

**Commit:** `fcb122c` na main (push feito). **Bug-fix posterior** (mesma sessão, antes do §23): a) mesma-loja YoY estava como média ponderada das taxas (−20,4%) → trocado por agregado real Σatual/Σant−1 (**−20,7%**); b) `gapAno` da Apresentação misturava base (rate de performance × GMV do CSV) → recalculado sobre a receita de performance anualizada pelos dias do período (**~R$ 1,74M/ano**), e "mesmo tamanho" → "mesma ordem de grandeza"; c) hero da Apresentação guardado contra YoY ausente (`temYoY`).

## 23. Aba Tendência — ponte YoY + funil de fidelização + alavancas de fluxo (23/06/2026)

Pedido do usuário ("tendo esses dados ricos, quais análises podem ser feitas" → "pode fazer"). Nova aba **Tendência** (`view-tendencia`, entre Diagnóstico e Análise) → **10 abas**. `renderTendencia(lojas, canal)`, 100% de `state.perf`. CSS novo: blocos `.ebar` (barra divergente centrada no zero) e `.funnel`.

**Insight central (verificado, soma exata):** Receita = boletos × itens/boleto × preço médio. A queda do canal **−17,6% YoY decompõe em: tráfego −24,8%, cesta −0,4% (≈flat), preço +10,0%** → em R$ no período: **tráfego −R$1.147k, cesta −R$15k, preço +R$345k = −R$817k**. **É 100% fluxo** (gente entrando menos), segurado por preço. Reframe definitivo: recuperação = recuperar tráfego.

3 blocos:
1. **Ponte da queda** (`bridge()` — decomposição sequencial exata vol/cesta/preço; soma = ΔR conferida nas 6). Barras divergentes do canal + tabela por loja (só as 4 com base; 24303/24617 sem ano anterior fora). Per-loja a receita é reconstruída por boletos×itens×preço (≈0,5% de ruído de arredondamento vs receita.v; a coluna YoY% usa o `receita.vsAnt` autoritativo).
2. **Funil de fidelização** — Identifica (`idCliente` 116%, RBV >100) → Cadastra (`penFid` 95,6%) → **Resgata (`resgateFid` 54,3%)**. Vaza no resgate (alavanca de retorno = fluxo barato). Resgate YoY +13,1pp. Heatmap de resgate por loja (24303 pior, 39%, apesar de bater meta — vende mas não traz de volta).
3. **Alavancas de fluxo** — Conversão de Ação de Fluxo (canal −5,6pp YoY, na direção dos boletos) + Serviços em Loja (contagem absoluta + YoY), com **Boletos YoY** ao lado pra associação loja-a-loja. Caveat n=6, sem série mensal: sinaliza, não prova.

**Validação:** `node --check` OK; jsdom — 10 abas, bridge soma exata (canal e por loja), funil 116/95,6/54,3, zero erros de console. Commit `476a487`.

## 24. Corte para 5 abas — foco em "por que o canal cai" (24/06/2026)

Usuário: *"essa app está muito poluída visualmente, remova tudo que for garbage e deixe só o que realmente interessa para uma análise do que porque meu canal loja está indo mal."* Delegou a priorização a mim. Critério: **manter só o que responde "por que o canal cai"**; cortar o que responde outra pergunta (sortimento/lucro/mix).

**De 10 → 5 abas. Mantidas:** Apresentação, Diagnóstico, Tendência, Lojas, Benchmark e achados. **Removidas:** Análise, Sortimento, Categorias, Lucro, Equipe.

**Racional do corte** (apoiado na análise profunda de 23-24/06 — a queda é **tráfego −25%**, mascarada por preço +10%): Lucro (margem 62% é saudável, não é a causa), Categorias (mix estável, não explica), Sortimento ("o que estocar", outra pergunta), Análise (frameworks acadêmicos demais), Equipe (ângulo de execução; a queda é de tráfego de rede, não de vendedora — é a "primeira a voltar" se sentir falta).

**Execução (script `cut.js`, por âncoras de conteúdo):** removidas 731 linhas (2408→1677). Duas faixas contíguas — Range 1 `aggCat`+`renderCategorias`; Range 2 `gini`…`renderEquipe` (helpers exclusivos + 4 render fns). `renderApresentacao` (keeper) ficava entre as duas. Removidos também 5 botões nav, 5 `<section>` e 5 chamadas no `renderAll`. **Helpers exclusivos removidos:** `gini, cvar, pearson, lorenzSVG, pareto80, scatterSVG, rangeBarSVG, crossSKU, aggCat, aggProfit, MOVIMENTACOES, gapSel, bridgeSel`. **Mantidos (compartilhados):** `catOB`/`marcaDe` (core de `analisarABC`), `heatCell`, `donutSVG`, `melhorRede`/`DIMKEY`/`canalVal`, `matrizHTML`, `svizHTML`/`SPEC`/`regua`, `perfFmt`/`perfPanel`/`perfMatrizHTML`, `resumoSlack`, `gerarEstrategias`.

**Fix de referência pendurada:** card 4 da Apresentação citava "a aba Sortimento lista…" → reescrito sem citar a aba removida.

**CSS morto:** as regras dos blocos removidos (`.hbar/.hbars`, `.stack`, `.gini-grid`, `.quad`, `.corr`, `.skuchip`, `.cell/.he/.ve/.mo/.au`, scatter/lorenz) ficaram **inertes** (0 uso confirmado; invisíveis no render). Decisão: **não remover** — risco de quebrar estilo de sobrevivente por ganho invisível; scrub opcional num passe dedicado.

**§§14–21 (Análise/Sortimento/Categorias/Lucro/Equipe) ficam como histórico** — as features saíram da UI em 24/06; código no histórico git (commit anterior `476a487`), recuperável.

**Validação:** `node --check` OK; jsdom — **5 abas** (apresentacao/canal/tendencia/lojas/estrategias), troca de abas, expand, rename/XSS escapado, `#fStrat` ok, **zero erros de console**. Pipeline ABC/derivar/perf intacto. Arquivo 2408→1676 linhas. Commit `d341f91`.

## 25. Remove Apresentação + abas mais visuais / menos poluídas (24/06/2026)

Usuário: *"remova a aba de apresentação e deixe as outras abas mais visuais, ainda está muito poluída."* → **4 abas**: Diagnóstico (agora **default**, `class on`), Tendência, Lojas, Benchmark.

- **Apresentação removida** — nav + view + `renderApresentacao` (65 linhas) + chamada no `renderAll`. `melhorRede` mantido (usado pelo Benchmark). Default movido p/ `canal`.
- **Diagnóstico (`renderCanal`) menos texto** — narrativa de ~5 parágrafos → só o `tendPara` (tendência YoY + meta, 2 linhas com nº em cor). Cortados "O canal faz…/forte e uniforme/onde a execução varia/aster". Pills (YoY+KPI) e tabela "Gargalo nº 1" mantidos. (As computações `ext/eDesc/eRec/eFlux/eTick/eFid/eTroca/top2/micro/merged/margLo/margHi` ficaram **inertes** — código morto invisível, não removido p/ não arriscar; scrub opcional.)
- **Lojas (`renderLojas`) cards leves** — o `perfPanel` (scorecard PEF de 15 linhas) saiu do corpo do card e foi pro **detalhe expansível** (`${perfPanel}${det}`), junto com mix/ABC/SKUs. Card default = KPIs + réguas (visual). Botão: "ver desempenho PEF, mix e curva ABC".
- **Benchmark (`renderEstrategias`) achados compactos** — removidos o parágrafo `diag` e a linha `bench-l` de cada card; sobra **badge (sev+loja) + título + régua `svizHTML` (o visual) + R$**. 28 réguas, 2 heatmaps (matriz + perfMatriz) mantidos como centro visual. (`dk/bench/cv/benchL` inertes.)
- **Tendência mantida** — já é a aba mais visual (barras divergentes + funil + tabela); leads são a explicação do "porquê" com nº em negrito/cor.

**CSS morto acumulado** (abas removidas + `.diag`/`.bench-l`): inerte, invisível. Scrub dedicado pendente se quiser arquivo limpo por dentro.

**Validação:** `node --check` OK; jsdom — 4 abas (default Diagnóstico), troca de abas, expand mostra PEF, rename/XSS escapado, `#fStrat` ok, **zero erros de console**. Tamanhos: canal 5945→4808, estrategias 48524→40957. Pipeline ABC/derivar/perf intacto. Commit `493e31b`.

## 26. Benchmark filtrado + scrub de código morto (24/06/2026)

Dois ajustes que o usuário pediu (após o §25):
1. **Benchmark abre filtrado na maior loja** — `filtro` default = `lojas[0].codigo` (antes `'all'`). Default mostra ~3 achados em vez de 28; os 2 heatmaps acima dão a visão de rede; o seletor tem "Todas as lojas". `estrategias` no boot: 40957→17376.
2. **Scrub de código morto** (orfanado pelos cortes §24/§25):
   - **JS:** `renderCanal` perdeu `tot/media/top2/top2pct/micro/merged/ext/eDesc/eRec/eFlux/eTick/eFid/eTroca/margN/margLo/margHi` (só `byGmv` + `gargalo` ficaram). `renderEstrategias` perdeu `dk/bench/cv/benchL`. Funções órfãs removidas: **`melhorRede`, `DIMKEY`, `canalVal`** (só alimentavam o `benchL`). Mantidos `vfmt`/`regua`/`SPEC`/`cls` (usados por Lojas/svizHTML).
   - **CSS:** 34 regras mortas removidas — `.gini-grid, .lorenz, .hbar(s) +desc, .skuchip +var, .quad, .scatter, .corr +desc, .stack(-row) +desc, .bench-l, .strat .diag, .hm .cell|.he|.ve|.mo|.au`. Vivas intactas (`.heat, .hm, .strat, .sviz, .pill, .kg, .ebar, .funnel, .sku-tbl, .vtag…`).

**Validação:** `node --check` OK; jsdom — 4 abas, Benchmark default 3 achados + heatmaps, Lojas 60 KPIs + PEF no expand, rename/XSS, **zero erros de console**. Arquivo 1610→1534 linhas.

## 27. Diagnóstico leva o fechamento "por que o canal cai" (24/06/2026)

A pedido do usuário, o `tendPara` do `renderCanal` foi reescrito para o **diagnóstico completo em 3 parágrafos** (tudo computado de `state.perf.cp`, p/ usar na conversa com gerentes):
1. **"o que está acontecendo"** — vendeu **−25,1% em unidades** e **−24,8% em clientes**; receita **−17,6%** só não foi pior porque **preço +10,0% (repasse de tabela) mascara**; é falta de gente entrando. Mesmas-lojas **−20,7%**.
2. **"por que (provável)"** — não é margem/mix/desconto; queda **larga** (todas −23% a −35%, perfis opostos) → **causa sistêmica**, não execução de loja; **conversão de ação de fluxo −5,6pp** na rede; meta PEF −14,6% (só São Sebastião bate).
3. **"o caminho"** — fluxo se recupera por **serviços / resgate de fidelidade (rede 54%) / conversão de ação de fluxo** (detalhe na aba Tendência); preço↑ com fluxo↓ é espiral, a defesa é tráfego.

Achado-chave consolidado: **a queda é tráfego (~25% menos unidades/clientes); o +10% de preço é repasse passivo que esconde o tamanho real do tombo.** Limites: causa externa não está no dado; faltam série **mensal** e base **por cliente (RFM)**. **Validação:** jsdom — 3 parágrafos, números corretos, zero erros de console.

## 28. "Gargalo nº 1" vira bubble de priorização (24/06/2026)

O usuário pediu o bloco "Gargalo nº 1 de cada loja" mais visual ("para um trabalho de data science"); escolheu **bubble/scatter** entre 3 opções. Nova função **`gargaloBubbleHTML(lojas, findings)`** (SVG puro, substitui a tabela `.gtbl`):
- **x = GMV/mês** (tamanho da loja), **y = R$/mês de oportunidade** do gargalo nº 1, **bolha ∝ % de ganho** sobre o GMV, **cor = tipo** do gargalo (`tipo()` por keyword: fluxo/desconto/cesta/recorrência/identificação/trocas/cauda). Medianas (sobre as lojas reais) desenham o quadrante **"ATACA PRIMEIRO ↗"**; `<title>` com detalhe no hover.
- **Armadilha tratada (essencial):** o gargalo de fluxo das lojas pequenas gera R$ irreal (São Sebastião +57%, Palmeira Sustentável **+364%** do GMV = *teto aritmético*). Plotado cru, a menor loja viraria a maior bolha no topo. **Fix:** `p.teto = op/gmv > 0,20` → essas saem da escala (`yMax`/`pctMax`/medianas só das reais), são fixadas no topo como **bolhas tracejadas pequenas (r=10) com "↑R$Xk\*"** + ressalva. Resultado honesto: **Palmeira** (grande + maior gargalo real, R$12k/8,3%, cesta) emerge no quadrante de prioridade junto da Coruripe; as 4 reais se espalham.
- **Polimento:** *dodge* horizontal p/ bolhas quase coincidentes (Penedo≈Teotônio, ~8px→52px) + **de-colisão de rótulos** (acima/abaixo). Legenda de cores + nota.
- **Limpeza:** `gargalo` e `byGmv` do `renderCanal` ficaram órfãos (a bolha tem `garg` próprio) → removidos.

**Validação:** `node --check` OK; jsdom — 4 abas, 6 bolhas (2 tracejadas/teto), rótulos sem colisão, rename/XSS escapado, `#fStrat` ok, **0 refs órfãs**, **zero erros de console**.

## 29. Tendência toda visual — cascata · tornado · slopegraph · lollipop (24/06/2026)

O usuário pediu a aba Tendência mais visual e **explicitamente não-bubble** ("não poderia ser outro estilo além de bubble?"). Trocadas as 3 tabelas por loja por **idiomas distintos** (SVG puro, helpers novos `waterfallSVG`/`tornadoSVG`/`slopeSVG`/`lollipopSVG` + `kBRL`/`heat3`):
- **Bloco 1 (ponte):** canal vira **cascata/waterfall** (Ano passado → −tráfego → −cesta → +preço → Atual, com conectores tracejados); por loja vira **tornado** (barras divergentes: tráfego perdido ◀ vermelho · preço ganho ▶ verde · Δreceita à direita).
- **Bloco 2 (funil):** mantém o funil do canal; por loja vira **slopegraph** (Cadastra → Resgata, uma linha por loja; mais íngreme = mais vaza; cor `heat3` pelo nível de resgate).
- **Bloco 3 (alavancas):** duas **lollipops** ranqueadas — Serviços em loja (Coruripe 613 lidera, Sustentável 0) e Conversão de Ação de Fluxo, com a variação YoY ao lado (lollipop aceita `vlabel` p/ %).
- **Limpeza:** removidos os helpers/row-builders órfãos (`ebar`/`mx`/`cR`/`sgnR`/`sgnPP`/`recYoY`-local/`funilRows`/`fluxRows`) e o CSS `.ebar*`.

Cada gráfico mantém uma leitura curta em cima. **Validação:** `node --check` OK; jsdom — 4 abas, **5 SVGs na Tendência** (cascata, tornado, slopegraph, 2 lollipops) + funil do canal, rename/XSS escapado, **0 refs órfãs**, **zero erros de console**.

## 30. Lojas mais visual — raio-X vs rede + scorecard PEF em bullet (24/06/2026)

O usuário escolheu **1 + 2** de três opções. Helpers novos (SVG): `perfilRedeSVG`, `pefBulletSVG`.
- **1 · Raio-X vs rede** (no card, sempre visível): cada KPI (`PERFIL` = cupons/dia, ticket, itens/cupom, fidelidade, desconto, recorrência, margem) numa faixa **min↔max das lojas** (`redeStats` calculado 1× sobre `lojas`), **● = esta loja**, **┃ = mediana**, cor pela posição (dir up/down). Fingerprint comparativo por loja, **adicionado abaixo do kgrid** (kgrid + réguas mantidos).
- **2 · Scorecard PEF em bullet bars** (no expand, substitui a tabela de 15 linhas do `perfPanel`): barra = atual; **┃ ouro = meta** (reconstruída por `v/(1+vsMeta)` p/ receita/cupons/ticket); penetrações = nível 0–100%; **cor da barra = vs ano** (verde melhorou / vermelho piorou); deltas "m"=meta, "a"=ano ao lado.
- **Limpeza:** removidos `dChip`/`sgnL`/`yoUnit` (só a tabela PEF antiga usava). As únicas tabelas restantes em Lojas são os top-10 SKUs (opção 3, não pedida).

**Validação:** `node --check` OK; jsdom — 4 abas, **6 raio-X nos cards + 6 bullets PEF nos expands**, rename/XSS escapado, **0 refs órfãs**, **zero erros de console**.

## 31. Benchmark — parallel coordinates no lugar do heatmap principal (24/06/2026)

O usuário escolheu **parallel coordinates** (entre 3 opções) para deixar a aba Benchmark mais visual. Helper novo `parallelCoordsSVG(lojas, axes)` (+ paleta `PC_PAL`):
- Cada loja é uma **polyline** cruzando 8 eixos (GMV/mês, Cupons/dia, Ticket, Itens/cupom, Fidelidade, Desconto, Margem, Recorrência), normalizados por eixo (min↔max das lojas) com **cima = melhor** (eixos `dir:'down'` invertidos — ex.: desconto). Cor = identidade da loja (legenda), valores no `<title>` (hover). Eixos sem spread são descartados.
- **Substitui o `matrizHTML`** (heatmap loja×indicador, removido). Mantidos o **heatmap PEF** (`perfMatrizHTML`) e os achados com régua (filtro abre na maior loja).
- **Limpeza:** `matrizHTML` removido + CSS `.rank*` (só ele usava).

Mostra o "formato" de cada loja e o espelho Coruripe↔Palmeira visualmente. **Validação:** `node --check` OK; jsdom — 4 abas, parallel coords (6 linhas/8 eixos) + heatmap PEF + achados, `#fStrat` ok, rename/XSS escapado, **0 refs órfãs**, **zero erros de console**.

> **Estado visual da app (24/06/2026):** 4 abas, todas com visual SVG próprio — Diagnóstico (pills + bubble de gargalos), Tendência (cascata/tornado/slopegraph/lollipop + funil), Lojas (raio-X vs rede + bullets PEF + donut/ABC), Benchmark (parallel coordinates + heatmap PEF + achados-régua).
