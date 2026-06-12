# CONTEXTO — RADAR LOJAS (handoff técnico)

> Documento de continuidade. Permite retomar o desenvolvimento do `radar-lojas.html` em qualquer ambiente sem perder contexto. Última atualização: 12/06/2026.

---

## 1. O que é

**Radar Lojas** é uma aplicação standalone (arquivo HTML único, processamento 100% client-side no navegador) de **diagnóstico e estratégias de recuperação do canal loja** do Grupo Alcina Maria (rede O Boticário, Penedo/AL). O usuário arrasta relatórios brutos exportados do ERP e a app produz: benchmarking entre lojas, curva ABC e mix por loja, motor de estratégias priorizadas por impacto em R$ e simulador de cenários.

- **Stack:** HTML/CSS/JS vanilla + pdf.js 3.11.174 (cdnjs) + SheetJS 0.18.5 (cdnjs). Sem build, sem servidor, sem localStorage (estado só em memória).
- **Padrão da casa:** mesmo modelo das ferramentas Foco Atividade e painel-vendas (arquivo único que processa arquivos reais no browser).
- **Privacidade:** nenhum dado sai da máquina.

## 2. Contexto de negócio

- Rede com **7 lojas** (canal loja) sob a operação ACQUA DISTRIBUIDORA DE PERFUMES E COSMETICOS LTDA. Só a loja **24303** foi carregada/analisada até agora; as demais serão adicionadas pelo usuário (de-para código → nome ainda não informado).
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
6. **Render** — 4 abas (Visão do canal / Lojas / Estratégias / Simulador): KPI pills, ranking com heatmap por coluna (direção configurada: desconto/trocas menor=melhor), cards de loja com **réguas de semáforo** (assinatura visual; `SPEC` mapeia valor→posição no gradiente verde→âmbar→vermelho), donut SVG de mix, barras da curva ABC, top 10 SKUs, chips de marcas, alertas; estratégias com checklist; simulador com 3 sliders (Δdesconto pp, Δitens/cupom, Δcupons/dia) e conta aberta + projeção mensal/anual; botão **"Copiar resumo p/ Slack"**.
7. **SEED** — dados completos da 24303 embutidos (meta do rodapé + 702 itens minificados `[sku, nome, qtd, fat, custo, lucro]` + linha de indicadores). App abre funcional; re-upload do PDF real da 24303 substitui o seed.

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

1. **Carregar as 6 lojas restantes** (PDFs ABC + planilha consolidada) → réguas passam a calibrar pelo canal.
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

**Validação re-executada (critério permanente mantido):** 7/7 PDFs reais com selo ✓ (somas do rodapé ao centavo; 5905 com nota de contagem), CSV real com 7 lojas e valores exatos, referência da 24303 intacta (169 dias, descEtiq 26,10%, margem 62,2%, curva A=220), pipeline completo com 7 lojas sem erro.

**⚠ Pendência de dados — loja 5905:** o PDF (fat R$ 420,7k no período, header ACQUA) **não bate** com a linha 5905 do CSV (GMV R$ 201,6k, razão ALAN MARTINS TAVARES E CIA LTDA) — ~2x de diferença na mesma janela. Provável código repetido entre bases/lojas distintas. O chip de alerta da correção 5 sinaliza isso na UI; falta o usuário confirmar qual fonte é a loja certa.

## 10. Deploy (Render, static site)

- `render.yaml` na raiz (Blueprint): o build copia `radar-lojas.html` → `public/index.html` e publica só a pasta `public/` — este MD e quaisquer dados não vão para o site. Header `X-Robots-Tag: noindex` para não indexar em buscadores.
- Repositório: https://github.com/alcinadadosti-worspace/helpLoja.git (os relatórios *.pdf/*.csv ficam fora por `.gitignore`).
- **Atenção privacidade:** a app embute o SEED com os dados reais da loja 24303; a URL do Render é pública para quem a tiver. O processamento de arquivos continua 100% client-side (nada é enviado a servidor).
