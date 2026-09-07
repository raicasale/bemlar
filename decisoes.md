# Definição do Alvo

O objetivo do piloto de cobrança preventiva é evitar que contratos de crediário atinjam **30 dias de atraso**, momento em que as regras contábeis da BemLar exigem provisionamento de perda (estimado em média de **R$ 1.200,00 por contrato**). 

Para que o modelo apoie uma ação verdadeiramente preventiva (ligar antes do vencimento), o alvo e a temporalidade foram estruturados com rigor metodológico.

---

### 1. Regra do Alvo Binário (`inadimplente_30d`)

A variável dependente ($y$) foi construída sobre a **parcela de referência** de cada contrato, seguindo a regra:

$$
\text{inadimplente\_30d} = \begin{cases} 
1, & \text{se a parcela de referência não foi paga até } D+30 \text{ dias após o vencimento (ou permaneceu em aberto até o snapshot)} \\ 
0, & \text{se a parcela foi paga em dia ou com atraso inferior ou igual a 30 dias} 
\end{cases}
$$

**Consequências práticas da regra:**
- **Atrasos leves não são inadimplência:** Clientes que atrasam 3, 10 ou 20 dias recebem valor $0$, pois não geram provisionamento de perda e costumam regularizar sem acionamento crítico.
- **Contratos quitados posteriormente:** Se o cliente atrasou mais de 30 dias na parcela avaliada, o alvo é $1$, mesmo que ele tenha quitado a dívida meses depois (a quitação futura não anula o risco existente no momento da decisão).

### 2. Critério de Escolha da Parcela de Referência

A base foi extraída em **15/03/2026** (`data_snapshot`). Para observar com certeza se um contrato atrasou mais de 30 dias, o desfecho precisa já ter ocorrido no mundo real:
- **Janela de observação mínima:** 30 dias antes do snapshot $\rightarrow$ **13/02/2026** ($15/03/2026 - 30\text{ dias}$).
- **Regra de seleção:** Para cada um dos 3.000 contratos, identificou-se a **última parcela vencida até 13/02/2026** em `pagamentos.csv`.
- Todo contrato da base possui pelo menos uma parcela elegível nessa janela.

### 3. Data de Referência e Prevenção de Vazamento (*Data Leakage*)

Para simular o momento operacional da ligação preventiva:
$$
\text{data\_referencia} = \text{vencimento da parcela de referência} - 7 \text{ dias}
$$
- O corte em $D-7$ reproduz o instante exato em que o time de cobrança recebe a fila para agir antes do vencimento.
- **Corte temporal estrito:** Nenhum evento (compras adicionais, chamados no SAC, pagamentos ou consultas de birô) ocorrido após a `data_referencia` entra nas features do modelo.

### 4. Prevalência dos Dados

Na carteira total avaliada (3.000 contratos):
- **Inadimplentes ($\text{alvo} = 1$):** 1.300 contratos (**43,3%**)
- **Adimplentes ($\text{alvo} = 0$):** 1.700 contratos (**56,7%**)

Essa taxa reflete o modelo de concessão em crediário próprio de varejo para classes populares, compatível com a dor financeira reportada pelo Ricardo no e-mail.

---

# Pedidos do Ricardo

No e-mail de especificação do piloto de cobrança preventiva, os pedidos do Ricardo foram:

    - Maior acurácia possível;
    - Uso do campo "status_contrato";
    - Uso das informações de sexo e localização;;
    - Uso do score bureau;
    - Uso da lista de motivos de atraso.

Inicialmente, havíamos decidido atender o pedido de maior acurácia, uso do campo "status_contrato", uso do score bureau e o uso da lista de motivos de atraso, anotados pelo time de SAC. Durante o desenvolvimento, percebemos que esses pedidos não poderiam ser atendidos na íntegra, pois comprometeriam o desempenho do modelo, ou porque tecnicamente, não seriam viáveis.

### Decisões

#### Pedido 1: "Quero a maior acurácia possível (> 80%). Se vier abaixo disso, prefiro nem apresentar ao conselho."
- **Status:** **RECUSADO COM ALTERNATIVA (Redefinição da Métrica)**
- **Motivo da Recusa:** A acurácia global mede o percentual de acertos sobre a base total em um limiar genérico de 50%. Em problemas com forte restrição operacional — onde a equipe tem capacidade fixa para apenas **80 ligações semanais** —, a acurácia é uma ilusão estatística. Um modelo poderia ter 82% de acurácia global simplesmente acertando os bons pagadores e errando quase todos os contratos da fila de 80, resultando em ligações desperdiçadas e prejuízo mantido.
- **Alternativa Implementada:** Avaliação centrada na **Precision no corte de 80 (Precision@80)** e no **Recall@80**. A mensagem para o conselho traduz valor financeiro real: *"Dos 80 clientes que a cobrança ligou, quantos de fato iam atrasar?"*. Nosso modelo atinge **76,25% de precisão nos 80** (61 inadimplentes interceptados = R$ 73.200,00 de prejuízo evitado por semana), superando com folga a fila por valor de contrato (51,25%) e a fila aleatória (41,25%).


#### Pedido 2: "Usem o campo `status_contrato` do cadastro (Em dia / Inadimplente / Quitado), que é a verdade do sistema."
- **Status:** **RECUSADO**
- **Motivo da Recusa:** Provoca **vazamento temporal severo (*data leakage*)**. O campo `status_contrato` é dinâmico e reflete o momento da extração do arquivo (snapshot em 15/03/2026), capturando acontecimentos ocorridos semanas *após* a data em que a ligação preventiva seria realizada ($D-7$). Além disso, há 62 divergências materiais: contratos que sofreram inadimplência de 30 dias na parcela avaliada aparecem como "Quitado" porque o cliente pagou a dívida meses depois, enquanto outros constam como "Inadimplente" por quebras ocorridas após o corte.
- **Alternativa Implementada:** Construção matemática e auditável da variável binária `inadimplente_30d`, avaliada exclusivamente na parcela de referência com atraso $> 30$ dias, alimentada apenas por histórico transacional ocorrido até 7 dias antes do vencimento.


#### Pedido 3: "Bairro e sexo entram na conta. Mulher de periferia atrasa mais."
- **Status:** **RECUSADO**
- **Motivo da Recusa:**
  1. **Critério Ético e Legal (LGPD e Não Discriminação):** `sexo` é dado pessoal sensível. Utilizá-lo em algoritmos de risco financeiro institucionaliza discriminação de gênero e expõe a BemLar a severas sanções legais e danos reputacionais.
  2. **Viés Socioeconômico e Geográfico (*Redlining*):** `bairro` atua como proxy discriminatório de vulnerabilidade social e CEP, punindo clientes pela sua localidade e não por sua conduta de pagamento. Apresenta ainda inconsistência cadastral (`"Jd. America"` vs `"Jardim América"`).
  3. **Comportamento vs. Identidade:** Modelos de crédito justos e eficientes devem julgar *como o cliente honra seus compromissos* e não *quem a pessoa é* ou *onde ela reside*.
- **Alternativa Implementada:** Substituição integral por variáveis objetivas de mérito financeiro e capacidade de pagamento: comprometimento do contrato sobre a renda (`valor_total / renda`), histórico de atrasos prévios no crediário (`media_dias_atraso`, `taxa_atrasos`) e fidelidade na rede (`compras_anteriores`).


#### Pedido 4: "Anexei o score de bureau (Serasa/SPC). Se melhorar o número, usem sem dó."
- **Status:** **REPROVADO**
- **Motivo da Recusa:** O score bureau de crédito externo agrega o comportamento do tomador em todo o sistema financeiro nacional, sendo um preditor legítimo de propensão ao pagamento. Porém, ao aplicarmos o filtro de data pela data de refêrenca do contrato, perdemos registros, já que contratos podem possuir mais de uma pontuação, uma feita no momento da aprovação, e outra em revisão. Foi observado que em 86% dos casos o score diminuiu no momento da revisão, mas como as datas são posteriores à data de referência, essa informação é perdida.
- **Alternativa Implementada:** Ao invés de utilizar o score bureau, utilizamos variáveis ​​internas comportamentais e financeiras, como `media_dias_atraso` e `taxa_atrasos` (que, juntas, representam 51,6% do poder preditivo), além de também termos usado `percentual_valor_contrato_renda`. Todas as variáveis citadas estão disponíveis em tempo real em D−7, sem depender de bureaus externos.


#### Pedido 5: "Tem também a lista de motivos de atraso anotada pelo SAC. Deve ajudar a entender o que está acontecendo."
- **Status:** **RECUSADO NO MODELO QUANTITATIVO (Mantido para diagnóstico qualitativo)**
- **Motivo da Recusa:** O arquivo `motivos_atraso.csv` (1.019 registros de texto livre) **não contém identificadores** (`id_contrato` nem `id_cliente`), possuindo apenas o texto digitado e a data. É tecnicamente inviável vincular esses registros de forma determinística e confiável a cada contrato na matriz de treino.
- **Alternativa Implementada:**
  1. No modelo preditivo tabular, utilizou-se a variável comportamental de chamados do SAC (`quantidade_atendimentos_sac`), que possui vínculo por contrato e captura o atrito operacional e pedidos de 2ª via de boleto.
  2. Como plano de ação operacional, recomendamos à área de TI e Operações que o sistema de CRM passe a vincular obrigatoriamente o `id_contrato` a qualquer anotação de SAC, permitindo que futuras versões do modelo utilizem Processamento de Linguagem Natural (NLP).

---

# Model Card — Piloto de Cobrança Preventiva (BemLar)

## 1. Dados de Treinamento
- **Fonte e Escopo:** 3.000 contratos de crediário próprio da BemLar (1.824 clientes únicos), extraídos em snapshot de 15/03/2026.
- **Bases Integradas:**
  - `contratos.csv`: dados cadastrais e financeiros do contrato (rendas zeradas imputadas pela mediana de R$ 2.823,22).
  - `pagamentos.csv`: 28.861 parcelas históricas anteriores à data de referência.
  - `compras.csv`: 3.983 compras adicionais na rede agregadas por `id_cliente` (ausência = 0).
  - `ocorrencias_sac.csv`: 3.446 chamados agregados por `id_contrato` (ausência = 0).
- **Corte Temporal:** Todas as variáveis respeitam a data de referência de cada contrato (7 dias antes do vencimento da parcela avaliada), impedindo vazamento de dados (*data leakage*).
- **Variável Alvo (`inadimplente_30d`):** 1 se a parcela atrasou > 30 dias (ou permaneceu sem pagamento até o snapshot); 0 caso contrário. Prevalência: 43,3% (1.300 contratos).
- **Estratégia de Validação:** Split temporal cronológico (75% contratos mais antigos no treino / 2.250 contratos; 25% mais recentes no teste / 750 contratos).

## 2. Features Utilizadas
O modelo final utiliza 10 variáveis preditivas (comportamentais e financeiras):

1. `media_dias_atraso`: média de dias de atraso nas parcelas anteriores (31,1% de importância).
2. `taxa_atrasos`: proporção de parcelas pagas com atraso sobre as vencidas (20,5% de importância).
3. `max_dias_atraso`: pior atraso histórico registrado em dias.
4. `total_parcelas_atrasadas`: quantidade acumulada de parcelas quitadas com atraso.
5. `percentual_parcelas_pagas_valor_total`: percentual financeiro amortizado do contrato.
6. `percentual_valor_contrato_renda`: comprometimento do valor contratado sobre a renda mensal declarada.
7. `valor_total_contrato`: valor financiado no crediário próprio.
8. `numero_compras_anteriores`: quantidade de compras adicionais realizadas na rede BemLar.
9. `media_valor_compras_anteriores`: ticket médio de outras compras do cliente na rede.
10. `quantidade_atendimentos_sac`: contagem de chamados abertos no SAC (achado negativo: 1,35% de importância).

## 3. Objetivo e Uso Pretendido
- **Finalidade:** Ordenar e gerar semanalmente a fila operacional fixa de **80 ligações** preventivas para a equipe de cobrança da BemLar acionar clientes 7 dias antes do vencimento.
- **Aplicação de Negócio:** Reduzir provisionamento e perdas financeiras (custo médio de R$ 1.200,00 por contrato com atraso > 30 dias) via oferta antecipada de renegociação amigável.
- **Métrica Prioritária:** **Precision@80** (eficiência de contato frente ao limite físico de 80 chamadas por semana).
- **Desempenho:** Atinge **76,25% de precisão no corte de 80** (61 contratos inadimplentes interceptados = R$ 73.200,00 de prejuízo evitado/semana), superando a fila por valor de contrato (51,25%) e a aleatória (41,25%).

## 4. Limites Conhecidos de Uso
- **Janela de Ação Estrita ($D-7$):** O modelo é projetado exclusivamente para predições com corte em 7 dias antes do vencimento; não deve ser utilizado na esteira de concessão inicial ($D-0$) nem em cobrança de dívidas já vencidas ($D+30$).
- **Capacidade Operacional Fixa:** Otimizado para alta precisão nos 80 primeiros nomes. Seu recall no corte é baixo (~19,4% dos inadimplentes da carteira de teste), logo não serve como medidor de risco sistêmico da carteira total.
- **Restrições Éticas e LGPD:** Variáveis sensíveis (`sexo` e `bairro/localização`) foram excluídas deliberadamente para evitar discriminação algorítmica e manter conformidade ética e legal.
- **Exclusões Técnicas por Inconsistência:**
  - `motivos_atraso.csv` foi descartado por falta de chave de identificação (`id_contrato`/`id_cliente`).
  - `status_contrato` e revisões de `score_bureau` foram descartados por conterem eventos posteriores à data de referência (*data leakage*).
- **Sensibilidade à Maturação Temporal:** Contratos mais recentes apresentam menor histórico comportamental, reduzindo a precisão em safras novas; demanda retreinamento e monitoramento contínuo de safra.
- **Escopo Exclusivo:** Calibrado especificamente para o crediário próprio de bens duráveis da BemLar, não sendo recomendada sua aplicação em outros produtos financeiros ou empresas sem recalibração prévia.

