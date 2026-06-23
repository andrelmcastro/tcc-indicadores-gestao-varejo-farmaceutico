APÊNDICE A — Consultas SQL utilizadas na construção das bases analíticas

Consulta 1 — Consolidação Mensal de Vendas por Categoria e Programa de Benefícios
Objetivo:
Consolidar, por unidade operacional, categoria de produto e mês de referência, os volumes de itens vendidos e os valores de venda bruta, comparando os três meses mais recentes do ano atual com os mesmos meses do ano anterior.
WITH vendas_filtradas AS (
    SELECT
        cod_unidade,
        categoria_produto_n2,
        CAST(
            CONCAT(
                CAST(ano_referencia AS varchar), '-',
                LPAD(CAST(mes_referencia AS varchar), 2, '0'), '-01'
            ) AS DATE
        ) AS mes_ref,
        qtd_itens_total,
        qtd_itens_programa_beneficio,
        valor_venda_bruta_total,
        valor_venda_bruta_programa_beneficio
    FROM camada_dados.fato_vendas_programa_beneficio
    WHERE
        (
            CAST(ano_referencia AS BIGINT) = YEAR(CURRENT_DATE)
            AND CAST(mes_referencia AS BIGINT) IN (
                MONTH(CURRENT_DATE),
                MONTH(DATE_ADD('month', -1, CURRENT_DATE)),
                MONTH(DATE_ADD('month', -2, CURRENT_DATE))
            )
        )
        OR
        (
            CAST(ano_referencia AS BIGINT) = YEAR(DATE_ADD('year', -1, CURRENT_DATE))
            AND CAST(mes_referencia AS BIGINT) IN (
                MONTH(CURRENT_DATE),
                MONTH(DATE_ADD('month', -1, CURRENT_DATE)),
                MONTH(DATE_ADD('month', -2, CURRENT_DATE))
            )
        )
)

SELECT
    cod_unidade,
    categoria_produto_n2,
    mes_ref AS data_referencia_mes,
    SUM(qtd_itens_total) AS qtd_itens_total,
    SUM(qtd_itens_programa_beneficio) AS qtd_itens_programa_beneficio,
    SUM(valor_venda_bruta_total) AS valor_venda_bruta_total,
    SUM(valor_venda_bruta_programa_beneficio) AS valor_venda_bruta_programa_beneficio
FROM vendas_filtradas
GROUP BY
    cod_unidade,
    categoria_produto_n2,
    mes_ref
ORDER BY
    mes_ref DESC,
    cod_unidade,
    categoria_produto_n2;



Consulta 2 — Indicadores Mensais de Captação e Conversão por Tipo de Documento Comercial
Objetivo:
Consolidar, por unidade operacional, mês e tipo de documento classificado, os indicadores de captação, conversão e participação em transações comerciais. A consulta considera uma janela móvel dos últimos seis meses e compara o período atual com o mesmo intervalo do ano anterior.
WITH periodo_analise AS (
    SELECT
        DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE)) AS data_inicio_atual,
        CURRENT_DATE AS data_fim_atual,
        DATE_TRUNC('month', DATE_ADD('year', -1, DATE_ADD('month', -5, CURRENT_DATE))) AS data_inicio_anterior,
        DATE_ADD('year', -1, CURRENT_DATE) AS data_fim_anterior
),

documentos_unicos AS (
    SELECT
        CAST(data_registro AS DATE) AS data_referencia,
        cod_unidade,
        id_documento
    FROM camada_origem.fato_documentos_registrados, periodo_analise p
    WHERE 
        data_registro BETWEEN p.data_inicio_atual AND p.data_fim_atual
        OR data_registro BETWEEN p.data_inicio_anterior AND p.data_fim_anterior
),

documentos_classificados AS (
    SELECT
        d.data_referencia,
        d.cod_unidade,
        d.id_documento,
        CASE
            WHEN dp.classificacao_item IN ('A1', 'A2', 'A3') THEN '01_TIPO_A'
            WHEN dp.classificacao_item IN ('B1', 'B2') THEN '02_TIPO_B'
            WHEN dp.classificacao_item IN ('C1', 'C2', 'C3', 'C4', 'C5') THEN '03_TIPO_C'
            ELSE '99_NAO_CLASSIFICADO'
        END AS tipo_documento
    FROM documentos_unicos d
    JOIN camada_origem.fato_documento_item di
        ON d.id_documento = di.id_documento
    LEFT JOIN camada_dados.dim_item dp
        ON LOWER(di.descricao_item) = LOWER(dp.nome_item)
    WHERE dp.cod_item IS NOT NULL
),

documentos_deduplicados AS (
    SELECT
        data_referencia,
        cod_unidade,
        id_documento,
        MIN(tipo_documento) AS tipo_documento
    FROM documentos_classificados
    GROUP BY
        data_referencia,
        cod_unidade,
        id_documento
),

documentos_captados AS (
    SELECT
        DATE_TRUNC('month', data_referencia) AS mes_referencia,
        cod_unidade,
        tipo_documento,
        COUNT(DISTINCT id_documento) AS qtd_documentos_captados
    FROM documentos_deduplicados
    GROUP BY
        DATE_TRUNC('month', data_referencia),
        cod_unidade,
        tipo_documento
),

transacoes_base AS (
    SELECT
        CAST(v.data_movimento AS DATE) AS data_referencia,
        v.cod_unidade,
        v.id_transacao,
        dp.classificacao_item
    FROM camada_dados.fato_vendas v
    LEFT JOIN camada_dados.dim_item dp
        ON v.cod_item = dp.cod_item,
        periodo_analise p
    WHERE 
        (
            v.data_movimento BETWEEN p.data_inicio_atual AND p.data_fim_atual
            OR v.data_movimento BETWEEN p.data_inicio_anterior AND p.data_fim_anterior
        )
        AND dp.categoria_item_n2 = 'CATEGORIA_ALVO'
        AND v.cod_item IS NOT NULL
),

transacoes_classificadas AS (
    SELECT
        DATE_TRUNC('month', data_referencia) AS mes_referencia,
        cod_unidade,
        id_transacao,
        CASE
            WHEN classificacao_item IN ('A1', 'A2', 'A3') THEN '01_TIPO_A'
            WHEN classificacao_item IN ('B1', 'B2') THEN '02_TIPO_B'
            WHEN classificacao_item IN ('C1', 'C2', 'C3', 'C4', 'C5') THEN '03_TIPO_C'
            ELSE '99_NAO_CLASSIFICADO'
        END AS tipo_documento
    FROM transacoes_base
),

transacoes_deduplicadas AS (
    SELECT
        mes_referencia,
        cod_unidade,
        id_transacao,
        MIN(tipo_documento) AS tipo_documento
    FROM transacoes_classificadas
    GROUP BY
        mes_referencia,
        cod_unidade,
        id_transacao
),

transacoes_classificadas_agg AS (
    SELECT
        mes_referencia,
        cod_unidade,
        tipo_documento,
        COUNT(DISTINCT id_transacao) AS qtd_transacoes_categoria
    FROM transacoes_deduplicadas
    GROUP BY
        mes_referencia,
        cod_unidade,
        tipo_documento
),

documentos_convertidos AS (
    SELECT DISTINCT
        d.id_documento,
        d.data_referencia,
        d.cod_unidade,
        d.tipo_documento
    FROM documentos_deduplicados d
    JOIN camada_origem.fato_documento_item di
        ON d.id_documento = di.id_documento
    LEFT JOIN camada_dados.dim_item dp
        ON LOWER(di.descricao_item) = LOWER(dp.nome_item)
    JOIN camada_dados.fato_vendas v
        ON d.cod_unidade = v.cod_unidade
        AND CAST(v.data_movimento AS DATE) = d.data_referencia
        AND v.cod_item = dp.cod_item
    WHERE dp.cod_item IS NOT NULL
),

documentos_convertidos_agg AS (
    SELECT
        DATE_TRUNC('month', data_referencia) AS mes_referencia,
        cod_unidade,
        tipo_documento,
        COUNT(DISTINCT id_documento) AS qtd_documentos_convertidos
    FROM documentos_convertidos
    GROUP BY
        DATE_TRUNC('month', data_referencia),
        cod_unidade,
        tipo_documento
),

base_indicadores AS (
    SELECT
        COALESCE(dc.mes_referencia, tc.mes_referencia, dcv.mes_referencia) AS mes_referencia,
        COALESCE(dc.cod_unidade, tc.cod_unidade, dcv.cod_unidade) AS cod_unidade,
        COALESCE(dc.tipo_documento, tc.tipo_documento, dcv.tipo_documento) AS tipo_documento,
        COALESCE(dc.qtd_documentos_captados, 0) AS qtd_documentos_captados,
        COALESCE(tc.qtd_transacoes_categoria, 0) AS qtd_transacoes_categoria,
        COALESCE(dcv.qtd_documentos_convertidos, 0) AS qtd_documentos_convertidos
    FROM documentos_captados dc
    FULL OUTER JOIN transacoes_classificadas_agg tc
        ON dc.mes_referencia = tc.mes_referencia
        AND dc.tipo_documento = tc.tipo_documento
        AND dc.cod_unidade = tc.cod_unidade
    FULL OUTER JOIN documentos_convertidos_agg dcv
        ON COALESCE(dc.mes_referencia, tc.mes_referencia) = dcv.mes_referencia
        AND COALESCE(dc.tipo_documento, tc.tipo_documento) = dcv.tipo_documento
        AND COALESCE(dc.cod_unidade, tc.cod_unidade) = dcv.cod_unidade
)

SELECT
    CAST(DATE_FORMAT(mes_referencia, '%Y-%m-01') AS DATE) AS ano_mes,
    cod_unidade,
    tipo_documento,
    qtd_documentos_captados,
    qtd_documentos_convertidos,
    qtd_transacoes_categoria,
    CASE 
        WHEN qtd_transacoes_categoria = 0 THEN NULL
        ELSE ROUND(100.0 * qtd_documentos_captados / qtd_transacoes_categoria, 2)
    END AS pct_captacao,
    CASE 
        WHEN qtd_documentos_captados = 0 THEN NULL
        ELSE ROUND(100.0 * qtd_documentos_convertidos / qtd_documentos_captados, 2)
    END AS pct_conversao
FROM base_indicadores
ORDER BY
    ano_mes DESC,
    tipo_documento,
    cod_unidade;


Consulta 4 — Tratamento e Padronização de Metas por Unidade Operacional
Objetivo:
Padronizar o campo de data de uma base de metas operacionais, tratando diferentes formatos de entrada e mantendo apenas os registros com datas válidas. A consulta retorna as metas por unidade operacional em três faixas de desempenho.
SELECT
    cod_unidade,
    COALESCE(
        TRY(CAST(data_referencia AS DATE)),
        TRY(CAST(DATE_PARSE(data_referencia, '%d/%m/%Y') AS DATE))
    ) AS data_referencia_formatada,
    meta_faixa_70,
    meta_faixa_100,
    meta_faixa_120
FROM camada_dados.fato_metas_operacionais_unidade
WHERE COALESCE(
        TRY(CAST(data_referencia AS DATE)),
        TRY(CAST(DATE_PARSE(data_referencia, '%d/%m/%Y') AS DATE))
      ) IS NOT NULL;


Consulta 5 — Consolidação de Perdas Gerenciais por Unidade e Hierarquia Operacional
Objetivo:
Selecionar os dados realizados de perdas gerenciais por unidade operacional e níveis de gestão, considerando o período dos últimos seis meses e o mesmo intervalo do ano anterior. A consulta permite acompanhar a evolução das perdas totais, identificadas e não identificadas em comparação temporal.
SELECT
    cod_unidade,
    gestor_nivel_1,
    gestor_nivel_2,
    DATE_PARSE(data_referencia, '%Y-%m-%d') AS data_referencia,
    valor_receita_base,
    valor_perdas_total,
    valor_perdas_nao_identificadas,
    valor_perdas_identificadas
FROM camada_dados.fato_perdas_gerenciais_realizado
WHERE
    (
        CAST(
            DATE_FORMAT(
                DATE_PARSE(data_referencia, '%Y-%m-%d'),
                '%Y%m'
            ) AS INT
        ) BETWEEN
            CAST(
                FORMAT_DATETIME(
                    DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE)),
                    'yyyyMM'
                ) AS INT
            )
            AND
            CAST(
                FORMAT_DATETIME(
                    CURRENT_DATE,
                    'yyyyMM'
                ) AS INT
            )
    )
    OR
    (
        CAST(
            DATE_FORMAT(
                DATE_PARSE(data_referencia, '%Y-%m-%d'),
                '%Y%m'
            ) AS INT
        ) BETWEEN
            CAST(
                FORMAT_DATETIME(
                    DATE_ADD(
                        'year',
                        -1,
                        DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
                    ),
                    'yyyyMM'
                ) AS INT
            )
            AND
            CAST(
                FORMAT_DATETIME(
                    DATE_ADD('year', -1, CURRENT_DATE),
                    'yyyyMM'
                ) AS INT
            )
    );



Consulta 6 — Consolidação Mensal de Vendas e Descontos Identificados por Unidade
Objetivo:
Consolidar, por unidade operacional e mês de referência, os valores de venda bruta, descontos totais e descontos de uma modalidade específica, considerando o mês atual e os dois meses anteriores.
WITH dados_base AS (
    SELECT
        cod_unidade_origem,
        CAST(DATE_PARSE(data_movimento_origem, '%d/%m/%Y') AS DATE) AS data_pedido,
        valor_venda_bruta,
        valor_desconto_total,
        valor_desconto_modalidade
    FROM camada_dados.vw_descontos_identificados_unidade
)

SELECT
    cod_unidade_origem AS cod_unidade,
    DATE_TRUNC('month', data_pedido) AS data_referencia_mes,
    SUM(valor_venda_bruta) AS valor_venda_bruta,
    SUM(valor_desconto_total) AS valor_desconto_total,
    SUM(valor_desconto_modalidade) AS valor_desconto_modalidade
FROM dados_base
WHERE data_pedido >= DATE_TRUNC('month', DATE_ADD('month', -2, CURRENT_DATE))
  AND data_pedido < DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE))
GROUP BY
    cod_unidade_origem,
    DATE_TRUNC('month', data_pedido)
ORDER BY
    data_referencia_mes DESC,
    cod_unidade;



Consulta 7 — Extração Geral de Indicadores de Eficiência Operacional
Objetivo:
Extrair a base consolidada de indicadores operacionais utilizada para análise de eficiência das unidades monitoradas. A consulta serve como origem para o tratamento, modelagem e construção dos indicadores no ambiente analítico.
SELECT
    *
FROM camada_dados.vw_indicadores_eficiencia_operacional;


Consulta 8 — Extração da Base de Unidades Monitoradas
Objetivo:
Extrair a base cadastral e hierárquica das unidades acompanhadas no projeto, contendo informações necessárias para segmentação, filtros e cruzamento com os indicadores operacionais e financeiros.
SELECT
    *
FROM camada_dados.vw_base_unidades_monitoradas;


Consulta 9 — Extração de Dados Financeiros Tratados por Unidade Operacional
Objetivo:
Extrair a base financeira tratada para análise dos indicadores econômicos por unidade operacional, considerando uma janela móvel dos últimos cinco meses e o mesmo período do ano anterior. A consulta também valida se o identificador da unidade está em formato numérico antes de convertê-lo para código de unidade.
WITH periodo_atual AS (
    SELECT
        DATE_TRUNC('month', DATE_ADD('month', -4, CURRENT_DATE)) AS data_inicio_atual,
        CURRENT_DATE AS data_fim_atual
),

periodo_anterior AS (
    SELECT
        DATE_ADD('year', -1, data_inicio_atual) AS data_inicio_anterior,
        DATE_ADD('year', -1, data_fim_atual) AS data_fim_anterior
    FROM periodo_atual
)

SELECT
    t.*,
    CAST(t.identificador_unidade_tratado AS INTEGER) AS cod_unidade
FROM camada_dados.fato_resultado_financeiro_tratado t
JOIN periodo_atual pa
    ON t.data_referencia BETWEEN pa.data_inicio_atual AND pa.data_fim_atual
    OR t.data_referencia BETWEEN
        (
            SELECT data_inicio_anterior
            FROM periodo_anterior
        )
        AND
        (
            SELECT data_fim_anterior
            FROM periodo_anterior
        )
WHERE REGEXP_LIKE(t.identificador_unidade_tratado, '^[0-9]+$');


Consulta 10 — Consolidação de Atendimentos Identificados e Ações Comerciais por Unidade
Objetivo:
Consolidar, por unidade operacional, mês, semana e hierarquia de gestão, os volumes de atendimentos totais, atendimentos identificados, impressões de ação comercial e compras vinculadas. A consulta considera uma janela móvel dos últimos seis meses e o mesmo intervalo do ano anterior, restringindo a análise a unidades ativas e canais comerciais selecionados.
WITH periodo_analise AS (
    SELECT
        DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE)) AS data_inicio_atual,
        CURRENT_DATE AS data_fim_atual,
        DATE_TRUNC('month', DATE_ADD('year', -1, DATE_ADD('month', -5, CURRENT_DATE))) AS data_inicio_anterior,
        DATE_ADD('year', -1, CURRENT_DATE) AS data_fim_anterior
),

atendimentos AS (
    SELECT
        CAST(v.data_movimento AS DATE) AS data_movimento,
        v.ano_referencia,
        v.mes_referencia,
        CAST(WEEK(v.data_movimento) AS VARCHAR) AS semana_referencia,
        v.cod_unidade,
        COUNT(DISTINCT v.id_transacao) AS qtd_atendimentos_total,
        COUNT(
            DISTINCT CASE 
                WHEN v.id_cliente IS NOT NULL 
                THEN v.id_transacao 
            END
        ) AS qtd_atendimentos_identificados
    FROM camada_dados.fato_movimento_vendas_cliente v
    CROSS JOIN periodo_analise p
    WHERE v.id_transacao IS NOT NULL
      AND (
            v.data_movimento BETWEEN p.data_inicio_atual AND p.data_fim_atual
            OR v.data_movimento BETWEEN p.data_inicio_anterior AND p.data_fim_anterior
          )
      AND v.cod_unidade NOT IN (9991, 9992, 9993)
      AND v.canal_comercial IN ('CANAL_PRESENCIAL', 'CANAL_ASSISTIDO')
    GROUP BY
        v.ano_referencia,
        v.mes_referencia,
        CAST(WEEK(v.data_movimento) AS VARCHAR),
        v.cod_unidade,
        CAST(v.data_movimento AS DATE)
),

acoes_comerciais AS (
    SELECT
        CAST(a.data_movimento AS DATE) AS data_movimento,
        a.cod_unidade,
        SUM(a.qtd_clientes_compra) AS qtd_clientes_compra,
        SUM(a.qtd_clientes_compra_acao) AS qtd_clientes_compra_acao,
        SUM(a.qtd_clientes_impressao) AS qtd_clientes_impressao
    FROM camada_dados.fato_acoes_comerciais_cliente a
    CROSS JOIN periodo_analise p
    WHERE (
            a.data_movimento BETWEEN p.data_inicio_atual AND p.data_fim_atual
            OR a.data_movimento BETWEEN p.data_inicio_anterior AND p.data_fim_anterior
          )
      AND a.cod_unidade NOT IN (9991, 9992, 9993)
      AND a.canal_comercial IN ('CANAL_PRESENCIAL', 'CANAL_ASSISTIDO')
    GROUP BY
        CAST(a.data_movimento AS DATE),
        a.cod_unidade
)

SELECT
    h.gestor_nivel_1,
    h.gestor_nivel_2,
    at.cod_unidade,
    at.ano_referencia,
    at.mes_referencia,
    at.semana_referencia,
    CAST(
        DATE_PARSE(
            CONCAT(
                CAST(at.ano_referencia AS VARCHAR),
                '-',
                LPAD(CAST(at.mes_referencia AS VARCHAR), 2, '0'),
                '-01'
            ),
            '%Y-%m-%d'
        ) AS DATE
    ) AS data_referencia_mes,
    at.qtd_atendimentos_total,
    at.qtd_atendimentos_identificados,
    COALESCE(ac.qtd_clientes_impressao, 0) AS qtd_impressoes,
    COALESCE(ac.qtd_clientes_compra_acao, 0) AS qtd_compras_acao,
    COALESCE(ac.qtd_clientes_compra, 0) AS qtd_compras_total
FROM atendimentos at
LEFT JOIN acoes_comerciais ac
    ON at.cod_unidade = ac.cod_unidade
    AND at.data_movimento = ac.data_movimento
LEFT JOIN camada_dados.dim_hierarquia_operacional_atual h
    ON at.cod_unidade = CAST(h.cod_unidade AS INT)
WHERE h.status_unidade = 'ATIVA'
ORDER BY
    at.ano_referencia,
    at.mes_referencia,
    at.data_movimento,
    h.gestor_nivel_1,
    h.gestor_nivel_2,
    at.cod_unidade;


Consulta 11 — Consolidação Mensal de Vendas, Quantidade e Transações por Unidade
Objetivo:
Consolidar, por unidade operacional e mês de referência, os principais indicadores comerciais: quantidade de transações, volume de itens vendidos e valor de venda bruta. A consulta considera uma janela móvel dos últimos seis meses e o mesmo intervalo do ano anterior.
WITH periodo_analise AS (
    SELECT
        DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE)) AS data_inicio_atual,
        CURRENT_DATE AS data_fim_atual,
        DATE_TRUNC('month', DATE_ADD('year', -1, DATE_ADD('month', -5, CURRENT_DATE))) AS data_inicio_anterior,
        DATE_ADD('year', -1, CURRENT_DATE) AS data_fim_anterior
)

SELECT
    CAST(DATE_TRUNC('month', v.data_movimento) AS DATE) AS mes_referencia,
    v.cod_unidade,
    COUNT(DISTINCT v.id_transacao) AS qtd_transacoes,
    SUM(v.qtd_itens) AS qtd_itens_vendidos,
    SUM(v.valor_venda_bruta) AS valor_venda_bruta
FROM camada_dados.fato_vendas v
CROSS JOIN periodo_analise p
WHERE (
          v.data_movimento BETWEEN p.data_inicio_atual AND p.data_fim_atual
       OR v.data_movimento BETWEEN p.data_inicio_anterior AND p.data_fim_anterior
      )
  AND v.id_transacao IS NOT NULL
GROUP BY
    CAST(DATE_TRUNC('month', v.data_movimento) AS DATE),
    v.cod_unidade
ORDER BY
    mes_referencia DESC,
    v.cod_unidade;

Consulta 12 — Consolidação Mensal de Descontos e Pedidos por Unidade Operacional
Objetivo:
Consolidar, por unidade operacional e mês de referência, o valor total de descontos aplicados e o valor total dos pedidos, considerando uma janela móvel dos últimos seis meses e o mesmo intervalo do ano anterior. A consulta considera apenas pedidos com status válido para análise.
WITH periodo_analise AS (
    SELECT
        DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE)) AS data_inicio_atual,
        DATE_TRUNC('month', CURRENT_DATE) AS data_fim_atual,
        DATE_TRUNC('month', DATE_ADD('year', -1, DATE_ADD('month', -5, CURRENT_DATE))) AS data_inicio_anterior,
        DATE_TRUNC('month', DATE_ADD('year', -1, CURRENT_DATE)) AS data_fim_anterior
)

SELECT
    f.cod_unidade,
    CAST(DATE_TRUNC('month', f.data_pedido) AS DATE) AS data_referencia_mes,
    SUM(f.valor_desconto_aplicado) AS valor_desconto_modalidade,
    SUM(f.valor_total_pedido) AS valor_total_pedido
FROM camada_dados.fato_pre_pedidos f
CROSS JOIN periodo_analise p
WHERE (
          f.data_pedido >= p.data_inicio_atual
      AND f.data_pedido < DATE_ADD('month', 1, p.data_fim_atual)
       OR f.data_pedido >= p.data_inicio_anterior
      AND f.data_pedido < DATE_ADD('month', 1, p.data_fim_anterior)
      )
  AND f.status_pedido IN ('STATUS_VALIDO_1', 'STATUS_VALIDO_2')
GROUP BY
    f.cod_unidade,
    CAST(DATE_TRUNC('month', f.data_pedido) AS DATE)
ORDER BY
    data_referencia_mes,
    f.cod_unidade;

Consulta 13 — Consolidação Mensal de Vendas, Clientes Únicos e Transações por Unidade Ativa
Objetivo:
Consolidar, por unidade operacional ativa e mês de referência, os valores de venda bruta, a quantidade de clientes únicos identificados e o total de transações realizadas. A consulta considera uma janela móvel dos últimos seis meses e o mesmo período do ano anterior.
WITH base_vendas AS (
    SELECT
        CAST(v.cod_unidade AS VARCHAR) AS cod_unidade,
        YEAR(v.data_movimento) AS ano_referencia,
        MONTH(v.data_movimento) AS mes_referencia,
        v.valor_venda_bruta,
        v.id_cliente,
        v.id_transacao,
        CAST(DATE_TRUNC('month', v.data_movimento) AS DATE) AS data_referencia_mes
    FROM camada_dados.fato_vendas v
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON CAST(v.cod_unidade AS VARCHAR) = h.cod_unidade
        AND h.status_unidade = 'ATIVA'
    WHERE (
        (
            v.data_movimento >= DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE))
            AND v.data_movimento < DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE))
        )
        OR
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE))
            )
            AND v.data_movimento < DATE_ADD(
                'year',
                -1,
                DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE))
            )
        )
    )
)

SELECT
    cod_unidade,
    ano_referencia,
    mes_referencia,
    data_referencia_mes,
    SUM(valor_venda_bruta) AS valor_venda_bruta,
    COUNT(DISTINCT id_cliente) AS qtd_clientes_unicos,
    COUNT(DISTINCT id_transacao) AS qtd_transacoes
FROM base_vendas
GROUP BY
    cod_unidade,
    ano_referencia,
    mes_referencia,
    data_referencia_mes
ORDER BY
    data_referencia_mes DESC,
    cod_unidade;

Consulta 14 — Consolidação Mensal de Indicador de Competitividade de Preços por Categoria e Região
Objetivo:
Consolidar, por mês, unidade federativa, categoria de produto e responsável regional, os valores de preços internos e preços de mercado coletados em pesquisas comerciais. A consulta permite apoiar a análise comparativa de competitividade de preços entre a organização e o mercado.
WITH base_pesquisas AS (
    WITH analise_precos AS (
        SELECT
            CAST(
                DATE_TRUNC('month', CAST(data_pesquisa AS DATE)) 
                AS DATE
            ) AS data_referencia_mes,
            unidade_federativa,
            categoria_produto_n2,
            CAST(preco_interno AS DECIMAL(10,2)) AS preco_interno,
            CAST(preco_mercado AS DECIMAL(10,2)) AS preco_mercado
        FROM camada_dados.fato_pesquisa_precos
        WHERE categoria_produto_n1 IN ('CATEGORIA_GRUPO_1', 'CATEGORIA_GRUPO_2')
          AND categoria_produto_n2 <> 'NAO_CLASSIFICADO'
          AND status_pesquisa = 'STATUS_VALIDO'
    )

    SELECT
        data_referencia_mes,
        unidade_federativa,
        categoria_produto_n2,
        SUM(preco_interno) AS valor_preco_interno,
        SUM(preco_mercado) AS valor_preco_mercado
    FROM analise_precos
    GROUP BY
        data_referencia_mes,
        unidade_federativa,
        categoria_produto_n2
),

base_indicador_competitividade AS (
    SELECT
        p.data_referencia_mes,
        p.unidade_federativa,
        p.categoria_produto_n2,
        h.gestor_nivel_1,
        p.valor_preco_interno,
        p.valor_preco_mercado
    FROM base_pesquisas p
    LEFT JOIN camada_dados.dim_hierarquia_regional h
        ON p.unidade_federativa = h.unidade_federativa
)

SELECT
    data_referencia_mes,
    unidade_federativa,
    categoria_produto_n2,
    gestor_nivel_1,
    SUM(valor_preco_interno) AS valor_preco_interno,
    SUM(valor_preco_mercado) AS valor_preco_mercado
FROM base_indicador_competitividade
GROUP BY
    data_referencia_mes,
    unidade_federativa,
    categoria_produto_n2,
    gestor_nivel_1
ORDER BY
    data_referencia_mes,
    unidade_federativa,
    categoria_produto_n2,
    gestor_nivel_1;


Consulta 15 — Base Híbrida de Vendas Realizadas e Estimadas por Unidade, Canal e Categoria
Objetivo:
Consolidar uma base de vendas por unidade operacional, canal comercial e categoria de produto, combinando vendas realizadas com valores estimados. A consulta utiliza uma lógica híbrida de data: mantém a granularidade diária para o mês atual e consolida os meses anteriores no primeiro dia de cada mês.
WITH vendas_realizadas AS (
    SELECT 
        v.cod_unidade,
        CAST(v.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN v.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        p.categoria_produto_n2,
        p.categoria_produto_n1,
        v.valor_venda_bruta AS valor_venda,
        v.qtd_itens AS qtd_vendida,
        v.id_transacao AS id_transacao
    FROM camada_dados.fato_vendas_realizadas v
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON v.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON v.cod_produto = p.cod_produto
    WHERE 
      (
        (
            v.data_movimento >= DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE)) 
            AND v.data_movimento < CURRENT_DATE
        )
        OR 
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
            )
            AND v.data_movimento < DATE_ADD('year', -1, CURRENT_DATE)
        )
      )
),

vendas_estimadas AS (
    SELECT 
        e.cod_unidade,
        CAST(e.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN e.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        p.categoria_produto_n1,
        p.categoria_produto_n2,
        e.valor_venda_bruta_estimado AS valor_venda,
        e.qtd_itens_estimado AS qtd_vendida,
        e.id_transacao_estimado AS id_transacao
    FROM camada_dados.fato_vendas_estimadas e
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON e.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON e.cod_produto = p.cod_produto 
    WHERE 
        e.data_movimento >= DATE_TRUNC('month', DATE_ADD('month', -1, CURRENT_DATE))
        AND e.data_movimento < DATE_TRUNC('month', DATE_ADD('month', 2, CURRENT_DATE))
),

base_vendas_unificada AS (
    SELECT * FROM vendas_realizadas
    UNION ALL
    SELECT * FROM vendas_estimadas
),

base_com_data_hibrida AS (
    SELECT
        *,
        CASE
            WHEN data_dia >= DATE_TRUNC('month', CURRENT_DATE)
                THEN data_dia
            ELSE CAST(DATE_TRUNC('month', data_dia) AS DATE)
        END AS data_agrupamento
    FROM base_vendas_unificada
)

SELECT 
    cod_unidade,
    data_agrupamento AS data_referencia_final,
    CASE 
        WHEN GROUPING(tipo_canal) = 1 THEN 'TOTAL_UNIDADE'
        ELSE tipo_canal 
    END AS tipo_canal,
    categoria_produto_n2,
    categoria_produto_n1,
    SUM(valor_venda) AS valor_venda,
    SUM(qtd_vendida) AS qtd_itens_vendidos,
    COUNT(DISTINCT id_transacao) AS qtd_transacoes_distintas
FROM base_com_data_hibrida
GROUP BY GROUPING SETS (
    (
        cod_unidade,
        data_agrupamento,
        tipo_canal,
        categoria_produto_n2,
        categoria_produto_n1
    ),
    (
        cod_unidade,
        data_agrupamento,
        categoria_produto_n2,
        categoria_produto_n1
    )
);



Consulta 16 — Base Híbrida de Vendas de Categoria Estratégica por Unidade e Canal
Objetivo:
Consolidar uma base de vendas realizadas e estimadas para uma categoria estratégica de produtos, desconsiderando itens de marca própria. A consulta permite analisar venda, quantidade e transações distintas por unidade operacional, canal comercial e período, utilizando granularidade diária no mês atual e mensal para períodos históricos.
WITH vendas_realizadas AS (
    SELECT 
        v.cod_unidade,
        CAST(v.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN v.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        v.valor_venda_bruta AS valor_venda,
        v.qtd_itens AS qtd_vendida,
        v.id_transacao AS id_transacao
    FROM camada_dados.fato_vendas_realizadas v
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON v.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    WHERE 
      (
        (
            v.data_movimento >= DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE)) 
            AND v.data_movimento < CURRENT_DATE
        )
        OR 
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
            )
            AND v.data_movimento < DATE_ADD('year', -1, CURRENT_DATE)
        )
      )
      AND v.categoria_produto_n5 IN ('CATEGORIA_ALVO_1', 'CATEGORIA_ALVO_2')
      AND v.indicador_marca_propria <> 'S'
),

vendas_estimadas AS (
    SELECT 
        e.cod_unidade,
        CAST(e.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN e.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        e.valor_venda_bruta_estimado AS valor_venda,
        e.qtd_itens_estimado AS qtd_vendida,
        e.id_transacao_estimado AS id_transacao
    FROM camada_dados.fato_vendas_estimadas e
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON e.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON e.cod_produto = p.cod_produto 
    WHERE 
        e.data_movimento >= DATE_TRUNC('month', DATE_ADD('month', -1, CURRENT_DATE))
        AND e.data_movimento < DATE_TRUNC('month', DATE_ADD('month', 2, CURRENT_DATE))
        AND p.categoria_produto_n5 IN ('CATEGORIA_ALVO_1', 'CATEGORIA_ALVO_2')
        AND p.indicador_marca_propria <> 'S'
),

base_vendas_unificada AS (
    SELECT * FROM vendas_realizadas
    UNION ALL
    SELECT * FROM vendas_estimadas
),

base_com_data_hibrida AS (
    SELECT
        *,
        CASE
            WHEN data_dia >= DATE_TRUNC('month', CURRENT_DATE)
                THEN data_dia
            ELSE CAST(DATE_TRUNC('month', data_dia) AS DATE)
        END AS data_agrupamento
    FROM base_vendas_unificada
)

SELECT 
    cod_unidade,
    data_agrupamento AS data_referencia,
    CASE 
        WHEN GROUPING(tipo_canal) = 1 THEN 'TOTAL_UNIDADE'
        ELSE tipo_canal 
    END AS tipo_canal,
    SUM(valor_venda) AS valor_venda,
    SUM(qtd_vendida) AS qtd_itens_vendidos,
    COUNT(DISTINCT id_transacao) AS qtd_transacoes_distintas
FROM base_com_data_hibrida
GROUP BY GROUPING SETS (
    (cod_unidade, data_agrupamento, tipo_canal),
    (cod_unidade, data_agrupamento)
);

Consulta 17 — Base Híbrida de Vendas de Produtos de Marca Própria por Unidade e Canal
Objetivo:
Consolidar uma base de vendas realizadas e estimadas para produtos classificados como marca própria, permitindo acompanhar venda, quantidade e transações distintas por unidade operacional, canal comercial e período. A consulta utiliza granularidade diária para o mês atual e consolidação mensal para os períodos históricos.
WITH vendas_realizadas AS (
    SELECT 
        v.cod_unidade,
        CAST(v.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN v.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        v.valor_venda_bruta AS valor_venda,
        v.qtd_itens AS qtd_vendida,
        v.id_transacao AS id_transacao
    FROM camada_dados.fato_vendas_realizadas v
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON v.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    WHERE 
      (
        (
            v.data_movimento >= DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE)) 
            AND v.data_movimento < CURRENT_DATE
        )
        OR 
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
            )
            AND v.data_movimento < DATE_ADD('year', -1, CURRENT_DATE)
        )
      )
      AND v.indicador_marca_propria = 'S'
),

vendas_estimadas AS (
    SELECT 
        e.cod_unidade,
        CAST(e.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN e.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        e.valor_venda_bruta_estimado AS valor_venda,
        e.qtd_itens_estimado AS qtd_vendida,
        e.id_transacao_estimado AS id_transacao
    FROM camada_dados.fato_vendas_estimadas e
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON e.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON e.cod_produto = p.cod_produto 
    WHERE 
        e.data_movimento >= DATE_TRUNC('month', DATE_ADD('month', -1, CURRENT_DATE))
        AND e.data_movimento < DATE_TRUNC('month', DATE_ADD('month', 2, CURRENT_DATE))
        AND p.indicador_marca_propria = 'S'
),

base_vendas_unificada AS (
    SELECT * FROM vendas_realizadas
    UNION ALL
    SELECT * FROM vendas_estimadas
),

base_com_data_hibrida AS (
    SELECT
        *,
        CASE
            WHEN data_dia >= DATE_TRUNC('month', CURRENT_DATE)
                THEN data_dia
            ELSE CAST(DATE_TRUNC('month', data_dia) AS DATE)
        END AS data_agrupamento
    FROM base_vendas_unificada
)

SELECT 
    cod_unidade,
    data_agrupamento AS data_referencia,
    CASE 
        WHEN GROUPING(tipo_canal) = 1 THEN 'TOTAL_UNIDADE'
        ELSE tipo_canal 
    END AS tipo_canal,
    SUM(valor_venda) AS valor_venda,
    SUM(qtd_vendida) AS qtd_itens_vendidos,
    COUNT(DISTINCT id_transacao) AS qtd_transacoes_distintas
FROM base_com_data_hibrida
GROUP BY GROUPING SETS (
    (cod_unidade, data_agrupamento, tipo_canal),
    (cod_unidade, data_agrupamento)
);



Consulta 18 — Base Híbrida de Vendas de Segmento Estratégico por Unidade, Canal e Subsegmento
Objetivo:
Consolidar uma base de vendas realizadas e estimadas para um segmento estratégico de produtos, classificando os itens em subsegmentos analíticos. A consulta permite acompanhar venda, quantidade e transações distintas por unidade operacional, canal comercial e período, com granularidade diária no mês atual e mensal para períodos históricos.
WITH vendas_realizadas AS (
    SELECT 
        v.cod_unidade,
        CAST(v.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN v.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        CASE 
            WHEN p.categoria_produto_n2 = 'CATEGORIA_SEGMENTO_A' 
                THEN 'SUBSEGMENTO_A'
            WHEN p.categoria_produto_n2 = 'CATEGORIA_SEGMENTO_B'
                 AND p.categoria_produto_n3 = 'SUBCATEGORIA_SEGMENTO_B'
                THEN 'SUBSEGMENTO_B'
        END AS segmento_estrategico,
        v.valor_venda_bruta AS valor_venda,
        v.qtd_itens AS qtd_vendida,
        v.id_transacao AS id_transacao
    FROM camada_dados.fato_vendas_realizadas v
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON v.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON v.cod_produto = p.cod_produto 
    WHERE 
      (
        (
            v.data_movimento >= DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE)) 
            AND v.data_movimento < CURRENT_DATE
        )
        OR 
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
            )
            AND v.data_movimento < DATE_ADD('year', -1, CURRENT_DATE)
        )
      )
),

vendas_estimadas AS (
    SELECT 
        e.cod_unidade,
        CAST(e.data_movimento AS DATE) AS data_dia,
        CASE 
            WHEN e.canal_comercial IN ('CANAL_PRESENCIAL_ORIGEM', '') 
                THEN 'CANAL_PRESENCIAL'
            ELSE 'CANAL_DIGITAL'
        END AS tipo_canal,
        CASE 
            WHEN p.categoria_produto_n2 = 'CATEGORIA_SEGMENTO_A' 
                THEN 'SUBSEGMENTO_A'
            WHEN p.categoria_produto_n2 = 'CATEGORIA_SEGMENTO_B'
                 AND p.categoria_produto_n3 = 'SUBCATEGORIA_SEGMENTO_B'
                THEN 'SUBSEGMENTO_B'
        END AS segmento_estrategico,
        e.valor_venda_bruta_estimado AS valor_venda,
        e.qtd_itens_estimado AS qtd_vendida,
        e.id_transacao_estimado AS id_transacao
    FROM camada_dados.fato_vendas_estimadas e
    JOIN camada_dados.dim_hierarquia_operacional_atual h
        ON e.cod_unidade = CAST(h.cod_unidade AS BIGINT)
    JOIN camada_dados.dim_produto p
        ON e.cod_produto = p.cod_produto 
    WHERE 
        e.data_movimento >= DATE_TRUNC('month', DATE_ADD('month', -1, CURRENT_DATE))
        AND e.data_movimento < DATE_TRUNC('month', DATE_ADD('month', 2, CURRENT_DATE))
),

base_vendas_unificada AS (
    SELECT * FROM vendas_realizadas
    UNION ALL
    SELECT * FROM vendas_estimadas
),

base_com_data_hibrida AS (
    SELECT
        *,
        CASE
            WHEN data_dia >= DATE_TRUNC('month', CURRENT_DATE)
                THEN data_dia
            ELSE CAST(DATE_TRUNC('month', data_dia) AS DATE)
        END AS data_agrupamento
    FROM base_vendas_unificada
)

SELECT 
    cod_unidade,
    data_agrupamento AS data_referencia,
    CASE 
        WHEN GROUPING(tipo_canal) = 1 THEN 'TOTAL_UNIDADE'
        ELSE tipo_canal 
    END AS tipo_canal,
    segmento_estrategico,
    SUM(valor_venda) AS valor_venda,
    SUM(qtd_vendida) AS qtd_itens_vendidos,
    COUNT(DISTINCT id_transacao) AS qtd_transacoes_distintas
FROM base_com_data_hibrida
WHERE segmento_estrategico IS NOT NULL
GROUP BY GROUPING SETS (
    (
        cod_unidade,
        data_agrupamento,
        tipo_canal,
        segmento_estrategico
    ),
    (
        cod_unidade,
        data_agrupamento,
        segmento_estrategico
    )
)
ORDER BY
    data_referencia DESC,
    cod_unidade;



Consulta 19 — Consolidação de Vendas Vinculadas a Programa Subsidiado por Unidade
Objetivo:
Consolidar, por unidade operacional e período de referência, os volumes e valores associados a vendas vinculadas a um programa subsidiado. A consulta considera os últimos seis meses e o mesmo período do ano anterior, mantendo a granularidade diária para o mês atual e mensal para períodos históricos.
WITH base_dados AS (
    SELECT
        data_movimento,
        cod_unidade,
        qtd_itens,
        valor_preco_venda,
        valor_programa_subsidiado,
        qtd_prescrita,
        valor_subsidiado,
        valor_total_repasse
    FROM camada_dados.fato_vendas_programa_subsidiado
    WHERE
        (
            (
                data_movimento >= DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
                AND data_movimento < CURRENT_DATE
            )
            OR
            (
                data_movimento >= DATE_ADD(
                    'year',
                    -1,
                    DATE_ADD('month', -5, DATE_TRUNC('month', CURRENT_DATE))
                )
                AND data_movimento < DATE_ADD('year', -1, CURRENT_DATE)
            )
        )
),

dados_agrupamento AS (
    SELECT
        *,
        CASE
            WHEN data_movimento >= DATE_TRUNC('month', CURRENT_DATE)
                THEN CAST(data_movimento AS DATE)
            ELSE CAST(DATE_TRUNC('month', data_movimento) AS DATE)
        END AS data_referencia
    FROM base_dados
)

SELECT
    data_referencia,
    cod_unidade,
    SUM(qtd_itens) AS qtd_itens_total,
    SUM(valor_preco_venda) AS valor_preco_venda_total,
    SUM(valor_programa_subsidiado) AS valor_programa_subsidiado_total,
    SUM(qtd_prescrita) AS qtd_prescrita_total,
    SUM(valor_subsidiado) AS valor_subsidiado_total,
    SUM(valor_total_repasse) AS valor_repasse_total
FROM dados_agrupamento
GROUP BY
    data_referencia,
    cod_unidade
ORDER BY
    data_referencia DESC,
    cod_unidade;



Consulta 20 — Extração da Base de Metas Diárias por Unidade Operacional
Objetivo:
Extrair a base consolidada de metas diárias utilizada para acompanhamento dos indicadores operacionais e comerciais. Essa base serve como referência para comparar o desempenho realizado com os valores planejados por unidade e período.
SELECT
    *
FROM camada_dados.vw_metas_diarias_operacionais AS meta;

Consulta 21 — Extração da Base de Margem por Categoria de Produto
Objetivo:
Extrair a base consolidada de margem por categoria de produto, utilizada para análise de rentabilidade comercial e acompanhamento do desempenho das categorias no painel analítico.
SELECT
    *
FROM camada_dados.vw_margem_categorias_produto;

Consulta 22 — Classificação Semestral de Clientes por Faixa de Valor e Loja Preferida
Objetivo:
Identificar clientes que atingiram uma faixa mínima de valor acumulado em compras dentro de cada semestre, atribuindo cada cliente à sua unidade operacional preferida com base na maior quantidade de transações realizadas. A consulta expande o período de validade da classificação para os meses correspondentes e consolida a quantidade de clientes classificados por safra mensal e unidade.
WITH parametros AS (
    SELECT
        CAST(1000.00 AS DECIMAL(10,2)) AS valor_minimo_classificacao
),

semestres_recentes AS (
    SELECT DISTINCT 
        CONCAT(
            CAST(ano_referencia AS VARCHAR),
            CASE 
                WHEN mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
        ) AS semestre_referencia
    FROM camada_dados.fato_movimento_vendas_cliente
    WHERE id_cliente IS NOT NULL
      AND id_transacao IS NOT NULL
    ORDER BY 1 DESC
    LIMIT 8
),

venda_mensal_cliente AS (
    SELECT
        m.id_cliente,
        CAST(DATE_TRUNC('month', m.data_movimento) AS DATE) AS data_referencia_mes,
        CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
        ) AS semestre_referencia,
        SUM(m.valor_venda_bruta + m.valor_venda_servico) AS valor_venda_mensal
    FROM camada_dados.fato_movimento_vendas_cliente m
    WHERE m.id_cliente IS NOT NULL
      AND m.id_transacao IS NOT NULL
      AND CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
          ) IN (
            SELECT semestre_referencia 
            FROM semestres_recentes
          )
    GROUP BY
        m.id_cliente,
        CAST(DATE_TRUNC('month', m.data_movimento) AS DATE),
        CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
        )
),

venda_acumulada_cliente AS (
    SELECT
        id_cliente,
        data_referencia_mes,
        semestre_referencia,
        SUM(valor_venda_mensal) OVER (
            PARTITION BY id_cliente, semestre_referencia
            ORDER BY data_referencia_mes
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS valor_venda_acumulada
    FROM venda_mensal_cliente
),

data_classificacao_cliente AS (
    SELECT
        v.semestre_referencia,
        v.id_cliente,
        MIN(v.data_referencia_mes) AS data_classificacao
    FROM venda_acumulada_cliente v
    CROSS JOIN parametros p
    WHERE v.valor_venda_acumulada >= p.valor_minimo_classificacao
    GROUP BY
        v.semestre_referencia,
        v.id_cliente
),

validade_classificacao AS (
    SELECT
        semestre_referencia,
        id_cliente,
        data_classificacao,
        CASE 
            WHEN MONTH(data_classificacao) >= 7
                THEN CAST(
                    CONCAT(
                        CAST(YEAR(data_classificacao) + 1 AS VARCHAR),
                        '-06-30'
                    ) AS DATE
                )
            ELSE CAST(
                    CONCAT(
                        CAST(YEAR(data_classificacao) AS VARCHAR),
                        '-12-31'
                    ) AS DATE
                )
        END AS data_fim_validade
    FROM data_classificacao_cliente
),

unidade_preferida_semestre AS (
    SELECT
        m.id_cliente,
        CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
        ) AS semestre_referencia,
        m.cod_unidade,
        COUNT(DISTINCT m.id_transacao) AS qtd_transacoes,
        ROW_NUMBER() OVER (
            PARTITION BY 
                m.id_cliente,
                CONCAT(
                    CAST(m.ano_referencia AS VARCHAR),
                    CASE 
                        WHEN m.mes_referencia >= 7 THEN '02' 
                        ELSE '01' 
                    END
                )
            ORDER BY 
                COUNT(DISTINCT m.id_transacao) DESC,
                m.cod_unidade ASC
        ) AS ordem_preferencia
    FROM camada_dados.fato_movimento_vendas_cliente m
    WHERE m.id_cliente IS NOT NULL
      AND m.id_transacao IS NOT NULL
      AND CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
          ) IN (
            SELECT semestre_referencia 
            FROM semestres_recentes
          )
    GROUP BY
        m.id_cliente,
        CONCAT(
            CAST(m.ano_referencia AS VARCHAR),
            CASE 
                WHEN m.mes_referencia >= 7 THEN '02' 
                ELSE '01' 
            END
        ),
        m.cod_unidade
),

dim_calendario_mes AS (
    SELECT
        ano,
        mes,
        DATE_PARSE(
            CONCAT(
                CAST(ano AS VARCHAR),
                '-',
                LPAD(CAST(mes AS VARCHAR), 2, '0'),
                '-01'
            ),
            '%Y-%m-%d'
        ) AS data_referencia_mes
    FROM (
        SELECT 
            ano,
            mes
        FROM (
            UNNEST(SEQUENCE(2020, 2030)) t (ano)
            CROSS JOIN UNNEST(SEQUENCE(1, 12)) m (mes)
        )
    ) calendario
),

clientes_classificados_expandido AS (
    SELECT
        c.ano,
        c.mes,
        c.data_referencia_mes,
        v.id_cliente,
        u.cod_unidade
    FROM validade_classificacao v
    JOIN unidade_preferida_semestre u
        ON v.id_cliente = u.id_cliente
        AND v.semestre_referencia = u.semestre_referencia
        AND u.ordem_preferencia = 1
    JOIN dim_calendario_mes c
        ON c.data_referencia_mes >= DATE_TRUNC('month', v.data_classificacao)
        AND c.data_referencia_mes <= v.data_fim_validade
),

clientes_classificados_safra AS (
    SELECT
        id_cliente,
        cod_unidade,
        DATE_FORMAT(
            DATE_TRUNC('month', data_referencia_mes),
            '%Y-%m'
        ) AS safra,
        MIN(data_referencia_mes) AS primeira_data_mes_classificacao
    FROM clientes_classificados_expandido
    GROUP BY
        id_cliente,
        cod_unidade,
        DATE_FORMAT(
            DATE_TRUNC('month', data_referencia_mes),
            '%Y-%m'
        )
)

SELECT
    safra,
    cod_unidade,
    COUNT(DISTINCT id_cliente) AS qtd_clientes_classificados
FROM clientes_classificados_safra
GROUP BY
    safra,
    cod_unidade
ORDER BY
    safra DESC,
    cod_unidade;


Consulta 23 — Consolidação de Clientes Ativos por Unidade Preferida
Objetivo:
Identificar a unidade operacional preferida de cada cliente, considerando a maior quantidade de transações realizadas nos últimos 12 meses fechados. Em seguida, a consulta consolida a quantidade de clientes ativos atribuídos a cada unidade.
WITH cliente_preferido AS (
    SELECT
        m.id_cliente,
        m.cod_unidade,
        COUNT(DISTINCT m.id_transacao) AS qtd_transacoes,
        ROW_NUMBER() OVER (
            PARTITION BY m.id_cliente
            ORDER BY COUNT(DISTINCT m.id_transacao) DESC
        ) AS ordem_preferencia
    FROM camada_dados.fato_movimento_vendas_cliente m
    WHERE
        m.data_movimento >= DATE_ADD('month', -12, DATE_TRUNC('month', CURRENT_DATE))
        AND m.data_movimento < DATE_TRUNC('month', CURRENT_DATE)
        AND m.id_transacao IS NOT NULL
        AND m.id_cliente IS NOT NULL
    GROUP BY
        m.id_cliente,
        m.cod_unidade
)

SELECT
    cod_unidade,
    COUNT(DISTINCT id_cliente) AS qtd_clientes_ativos
FROM cliente_preferido
WHERE ordem_preferencia = 1
GROUP BY
    cod_unidade
ORDER BY
    cod_unidade;


Consulta 24 — Consolidação Mensal de Vendas por Classificação de Cliente
Objetivo:
Consolidar, por mês de referência e classificação de cliente, o valor total de vendas registrado na base analítica. A consulta considera uma janela móvel dos últimos seis meses e o mesmo período do ano anterior, permitindo comparação temporal entre os grupos de clientes.
WITH base_classificacao_cliente AS (
    SELECT
        tipo_registro,
        classificacao_cliente,
        ano_referencia,
        mes_referencia,
        valor_vendas,
        CAST(
            DATE_PARSE(
                CONCAT(
                    CAST(ano_referencia AS VARCHAR),
                    '-',
                    LPAD(CAST(mes_referencia AS VARCHAR), 2, '0'),
                    '-01'
                ),
                '%Y-%m-%d'
            ) AS DATE
        ) AS data_referencia_mes
    FROM camada_dados.fato_vendas_classificacao_cliente
)

SELECT
    DATE_TRUNC('month', data_referencia_mes) AS mes_referencia,
    classificacao_cliente,
    SUM(valor_vendas) AS valor_vendas
FROM base_classificacao_cliente
WHERE
    (
        data_referencia_mes >= DATE_TRUNC('month', DATE_ADD('month', -5, CURRENT_DATE))
        AND data_referencia_mes < DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE))
    )
    OR
    (
        data_referencia_mes >= DATE_TRUNC(
            'month',
            DATE_ADD('year', -1, DATE_ADD('month', -5, CURRENT_DATE))
        )
        AND data_referencia_mes < DATE_ADD(
            'year',
            -1,
            DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE))
        )
    )
GROUP BY
    DATE_TRUNC('month', data_referencia_mes),
    classificacao_cliente
ORDER BY
    mes_referencia DESC,
    classificacao_cliente;

Consulta 25 — Consolidação Mensal de Atendimentos Totais e Identificados
Objetivo:
Consolidar, por mês de referência, a quantidade total de atendimentos e a quantidade de atendimentos com cliente identificado. A consulta considera o período a partir do início do ano de dois anos atrás até o mês atual, excluindo unidades específicas e restringindo a análise a canais comerciais selecionados.
WITH periodo_analise AS (
    SELECT
        DATE_TRUNC('year', DATE_ADD('year', -2, CURRENT_DATE)) AS data_inicio,
        DATE_ADD('month', 1, DATE_TRUNC('month', CURRENT_DATE)) AS data_fim_exclusiva
)

SELECT
    CAST(DATE_TRUNC('month', m.data_movimento) AS DATE) AS mes_referencia,
    COUNT(DISTINCT m.id_transacao) AS qtd_atendimentos_total,
    COUNT(
        DISTINCT CASE 
            WHEN m.id_cliente IS NOT NULL 
            THEN m.id_transacao 
        END
    ) AS qtd_atendimentos_identificados
FROM camada_dados.fato_movimento_vendas_cliente m
CROSS JOIN periodo_analise p
WHERE
    m.id_transacao IS NOT NULL
    AND m.data_movimento >= p.data_inicio
    AND m.data_movimento < p.data_fim_exclusiva
    AND m.cod_unidade NOT IN (9991, 9992, 9993)
    AND m.canal_comercial IN ('CANAL_PRESENCIAL', 'CANAL_ASSISTIDO')
GROUP BY
    CAST(DATE_TRUNC('month', m.data_movimento) AS DATE)
ORDER BY
    mes_referencia DESC;


Consulta 26 — Consolidação Mensal de Avaliações de Satisfação do Cliente por Unidade
Objetivo:
Consolidar, por unidade operacional e mês de referência, a quantidade total de avaliações de satisfação do cliente, classificando as respostas em grupos positivos, neutros e negativos. A consulta considera os últimos 13 meses e serve como base para o cálculo de indicadores de experiência do cliente.
SELECT
    CAST(DATE_TRUNC('month', t.data_resposta) AS DATE) AS mes_referencia,
    t.cod_unidade,
    COUNT(t.nota_satisfacao) AS qtd_avaliacoes,
    SUM(
        CASE 
            WHEN UPPER(t.classificacao_satisfacao) IN ('CLASSIFICACAO_POSITIVA', 'POSITIVE') 
            THEN 1 
            ELSE 0 
        END
    ) AS qtd_avaliacoes_positivas,
    SUM(
        CASE 
            WHEN UPPER(t.classificacao_satisfacao) IN ('CLASSIFICACAO_NEUTRA', 'NEUTRAL') 
            THEN 1 
            ELSE 0 
        END
    ) AS qtd_avaliacoes_neutras,
    SUM(
        CASE 
            WHEN UPPER(t.classificacao_satisfacao) IN ('CLASSIFICACAO_NEGATIVA', 'NEGATIVE') 
            THEN 1 
            ELSE 0 
        END
    ) AS qtd_avaliacoes_negativas
FROM camada_dados.fato_pesquisa_satisfacao_cliente t
WHERE
    t.nota_satisfacao IS NOT NULL
    AND t.classificacao_satisfacao IS NOT NULL
    AND DATE_DIFF(
          'month',
          DATE_TRUNC('month', t.data_resposta),
          DATE_TRUNC('month', CURRENT_DATE)
        ) BETWEEN 0 AND 12
GROUP BY
    CAST(DATE_TRUNC('month', t.data_resposta) AS DATE),
    t.cod_unidade
ORDER BY
    mes_referencia DESC,
    t.cod_unidade;



Consulta 27 — Consolidação Mensal de Avaliação Operacional por Unidade
Objetivo:
Consolidar, por unidade operacional e mês de referência, a pontuação obtida em avaliações operacionais periódicas. A consulta soma a pontuação total registrada e identifica a maior pontuação máxima disponível no período, considerando os últimos seis meses.
SELECT
    CAST(
        DATE_TRUNC(
            'month',
            DATE_PARSE(data_avaliacao, '%d/%m/%Y')
        ) AS DATE
    ) AS mes_referencia,
    cod_unidade,
    SUM(valor_nota_obtida) AS soma_nota_obtida,
    MAX(valor_nota_maxima) AS maior_nota_maxima_registrada
FROM camada_dados.fato_avaliacao_operacional
WHERE
    DATE_PARSE(data_avaliacao, '%d/%m/%Y') >= DATE_TRUNC(
        'month',
        CURRENT_DATE - INTERVAL '5' MONTH
    )
GROUP BY
    1,
    2
ORDER BY
    mes_referencia DESC,
    cod_unidade;


Consulta 28 — Cálculo Mensal de Frequência Média de Compra por Cliente
Objetivo:
Calcular, por mês de referência, a frequência média de compra dos clientes identificados. O indicador é obtido pela razão entre a quantidade de transações distintas e a quantidade de clientes únicos no período. A consulta considera o ano atual e os dois anos anteriores.
SELECT
    CAST(DATE_TRUNC('month', data_movimento) AS DATE) AS mes_referencia,
    NULLIF(
        CAST(COUNT(DISTINCT id_transacao) AS DECIMAL(15, 2)),
        0
    ) / CAST(
        COUNT(DISTINCT id_cliente) AS DECIMAL(15, 2)
    ) AS frequencia_media_compra
FROM camada_dados.fato_movimento_vendas_cliente
WHERE
    data_movimento >= DATE_TRUNC('year', DATE_ADD('year', -2, CURRENT_DATE))
    AND id_cliente IS NOT NULL
    AND id_transacao IS NOT NULL
GROUP BY
    CAST(DATE_TRUNC('month', data_movimento) AS DATE)
ORDER BY
    mes_referencia DESC;


Consulta 29 — Evolução Anual de Clientes Únicos e Crescimento Percentual
Objetivo:
Calcular a quantidade de clientes únicos por ano e comparar a evolução percentual em relação ao ano anterior. Para anos fechados, a consulta considera o total anual de clientes; para o ano corrente, utiliza uma janela móvel dos últimos 12 meses, permitindo acompanhar a evolução recente da base ativa de clientes.
WITH clientes_anuais AS (
    SELECT
        CAST(DATE_TRUNC('year', data_movimento) AS DATE) AS ano_referencia,
        COUNT(DISTINCT id_cliente) AS qtd_clientes_unicos
    FROM camada_dados.fato_movimento_vendas_cliente
    WHERE
        canal_comercial IN ('CANAL_PRESENCIAL', 'CANAL_ASSISTIDO')
    GROUP BY
        CAST(DATE_TRUNC('year', data_movimento) AS DATE)
),

clientes_12m_moveis AS (
    SELECT
        COUNT(DISTINCT id_cliente) AS qtd_clientes_12m_unicos
    FROM camada_dados.fato_movimento_vendas_cliente
    WHERE
        data_movimento >= DATE_ADD('month', -12, CURRENT_DATE)
        AND canal_comercial IN ('CANAL_PRESENCIAL', 'CANAL_ASSISTIDO')
),

base_clientes AS (
    SELECT
        ano_referencia,
        qtd_clientes_unicos
    FROM clientes_anuais
    WHERE
        ano_referencia < CAST(DATE_TRUNC('year', CURRENT_DATE) AS DATE)

    UNION ALL

    SELECT
        CAST(DATE_TRUNC('year', CURRENT_DATE) AS DATE) AS ano_referencia,
        qtd_clientes_12m_unicos AS qtd_clientes_unicos
    FROM clientes_12m_moveis
)

SELECT
    ano_referencia,
    qtd_clientes_unicos,
    LAG(qtd_clientes_unicos) OVER (
        ORDER BY ano_referencia
    ) AS qtd_clientes_ano_anterior,
    ROUND(
        (
            CAST(qtd_clientes_unicos AS DOUBLE)
            - LAG(CAST(qtd_clientes_unicos AS DOUBLE)) OVER (
                ORDER BY ano_referencia
            )
        )
        / NULLIF(
            LAG(CAST(qtd_clientes_unicos AS DOUBLE)) OVER (
                ORDER BY ano_referencia
            ),
            0
        ) * 100,
        2
    ) AS crescimento_anual_pct
FROM base_clientes
ORDER BY
    ano_referencia;



Consulta 30 — Extração Mensal de Indicadores de Rotatividade por Unidade
Objetivo:
Extrair a base mensal de indicadores de rotatividade de pessoas por unidade operacional, considerando os últimos seis meses até a data atual. A consulta permite acompanhar variações na composição do quadro de colaboradores e apoiar análises relacionadas à gestão de pessoas.
SELECT
    *
FROM camada_dados.vw_indicadores_rotatividade_unidade_mensal
WHERE
    data_referencia >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '5' MONTH)
    AND data_referencia < CURRENT_DATE + INTERVAL '1' DAY
ORDER BY
    data_referencia DESC,
    cod_unidade;


Consulta 31 — Extração da Dimensão de Produtos
Objetivo:
Extrair a base dimensional de produtos utilizada para enriquecer as análises comerciais, permitindo relacionar os registros de venda a atributos como código do produto, descrição, categoria, subcategoria, marca e demais classificações analíticas.
SELECT
    *
FROM camada_dados.dim_produtos;



Consulta 32 — Consolidação de Vendas de Clientes Vinculados a Programa Recorrente por Unidade Preferida
Objetivo:
Consolidar, por mês e unidade operacional, a quantidade de clientes vinculados a um programa recorrente e o valor de vendas associado a esses clientes. A consulta combina a base histórica com a base atual de clientes do programa, evita duplicidade de safra e atribui os resultados à unidade preferida do cliente.
WITH ultima_safra_historica AS (
    SELECT 
        MAX(safra_referencia) AS safra_maxima
    FROM camada_dados.fato_clientes_programa_recorrente_historico
),

safra_atual AS (
    SELECT 
        DATE_FORMAT(CURRENT_DATE, '%Y-%m') AS safra_atual
),

base_clientes_programa AS (
    SELECT
        id_cliente_programa,
        safra_referencia
    FROM camada_dados.fato_clientes_programa_recorrente_historico
    WHERE safra_referencia < (
        SELECT safra_atual 
        FROM safra_atual
    )

    UNION ALL

    SELECT
        id_cliente_programa,
        DATE_FORMAT(CURRENT_DATE, '%Y-%m') AS safra_referencia
    FROM camada_dados.fato_clientes_programa_recorrente_atual
    WHERE (
        SELECT safra_atual 
        FROM safra_atual
    ) > (
        SELECT safra_maxima 
        FROM ultima_safra_historica
    )
),

dim_clientes_programa AS (
    SELECT
        id_cliente_programa,
        CAST(
            DATE_PARSE(safra_referencia, '%Y-%m') 
            AS DATE
        ) AS safra_referencia
    FROM base_clientes_programa
)

SELECT
    CAST(
        DATE_TRUNC('month', v.data_movimento) 
        AS DATE
    ) AS periodo_referencia,
    h.cod_unidade,
    COUNT(DISTINCT v.id_cliente) AS qtd_clientes_programa,
    SUM(v.valor_venda_bruta) + SUM(v.valor_venda_servico) AS valor_vendas
FROM camada_dados.fato_movimento_vendas_cliente v
JOIN dim_clientes_programa c
    ON v.id_cliente = c.id_cliente_programa
    AND CAST(
        DATE_TRUNC('month', v.data_movimento) 
        AS DATE
    ) = c.safra_referencia
LEFT JOIN camada_dados.dim_cliente_unidade_preferida p
    ON v.id_cliente = p.id_cliente
JOIN camada_dados.dim_hierarquia_operacional h
    ON p.cod_unidade_preferida_12m = h.cod_unidade
WHERE
    (
        (
            v.data_movimento >= DATE_TRUNC(
                'month', 
                DATE_ADD('month', -5, CURRENT_DATE)
            )
            AND v.data_movimento < DATE_ADD(
                'month', 
                1, 
                DATE_TRUNC('month', CURRENT_DATE)
            )
        )
        OR
        (
            v.data_movimento >= DATE_ADD(
                'year',
                -1,
                DATE_TRUNC(
                    'month', 
                    DATE_ADD('month', -5, CURRENT_DATE)
                )
            )
            AND v.data_movimento < DATE_ADD(
                'year',
                -1,
                DATE_ADD(
                    'month', 
                    1, 
                    DATE_TRUNC('month', CURRENT_DATE)
                )
            )
        )
    )
GROUP BY
    CAST(
        DATE_TRUNC('month', v.data_movimento) 
        AS DATE
    ),
    h.cod_unidade
ORDER BY
    periodo_referencia DESC,
    h.cod_unidade;
