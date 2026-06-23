APÊNDICE B — Medidas DAX criadas no modelo semântico

Apêndice — Medidas DAX descaracterizadas
As medidas abaixo foram descaracterizadas para fins acadêmicos. Os nomes das tabelas, colunas, grupos financeiros, categorias comerciais, programas internos e classificações originais foram substituídos por nomenclaturas genéricas, preservando a lógica de cálculo utilizada no modelo analítico.
Bloco 1 — Resultado financeiro e perdas gerenciais
Medida 1 — Valor de Receita Bruta
Valor Receita Bruta =
CALCULATE (
    SUM ( f_resultado_financeiro[valor_real] ),
    f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA"
)

Medida 2 — Meta de Receita Bruta
Meta Receita Bruta =
CALCULATE (
    SUM ( f_resultado_financeiro[valor_meta] ),
    f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA"
)

Medida 3 — Percentual de Atingimento da Receita Bruta
% Atingimento Receita Bruta =
DIVIDE (
    [Valor Receita Bruta],
    [Meta Receita Bruta],
    0
)

Medida 4 — Percentual de Atingimento da Receita Bruta Consolidada
% Atingimento Receita Bruta Consolidada =
VAR ReceitaConsolidada =
    CALCULATE (
        [Valor Receita Bruta],
        REMOVEFILTERS (
            d_hierarquia_operacional[cod_unidade],
            d_hierarquia_operacional[gestor_nivel_1],
            d_hierarquia_operacional[gestor_nivel_2]
        )
    )

VAR MetaConsolidada =
    CALCULATE (
        [Meta Receita Bruta],
        REMOVEFILTERS (
            d_hierarquia_operacional[cod_unidade],
            d_hierarquia_operacional[gestor_nivel_1],
            d_hierarquia_operacional[gestor_nivel_2]
        )
    )

RETURN
DIVIDE (
    ReceitaConsolidada,
    MetaConsolidada,
    0
)

Medida 5 — Receita Bruta Mês Anterior
Receita Bruta Mês Anterior =
CALCULATE (
    [Valor Receita Bruta],
    DATEADD ( d_calendario[data], -1, MONTH )
)

Medida 6 — Receita Bruta Ano Anterior
Receita Bruta Ano Anterior =
CALCULATE (
    [Valor Receita Bruta],
    DATEADD ( d_calendario[data], -1, YEAR )
)

Medida 7 — Variação Receita Bruta LM
Variação Receita Bruta LM =
VAR ValorAtual =
    [Valor Receita Bruta]

VAR ValorMesAnterior =
    [Receita Bruta Mês Anterior]

RETURN
DIVIDE (
    ValorAtual - ValorMesAnterior,
    ValorMesAnterior,
    0
)

Medida 8 — Variação Receita Bruta LY
Variação Receita Bruta LY =
VAR ValorAtual =
    [Valor Receita Bruta]

VAR ValorAnoAnterior =
    [Receita Bruta Ano Anterior]

RETURN
DIVIDE (
    ValorAtual - ValorAnoAnterior,
    ValorAnoAnterior,
    0
)

Medida 9 — Índice Comparativo Receita Bruta
Índice Comparativo Receita Bruta =
VAR MesesSelecionados =
    VALUES ( d_calendario[mes] )

VAR AnoAtual =
    MAX ( d_calendario[ano] )

VAR AnoComparacao =
    AnoAtual - 1

VAR ReceitaAnoAtual =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA",
        FILTER (
            ALL ( d_calendario ),
            d_calendario[ano] = AnoAtual
                && d_calendario[mes] IN MesesSelecionados
        )
    )

VAR ReceitaAnoComparacao =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA",
        FILTER (
            ALL ( d_calendario ),
            d_calendario[ano] = AnoComparacao
                && d_calendario[mes] IN MesesSelecionados
        )
    )

RETURN
DIVIDE (
    ReceitaAnoAtual,
    ReceitaAnoComparacao,
    0
)

Medida 10 — Índice Comparativo Despesas
Índice Comparativo Despesas =
VAR MesesSelecionados =
    VALUES ( d_calendario[mes] )

VAR AnoAtual =
    MAX ( d_calendario[ano] )

VAR AnoComparacao =
    AnoAtual - 1

VAR DespesaAnoAtual =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS",
        FILTER (
            ALL ( d_calendario ),
            d_calendario[ano] = AnoAtual
                && d_calendario[mes] IN MesesSelecionados
        )
    )

VAR DespesaAnoComparacao =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS",
        FILTER (
            ALL ( d_calendario ),
            d_calendario[ano] = AnoComparacao
                && d_calendario[mes] IN MesesSelecionados
        )
    )

RETURN
DIVIDE (
    DespesaAnoAtual,
    DespesaAnoComparacao,
    0
)

Medida 11 — Amplitude Receita x Despesas
Amplitude Receita x Despesas =
VAR Resultado =
    (
        [Índice Comparativo Despesas]
        - [Índice Comparativo Receita Bruta]
    ) * -1

RETURN
IF (
    ISBLANK ( Resultado ),
    0,
    Resultado
)

Medida 12 — Amplitude Mês Anterior
Amplitude Mês Anterior =
VAR Resultado =
    (
        [Índice Comparativo Despesas Mês Anterior]
        - [Índice Comparativo Receita Bruta Mês Anterior]
    ) * -1

RETURN
IF (
    ISBLANK ( Resultado ),
    0,
    Resultado
)

Medida 13 — Variação Amplitude pp
Variação Amplitude pp =
(
    [Amplitude Receita x Despesas]
    - [Amplitude Mês Anterior]
) * 100

Medida 14 — Amplitude Formatada pp
Amplitude Formatada pp =
IF (
    [Amplitude Receita x Despesas] = 0,
    "0 p.p.",
    FORMAT ( [Amplitude Receita x Despesas] * 100, "0.00" ) & " p.p."
)

Medida 15 — Margem Bruta
Margem Bruta =
VAR ResultadoBruto =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] IN {
            "RECEITA_BRUTA",
            "DEDUCOES_RECEITA",
            "CUSTO_MERCADORIA"
        }
    )

VAR ReceitaBruta =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA"
    )

RETURN
DIVIDE (
    ResultadoBruto,
    ReceitaBruta,
    0
)

Medida 16 — Margem Bruta Mês Anterior
Margem Bruta Mês Anterior =
CALCULATE (
    [Margem Bruta],
    DATEADD ( d_calendario[data], -1, MONTH )
)

Medida 17 — Margem Bruta Ano Anterior
Margem Bruta Ano Anterior =
CALCULATE (
    [Margem Bruta],
    DATEADD ( d_calendario[data], -1, YEAR )
)

Medida 18 — Variação Margem Bruta LM
Variação Margem Bruta LM =
DIVIDE (
    [Margem Bruta] - [Margem Bruta Mês Anterior],
    [Margem Bruta Mês Anterior],
    0
)

Medida 19 — Variação Margem Bruta LY
Variação Margem Bruta LY =
DIVIDE (
    [Margem Bruta] - [Margem Bruta Ano Anterior],
    [Margem Bruta Ano Anterior],
    0
)

Medida 20 — Resultado Operacional Ajustado
Resultado Operacional Ajustado =
VAR ReceitaBruta =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA"
    )

VAR DeducoesReceita =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "DEDUCOES_RECEITA"
    )

VAR CustoMercadoria =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "CUSTO_MERCADORIA"
    )

VAR DespesasOperacionais =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS"
    )

RETURN
    ReceitaBruta
    + DeducoesReceita
    + CustoMercadoria
    + DespesasOperacionais

Medida 21 — Meta Resultado Operacional Ajustado
Meta Resultado Operacional Ajustado =
VAR ReceitaBruta =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_meta] ),
        f_resultado_financeiro[grupo_financeiro] = "RECEITA_BRUTA"
    )

VAR DeducoesReceita =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_meta] ),
        f_resultado_financeiro[grupo_financeiro] = "DEDUCOES_RECEITA"
    )

VAR CustoMercadoria =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_meta] ),
        f_resultado_financeiro[grupo_financeiro] = "CUSTO_MERCADORIA"
    )

VAR DespesasOperacionais =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_meta] ),
        f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS"
    )

RETURN
    ReceitaBruta
    + DeducoesReceita
    + CustoMercadoria
    + DespesasOperacionais

Medida 22 — Percentual de Atingimento Resultado Operacional
% Atingimento Resultado Operacional =
DIVIDE (
    [Resultado Operacional Ajustado],
    [Meta Resultado Operacional Ajustado],
    0
)

Medida 23 — Resultado Operacional Ano Anterior
Resultado Operacional Ano Anterior =
CALCULATE (
    [Resultado Operacional Ajustado],
    DATEADD ( d_calendario[data], -1, YEAR )
)

Medida 24 — Variação Resultado Operacional LY
Variação Resultado Operacional LY =
DIVIDE (
    [Resultado Operacional Ajustado] - [Resultado Operacional Ano Anterior],
    [Resultado Operacional Ano Anterior],
    0
)

Medida 25 — Despesas Operacionais Totais
Despesas Operacionais Totais =
VAR ValorDespesa =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS"
    )

RETURN
ABS ( ValorDespesa )

Medida 26 — Despesas Operacionais Mês Anterior
Despesas Operacionais Mês Anterior =
CALCULATE (
    [Despesas Operacionais Totais],
    DATEADD ( d_calendario[data], -1, MONTH )
)

Medida 27 — Despesas Operacionais Ano Anterior
Despesas Operacionais Ano Anterior =
CALCULATE (
    [Despesas Operacionais Totais],
    DATEADD ( d_calendario[data], -1, YEAR )
)

Medida 28 — Variação Despesas Operacionais LM
Variação Despesas Operacionais LM =
DIVIDE (
    [Despesas Operacionais Totais] - [Despesas Operacionais Mês Anterior],
    [Despesas Operacionais Mês Anterior],
    0
)

Medida 29 — Variação Despesas Operacionais LY
Variação Despesas Operacionais LY =
DIVIDE (
    [Despesas Operacionais Totais] - [Despesas Operacionais Ano Anterior],
    [Despesas Operacionais Ano Anterior],
    0
)

Medida 30 — Percentual Despesas sobre Receita
% Despesas sobre Receita =
VAR ReceitaLiquida =
    CALCULATE (
        SUM ( f_resultado_financeiro[valor_real] ),
        f_resultado_financeiro[grupo_financeiro] IN {
            "RECEITA_BRUTA",
            "DEDUCOES_RECEITA"
        }
    )

VAR DespesasOperacionais =
    ABS (
        CALCULATE (
            SUM ( f_resultado_financeiro[valor_real] ),
            f_resultado_financeiro[grupo_financeiro] = "DESPESAS_OPERACIONAIS"
        )
    )

RETURN
DIVIDE (
    DespesasOperacionais,
    ReceitaLiquida,
    0
)

Medida 31 — Percentual Despesas sobre Receita Mês Anterior
% Despesas sobre Receita Mês Anterior =
CALCULATE (
    [% Despesas sobre Receita],
    DATEADD ( d_calendario[data], -1, MONTH )
)

Medida 32 — Percentual Despesas sobre Receita Ano Anterior
% Despesas sobre Receita Ano Anterior =
CALCULATE (
    [% Despesas sobre Receita],
    DATEADD ( d_calendario[data], -1, YEAR )
)

Medida 33 — Variação Percentual Despesas sobre Receita LM
Variação % Despesas sobre Receita LM =
DIVIDE (
    [% Despesas sobre Receita] - [% Despesas sobre Receita Mês Anterior],
    [% Despesas sobre Receita Mês Anterior],
    0
)

Medida 34 — Variação Percentual Despesas sobre Receita LY
Variação % Despesas sobre Receita LY =
DIVIDE (
    [% Despesas sobre Receita] - [% Despesas sobre Receita Ano Anterior],
    [% Despesas sobre Receita Ano Anterior],
    0
)
