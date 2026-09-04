# Prompt para recriar o notebook de forecast sazonal híbrido

Crie um notebook Jupyter em Python que leia uma base mensal de empresas e gere forecast mensal até dezembro do ano seguinte ao último realizado.

## Entrada

- Arquivo Excel ou CSV.
- Colunas configuráveis no início:
  - `anomes`: competência mensal;
  - `empresa`: identificador da empresa;
  - `qtd`: valor a projetar.
- Aceitar datas nos formatos `YYYY-MM`, `YYYYMM` ou datas reconhecidas pelo pandas.
- Converter as datas para o primeiro dia do mês.
- Somar linhas duplicadas por empresa e mês.

## Regras do modelo

1. A projeção não deve ser recursiva mês a mês. Primeiro estime o total anual e depois distribua-o pelos meses conforme a sazonalidade.

2. Detecte automaticamente mudança de regime separadamente para cada empresa:
   - ordene a série mensal;
   - se houver menos de 8 observações, não crie quebra;
   - aplique `log1p` aos valores;
   - compare um modelo de uma média com modelos de duas médias separados por cada corte possível;
   - exija em cada lado no mínimo `max(3, ceil(n/4))` observações;
   - use `BIC = n*ln(SSE/n) + k*ln(n)`;
   - use `k=2` no modelo sem quebra e `k=4` no modelo com quebra;
   - aceite o melhor corte somente quando `BIC_com_quebra < BIC_sem_quebra`;
   - para nível e crescimento, use somente os dados a partir da quebra. Se não houver quebra, use toda a série.

3. Calcule a sazonalidade global:
   - use somente anos com os 12 meses observados;
   - em cada empresa/ano, calcule `participacao_mes = valor_mes / total_ano`;
   - para cada mês, use a mediana das participações;
   - preencha meses ausentes com `1/12` e normalize os 12 pesos para somarem 1.

4. Calcule a sazonalidade individual no regime atual:
   - use somente anos completos dentro do regime atual;
   - obtenha a mediana da participação de cada mês;
   - combine sazonalidade individual e global com peso automático `n_anos_individuais / (n_anos_individuais + referencia)`, em que `referencia` é a mediana da quantidade de anos completos disponíveis entre as empresas e tem mínimo 1;
   - se não houver histórico individual completo, use a sazonalidade global;
   - normalize o resultado para os 12 meses somarem 1.

5. Calcule comparações YoY válidas:
   - compare apenas o mesmo mês contra o mesmo mês do ano anterior;
   - aceite somente valores positivos;
   - o ano-base anterior deve conter os 12 meses dentro do regime atual;
   - selecione o ano de comparação mais recente;
   - calcule o fator individual por volume: `soma_volume_atual / soma_volume_base` nos meses comparáveis;
   - crescimento individual = fator individual menos 1.

6. Calcule o crescimento global ponderado por volume:
   - use apenas as comparações individuais válidas pertencentes ao ano de comparação mais recente encontrado no grupo;
   - `fator_global = soma_volumes_atuais / soma_volumes_base`;
   - crescimento global = fator global menos 1;
   - se não houver comparação válida, use crescimento global zero.

7. Crescimento usado no ano corrente:
   - representatividade sazonal = soma dos pesos sazonais finais dos meses usados na comparação individual;
   - `crescimento_final = representatividade_sazonal * crescimento_individual + (1 - representatividade_sazonal) * crescimento_global`;
   - se não houver comparação individual válida, use somente o crescimento global.

8. Crescimento estrutural usado no ano seguinte:
   - volume típico do grupo = mediana dos volumes-base comparáveis válidos;
   - representatividade de volume = `volume_base_empresa / (volume_base_empresa + volume_tipico_grupo)`;
   - confiança estrutural = representatividade sazonal vezes representatividade de volume;
   - `crescimento_estrutural_futuro = confianca_estrutural * crescimento_individual + (1 - confianca_estrutural) * crescimento_global`;
   - se não houver comparação individual válida, use somente o crescimento global.

9. Estime o total do último ano observado para cada empresa:
   - realizado acumulado = soma dos meses observados desse ano dentro do regime atual;
   - peso observado = soma dos pesos sazonais desses meses;
   - total implícito sazonal = realizado acumulado dividido pelo peso observado;
   - aceite o total do ano anterior como base somente se seus 12 meses estiverem dentro do regime atual;
   - quando a base anterior for válida, calcule `target_crescimento = total_ano_anterior * (1 + crescimento_final)`;
   - combine: `total_estimado = peso_observado * total_implicito_sazonal + (1 - peso_observado) * target_crescimento`;
   - sem base anterior válida, use apenas o total implícito sazonal;
   - o total estimado nunca pode ser menor que o realizado acumulado.

10. Complete o ano corrente:
    - saldo = total anual estimado menos realizado acumulado;
    - distribua o saldo somente nos meses futuros do ano corrente;
    - renormalize os pesos sazonais apenas entre esses meses restantes.

11. Projete o ano seguinte por empresa:
    - `total_bruto_ano_seguinte = total_estimado_ano_corrente * (1 + crescimento_estrutural_futuro)`;
    - limite o total anual ao mínimo zero;
    - distribua o total pelos 12 pesos sazonais finais da empresa.

12. Aplique uma âncora de incremento absoluto no primeiro ano futuro completo:
    - agregue todo o portfólio por mês;
    - use somente anos do portfólio com os 12 meses;
    - calcule diferenças absolutas entre anos completos consecutivos;
    - se o último ano for parcial e houver o ano anterior completo, inclua também `total_portfolio_ano_corrente_estimado - total_portfolio_ano_anterior_realizado`;
    - não duplique esse incremento se o ano corrente já for completo;
    - incremento sustentável = mediana dos incrementos válidos;
    - total-alvo do ano seguinte = total estimado do portfólio no ano corrente mais incremento sustentável, limitado ao mínimo zero;
    - fator de ajuste = total-alvo dividido pelo total bruto projetado do ano seguinte;
    - multiplique todos os forecasts de todas as empresas e meses do ano seguinte pelo mesmo fator, preservando sazonalidade e participações relativas;
    - se faltarem dados para a âncora, use fator 1.

13. Faça backtest walk-forward:
    - crie cortes mensais começando após pelo menos 12 meses de histórico e terminando antes do último realizado;
    - em cada corte, treine apenas com dados até o corte e projete o futuro disponível;
    - compare projetado e realizado;
    - calcule erro, erro absoluto, APE, MAE, RMSE, MAPE, WMAPE e BIAS;
    - gere métricas gerais, por empresa, horizonte e corte.

## Saídas

Gere `resultado_forecast_sazonal.xlsx` com as abas:

- `realizado_projetado`;
- `forecast_detalhado`;
- `resumo_anual`;
- `sazonalidade`;
- `crescimento`;
- `estimativa_ano_atual`;
- `crescimento_2027` ou nome dinâmico equivalente ao ano futuro;
- `ancora_incrementos`;
- `ancora_resumo`;
- `backtest_resumo`, quando houver dados.

Gere também `backtest_treino_realizado.xlsx` com:

- `treino_resumo`;
- `realizado_vs_projetado`;
- `metricas_gerais`;
- `metricas_empresa`;
- `metricas_horizonte`;
- `metricas_corte`.

Inclua no notebook tabelas de auditoria para mudança de regime, pares YoY válidos, crescimento individual/global, nível anual estimado e aplicação da âncora. Formate os arquivos Excel com cabeçalho, filtro, congelamento da primeira linha e ajuste de largura das colunas. Use `pandas`, `numpy`, `matplotlib` e `openpyxl`, organize o código em funções e inclua comentários curtos.
