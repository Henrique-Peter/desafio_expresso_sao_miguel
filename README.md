# Projeto de Análise de Dados Operacionais e Estruturais

## Contexto do projeto

Este projeto analisa dados operacionais da Expresso São Miguel (logística na região Sul do Brasil), com foco em extrair insights acionáveis a partir de três bases:

- `volumetria`: registros operacionais por viagem (peso, cubagem, volumes, CTEs, desconto, tipo de tarifação)
- `cliente`: cadastro básico do cliente (UF, município, tipo pessoa, razão social)
- `cliente_cnae`: CNAE(s) associado(s) ao cliente, com indicação de principal

O objetivo foi entender padrões de operação, concentração de volume, diferenças estruturais entre grupos de clientes e implicações comerciais/operacionais do desconto aplicado.


## Principais achados consolidados

### 1) Concentração extrema de volume por cliente

Foi observado comportamento fortemente concentrado:

- ~4% dos clientes geram ~80% do volume (proxy em `pesom3`)

Isso caracteriza um cenário clássico de dependência de poucos clientes (risco de concentração), com impacto direto em estabilidade de receita e ocupação operacional.

Implicação:
- qualquer decisão comercial sobre a “cauda curta” tem efeito desproporcional no negócio


### 2) Padrões temporais operacionais consistentes (dia e hora)

#### Por dia da semana
A operação é tipicamente “de dias úteis”:

- picos entre terça e quinta, com destaque para quarta
- queda forte em fim de semana, principalmente domingo

Implicação:
- capacidade e escala operacional parecem dimensionadas para dias úteis
- existe potencial de ações para suavizar demanda ou elevar ocupação em janelas ociosas (fim de semana / horários específicos), se houver viabilidade comercial

#### Por hora do dia (com distribuição e heatmap)
A distribuição horária mostrou padrões claros e recorrentes, inclusive com diferenças por dia:

- concentração em faixas específicas (manhã cedo e noite)
- finais de semana com comportamento distinto (especialmente domingo)

Implicação:
- planejamento de pátio, docas e recursos pode ser alinhado a janelas de pico
- oportunidades de ajuste de SLA/agenda (quando aplicável) podem reduzir gargalos


### 3) Desconto e tarifação: comportamento consistente com regra operacional

Foi verificado que `pesom3` (peso cubado/tarifário) tende a ser maior ou igual ao peso real quase sempre, como esperado:

- em praticamente todos os registros, `pesom3 >= peso`

Além disso, a análise de `fator_cubagem = pesom3 / m3` mostrou:

- mediana próxima de 1.0, mas com cauda longa e outliers extremos
- grande volume de registros com `m3 = 0` (~80%), o que impede uso direto do fator sem filtragem

Implicação:
- `m3` não é um driver confiável universalmente (muitos zeros)
- `pesom3` é a métrica mais robusta para representar “capacidade cobrada/ocupada” na base


### 4) Top 4% vs Demais 96%: diferenças operacionais claras

O Top 4% é estruturalmente diferente do restante:

- peso médio por envio muito maior
- cubagem média maior
- volumes médios muito maiores (fragmentação)
- CTEs médios maiores

E, ao mesmo tempo:

- CTEs por tonelada menores (mais consolidação por documento)

Interpretação:
- Top 4% combina escala (mais peso/cubagem) com maior complexidade física (muitos volumes)
- não é apenas um grupo “que gera receita”; é um grupo com impacto real na operação

Implicação:
- desconto no Top 4% não deve ser analisado só como “perda comercial” ou “alavanca de preço”
- existe trade-off entre eficiência (escala) e custo operacional (manuseio)


### 5) CNAE: diversidade relevante dentro do Top 4%

Ao decompor por CNAE no Top 4%:

- é preciso um conjunto relativamente grande de CNAEs para cobrir frações altas do volume (ex.: dezenas para 50%, quase 100 para 80%)

Interpretação:
- a concentração de volume é por cliente, mas não necessariamente por um único setor
- o mix setorial do Top 4% é mais diversificado do que parece à primeira vista

Implicação:
- iniciativas comerciais devem ser mais “account-based” do que “setor-based”
- mas CNAE ajuda a orientar hipóteses de perfil de carga e sazonalidade


### 6) Geografia e dependência regional: concentração forte no Sul

A participação do volume total por UF mostrou concentração:

- SC, RS e PR respondem pela maior parte do volume (com SP como quarto polo relevante)

Além disso, ao comparar desconto (Top 4% - Demais 96%) por UF:

- algumas UFs exibem gap maior de desconto
- outras têm valores próximos de zero ou inconsistentes por baixa amostra

Implicação:
- a dependência regional é coerente com o escopo da empresa (Sul), mas também explicita risco de concentração geográfica
- gap de desconto por UF pode indicar estratégia comercial diferenciada ou composição distinta do mix de clientes


## Simulações e decisões: sensibilidade entre desconto e churn

Foi construída uma simulação para quantificar o trade-off:

- redução de desconto no Top 4% (com foco em cargas “pesadas”, faixa 81+)
versus
- churn percentual do Top 4%

Os resultados mostraram que:

- pequenas reduções de desconto geram ganho limitado
- churn mesmo baixo rapidamente supera esse ganho (impacto líquido negativo em quase todos os cenários realistas)

Interpretação executiva:
- para “pagar” uma redução de desconto pequena, o churn aceitável precisa ser muito baixo
- a elasticidade implícita (risco de churn) domina o resultado

Implicação:
- política de reajuste deve ser cirúrgica e orientada por risco
- decisões generalistas no Top 4% tendem a piorar o resultado líquido


## Recomendações acionáveis

### A) Estratégia comercial (Top 4%)

1. Evitar reajustes horizontais no Top 4%
   - a simulação indica que o risco de churn domina o ganho

2. Priorizar negociação por clusters operacionais
   - segmentar Top 4% por estrutura (peso/CTE, volumes/CTE, cubagem)
   - atacar onde há “alto volume + alta complexidade + alto desconto”

3. Criar política de desconto baseada em perfil de carga
   - ex.: maior desconto para cargas consolidadas (boa relação peso/CTE)
   - menor desconto (ou taxa operacional) para alta fragmentação (volumes/CTE alto)


### B) Eficiência operacional

4. Tratar `m3 = 0` como problema de qualidade de dado / processo
   - pode ser falha de preenchimento, integração ou regra operacional
   - recomendação: mapear origem e corrigir upstream

5. Criar indicadores recorrentes de complexidade
   - `volumes_por_cte`
   - `ctes_por_ton`
   - `peso_por_cte`
   - esses indicadores são bons candidatos a KPIs para custo operacional


### C) Planejamento de capacidade (temporal)

6. Usar o padrão dia/hora como base para dimensionamento
   - alocação de recursos e janelas críticas de operação
   - analisar se picos são “demanda real” ou “agenda operacional”

7. Explorar oportunidade de suavização (quando aplicável)
   - incentivos para janelas de menor ocupação
   - acordos comerciais por faixa horária/dia


### D) Geografia e risco regional

8. Consolidar estratégia para os três polos principais (SC/RS/PR)
   - tratar como “regiões core”
   - mapear dependência por contas-chave em cada UF

9. Investigar UFs com gap alto de desconto e baixa participação
   - risco de distorção por pouca amostra
   - ou indício de política comercial específica (vale aprofundar)


## Próximos passos recomendados (encerramento prático)

Se eu tivesse mais um ciclo curto de trabalho, priorizaria:

1. Qualidade de dado de `m3`:
   - entender por que ~80% é zero
   - checar por `tipopesocubico` / cliente / UF / data

2. Modelo simples de propensão a churn:
   - clientes do Top 4% com desconto alto vs estrutura operacional
   - priorização de contas para retenção (sem precisar dado externo)

3. Validação com área comercial/operacional:
   - confirmar se padrões temporais refletem agenda interna
   - confirmar hipóteses de complexidade por volumes


## Encerramento

Os dados mostram que a Expresso São Miguel opera com forte concentração de volume em poucos clientes, em uma malha regionalmente concentrada no Sul, e com padrões temporais consistentes.

O Top 4% é crítico não só financeiramente, mas também por ocupação e estrutura operacional. A recomendação principal é tratar a gestão de desconto e retenção como problema de risco (sensibilidade a churn), e não como simples ajuste de preço.

A partir daqui, o caminho mais seguro é:
- segmentar contas por perfil operacional
- calibrar política de desconto por custo/complexidade
- evitar ações generalistas que aumentem churn no grupo concentrador
