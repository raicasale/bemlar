# Correção — Piloto de Cobrança Preventiva (BemLar)

**Avaliação final da disciplina** · Correção baseada em `enunciado_prova_bemlar.md` e `00_email_financeiro.md`

## NOTA: **8,0 / 10**

Entrega forte, de nível profissional na documentação e na disciplina temporal, com duas falhas objetivas que o próprio enunciado nomeia como eliminatórias de ponto.

**Método da correção:** o notebook foi executado do zero em pasta limpa, a partir dos CSVs originais; todos os números do texto foram reproduzidos; o vazamento temporal foi auditado parcela a parcela; e as duas recusas mais frágeis (score de bureau e SAC) foram reexaminadas empiricamente.

---

## O que está muito bem feito

**Alvo e recorte temporal (o núcleo da prova) — quase impecável.** A parcela de referência, a `data_referencia = vencimento − 7d` e a regra `atraso > 30` estão corretas. A prevalência de 43,3% bate. O cruzamento com `status_contrato` não parou na tabela: as 62 discordâncias foram abertas nas três categorias e interpretadas corretamente, inclusive o caso difícil (22 contratos "Quitado" que *eram* risco em D−7 — a quitação posterior é informação futura).

**A investigação do score de bureau é o ponto alto da entrega.** Descobrir que 100% das 1.490 "revisões de carteira" ocorrem *depois* da data de referência, e demonstrar o vazamento quantificando a AUC falsa (0,693 → 0,949), é a armadilha mais difícil do enunciado. Verificado: os 86,2% de queda no score da revisão conferem. Poucos alunos chegam nesse nível de evidência.

**Recusas éticas bem construídas.** Sexo e bairro recusados com os três eixos certos — LGPD, *redlining*, comportamento vs. identidade — e com alternativa concreta. O enunciado exige "não dá desse jeito; me mande X e dá", e a entrega faz isso.

**Avaliação no corte de 80 exemplar.** Duas filas-base (aleatória 41,25% e por valor 51,25%) contra o modelo (76,25%), FP/FN traduzidos em reais, e a escolha por *precision* justificada pelo negócio (capacidade física de 80 ligações), não pela métrica. Split temporal comparado ao aleatório com a seção "o que isso NÃO diz" — exatamente o tipo de honestidade pedida.

**`decisoes.md` completo**, com model card cobrindo os quatro elementos exigidos, incluindo limites de uso realistas (recall de ~19%, janela D−7 estrita, sensibilidade à safra).

**`features.csv` e `fila_80.csv` são bit-a-bit idênticos ao que o notebook regenera.** Nada foi editado à mão.

---

## Os problemas

### 1. O notebook não roda do zero *(falha grave — item explícito do enunciado)*

Executado em pasta limpa, ele **quebra**:

```
NameError: name 'features' is not defined
```

Em `enunciado.ipynb`, dentro de `construir_features`, a linha de exportação usa a variável global `features.to_csv(...)` em vez de `df_out.to_csv(...)`. Como `features` só passa a existir *depois* da chamada, o notebook só funciona com um kernel que já rodou antes. É literalmente o cenário que o enunciado cita pelo nome: *"Notebook que só funciona na sua máquina não é entregável."* Corrigindo essa única linha, tudo o mais roda e reproduz os números exatos.

### 2. A assinatura de `construir_features` não é a especificada

O enunciado pede `construir_features(caminho_dados, datas_referencia)`. Foi entregue `construir_features(df_contratos=contratos, df_alvo=alvo, df_pags=df_pagamentos, ...)`, recebendo DataFrames globais **já filtrados** como default. A função não lê os CSVs nem aplica o corte temporal — ela depende do estado do notebook, que é justamente o que o encapsulamento deveria eliminar.

### 3. O "achado negativo" do SAC está errado — e foi para o slide do Ricardo

Este é o achado mais sério da correção. A entrega usa `quantidade_atendimentos_sac` como **contagem total**, obtém 1,35% de importância e conclui no slide 2 que *"cruzar chamados do SAC com o fluxo de cobrança [é] uma mobilização de atendentes que não traria resultados."*

Desagregando por `tipo_ocorrencia` (respeitando o mesmo corte D−7):

| Tipo de ocorrência | Inadimplência com | sem | lift |
|---|---|---|---|
| **Segunda via de boleto** (n=982) | **57,2%** | 36,6% | **+20,7 p.p.** |
| Reclamação de entrega | 39,2% | 43,8% | −4,6 p.p. |
| Dúvida sobre produto | 38,5% | 43,8% | −5,3 p.p. |
| Assistência técnica | 40,7% | 43,6% | −2,9 p.p. |
| Atualização cadastral | 42,2% | 43,5% | −1,3 p.p. |

Pedir 2ª via de boleto é o sinal comportamental mais forte que existe fora do histórico de atraso. Ao somar as cinco categorias — quatro delas com lift levemente negativo — o sinal foi diluído para +9,1 p.p. e depois declarado irrelevante. A célula 38 chega a afirmar que *"clientes ligam no SAC por razões operacionais (ex.: 2ª via de boleto), mas isso não indica deterioração financeira"* — exatamente o contrário do que os dados mostram.

É a falha do item do checklist *"Verifiquei os valores das colunas categóricas, não só os nomes das colunas"*: os valores foram **listados** na Fase 2, mas nunca **usados**. E o enunciado pede um achado negativo justamente para *"poupar a empresa de uma política inútil"* — aqui, o achado negativo faria a empresa descartar uma política útil.

### 4. A recusa do score de bureau contradiz o próprio notebook

A célula 16 conclui, corretamente: *"Decisão: usar o score da consulta de aprovação (único disponível antes da data de referência)"* — e observa que após o filtro restam 3.000 registros, um por contrato. Mas a tabela de auditoria (célula 28) e o `decisoes.md` dizem **"Sai"**, com a justificativa de que *"essa informação é perdida"*. Não é: o score de aprovação sobrevive ao filtro para 100% dos contratos.

A decisão certa era **atender parcialmente** — usar o score de aprovação, recusar a revisão. Impacto testado: incluir o score de aprovação dá Precision@80 idêntica (62/80). Ou seja, o prejuízo é de coerência e de argumentação, não de performance — mas é uma recusa a um pedido do Ricardo apoiada numa premissa falsa, e o enunciado avalia exatamente a qualidade dessas recusas.

### 5. Vazamento residual em `pagamentos` *(pequeno)*

`filtra_por_data` filtra por `data_vencimento`, mas não por `data_pagamento`. Parcelas vencidas antes de D−7 e **pagas depois** entram no cálculo de atraso com informação futura: 108 parcelas (0,4%), 2 dias de futuro em média. Efeito desprezível, mas é uma falha na dimensão que a prova está medindo — e destoa do cuidado demonstrado no resto.

### 6. Entregáveis e deck

- **`modelo.ipynb` não existe** — a entrega está em `enunciado.ipynb`. Não há `.zip`.
- **A iteração CRISP-DM não foi registrada** como pedido na seção 6. Há vestígios de mudança de ideia ("Inicialmente... mas após discussões", células 19 e 24), mas nenhum registro explícito de retorno de Modelagem → Preparação.
- **Gráficos cortados.** O gráfico do slide 1 perdeu o painel de R$ na captura — some justamente a "manchete em dinheiro" que o enunciado pede. O do slide 2 está com o título truncado ("...ACHADO NEGATIV") e com dois ícones da interface do notebook dentro da imagem. São *screenshots*, não exportações.
- **Slide 3 não declara a decisão explícita.** O enunciado exige "liga / liga com ressalva / não liga ainda". O texto ("funciona e melhora, porém antes seria necessário...") equivale a *liga com ressalva*, mas nunca usa o rótulo. Slides 1 e 3 também estão densos demais para um CFO. Ponto positivo: o slide 3 não cai no "acurácia de 81%, aprovado".
- `fila_80.csv` traz a probabilidade como texto (`"97.21%"`) em vez de número — desconfortável para um arquivo operacional.

---

## Resumo por bloco

| Bloco | Peso | Nota |
|---|---|---|
| Alvo, data de referência, corte temporal | 2,5 | 2,3 |
| Avaliação no corte de 80 | 2,0 | 1,9 |
| Features e auditoria de colunas | 1,5 | 1,1 |
| Modelagem e splits | 1,5 | 1,3 |
| `decisoes.md` + model card | 1,5 | 1,25 |
| Notebook executável + função especificada | 1,0 | 0,3 |
| Deck (3 slides) | 1,0 | 0,65 |
| **Total** | **10,0** | **8,0** |

---

## Checklist do enunciado (seção 8)

| Item | Situação |
|---|---|
| Alvo segue a regra dos 30 dias, explicável em uma frase | ✅ |
| Nenhuma feature usa evento posterior à data de referência | ⚠️ 108 parcelas (0,4%) com pagamento posterior a D−7 |
| Verifiquei os **valores** das colunas categóricas | ❌ listados, mas não usados — ver problema 3 |
| Toda base agregada virou uma linha por contrato, ausência = valor certo | ✅ |
| Comparei minha fila com uma fila-base | ✅ duas filas-base |
| Cada pedido do e-mail respondido: atendido ou recusado, com motivo | ⚠️ os 5 respondidos, mas o do bureau em premissa falsa |
| O notebook roda do zero, em outra máquina, a partir dos CSVs originais | ❌ `NameError` |
| O slide 3 não diz "acurácia de 81%, piloto aprovado" | ✅ |

---

## As três correções de maior retorno

1. **Trocar `features.to_csv` por `df_out.to_csv`** e reescrever a função com a assinatura `construir_features(caminho_dados, datas_referencia)` — vale quase um ponto inteiro por uma tarde de trabalho.
2. **Desagregar o SAC por tipo** e substituir o achado negativo por um achado *positivo* real ("cliente que pede 2ª via de boleto atrasa 57% contra 37%"), que é uma manchete muito melhor para o Ricardo.
3. **Reverter a recusa do bureau para "atendido com ressalva"**, que é o que o próprio notebook já concluiu na célula 16.
