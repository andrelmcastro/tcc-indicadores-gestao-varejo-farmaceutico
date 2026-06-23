APÊNDICE C — Script Python para extração e tratamento dos dados de EBITDA

# BLOCO 1 — IMPORTAÇÃO DAS BIBLIOTECAS
# Objetivo: carregar as bibliotecas utilizadas para conexão, tratamento dos dados, cálculo dos
# indicadores, exportação dos arquivos e geração dos gráficos.
# ==================================================================================================

import warnings
import re
import unicodedata
from pathlib import Path
from textwrap import dedent

import numpy as np
import pandas as pd
import pyodbc
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter


# ==================================================================================================
# BLOCO 2 — CONFIGURAÇÕES GERAIS DA ANÁLISE
# Objetivo: definir parâmetros de conexão, período de análise, diretórios de saída e regras usadas
# no cálculo dos indicadores.
# ==================================================================================================

BASE_DIR = Path.cwd()

DSN_NAME = "<CONEXAO_ODBC>"
CONN_STR = f"DSN={DSN_NAME}"

PASTA_SAIDA = BASE_DIR / "saidas_analise_ebitda_tcc"
PASTA_SAIDA.mkdir(parents=True, exist_ok=True)

TIMESTAMP_EXECUCAO = pd.Timestamp.now().strftime("%Y%m%d_%H%M%S")

PASTA_GRAFICOS = PASTA_SAIDA / f"graficos_tcc_{TIMESTAMP_EXECUCAO}"
PASTA_RESUMOS = PASTA_SAIDA / f"resumos_tcc_{TIMESTAMP_EXECUCAO}"

PERIODO_INICIO = "2025-01-01"
PERIODO_FIM = "2026-04-30"

MESES_BASE_BIMESTRE = [1, 2]
ANO_BASE = 2025
ANO_COMPARACAO = 2026

LIMIAR_TENDENCIA_REL_MENSAL = 1.0
QTD_EXTREMOS = 10
ANONIMIZAR_UNIDADES = True


# ==================================================================================================
# BLOCO 3 — CONSULTA SQL PARA EXTRAÇÃO DOS DADOS
# Objetivo: integrar a base de unidades monitoradas com a base financeira utilizada no cálculo
# do EBITDA.
# ==================================================================================================

SQL_QUERY = dedent("""
WITH base_unidades AS (
    SELECT
        cod_unidade,
        situacao_unidade,
        maturacao_unidade,
        municipio,
        uf,
        regional,
        grupo_monitoramento,
        data_inicio_monitoramento,
        status_monitoramento
    FROM (
        SELECT
            CAST(cod_unidade AS VARCHAR) AS cod_unidade,
            situacao_unidade,
            maturacao_unidade,
            municipio,
            uf,
            regional,
            grupo_monitoramento,
            data_inicio_monitoramento,
            status_monitoramento,
            ROW_NUMBER() OVER (
                PARTITION BY CAST(cod_unidade AS VARCHAR)
                ORDER BY
                    CASE WHEN data_inicio_monitoramento IS NULL THEN 1 ELSE 0 END,
                    data_inicio_monitoramento DESC
            ) AS rn
        FROM <schema_operacional>.<base_unidades_monitoradas>
    ) x
    WHERE rn = 1
),

base_financeira AS (
    SELECT
        f.*,
        COALESCE(
            NULLIF(TRIM(CAST(f.codigo_unidade_tratado AS VARCHAR)), ''),
            NULLIF(REGEXP_REPLACE(TRIM(CAST(f.descricao_unidade AS VARCHAR)), '[^0-9]', ''), '')
        ) AS codigo_unidade_join
    FROM <schema_financeiro>.<base_resultado_financeiro> f
)

SELECT
    f.codigo_unidade_join,
    f.descricao_unidade,
    f.codigo_unidade_tratado,
    f.grupo_resultado,
    f.data_competencia,
    f.valor_resultado,
    u.situacao_unidade,
    u.maturacao_unidade,
    u.municipio,
    u.uf,
    u.regional,
    u.grupo_monitoramento,
    u.data_inicio_monitoramento,
    u.status_monitoramento
FROM base_financeira f
LEFT JOIN base_unidades u
    ON f.codigo_unidade_join = u.cod_unidade
""")


# ==================================================================================================
# BLOCO 4 — FUNÇÕES DE CARREGAMENTO E PADRONIZAÇÃO
# Objetivo: executar a consulta SQL, carregar os dados em um DataFrame e padronizar os nomes das
# colunas para facilitar o tratamento no Python.
# ==================================================================================================

def carregar_dados(sql_query: str) -> pd.DataFrame:
    if "CONN_STR" not in globals():
        raise RuntimeError(
            "A variável CONN_STR não foi definida. Execute o bloco de configuração antes de carregar os dados."
        )

    conn = None

    try:
        with warnings.catch_warnings():
            warnings.simplefilter("ignore")
            conn = pyodbc.connect(CONN_STR)
            df = pd.read_sql_query(sql_query, conn)

    except Exception as e:
        raise RuntimeError(f"Erro ao carregar dados: {e}") from e

    finally:
        if conn is not None:
            conn.close()

    if df.empty:
        raise ValueError("A consulta foi executada, mas não retornou dados.")

    return df


def normalizar_nome_coluna(nome: str) -> str:
    texto = unicodedata.normalize("NFKD", str(nome)).encode("ascii", "ignore").decode("ascii")
    texto = texto.strip().lower()
    texto = re.sub(r"[^a-z0-9]+", "_", texto)
    texto = re.sub(r"_+", "_", texto).strip("_")
    return texto


def padronizar_colunas(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()
    df.columns = [normalizar_nome_coluna(col) for col in df.columns]
    return df


def inspecionar_dataframe(df: pd.DataFrame, nome: str = "DataFrame") -> None:
    print("=" * 100)
    print(f"INSPEÇÃO — {nome}")
    print("=" * 100)
    print(f"Linhas: {df.shape[0]}")
    print(f"Colunas: {df.shape[1]}")
    print("\nColunas:")
    print(df.columns.tolist())
    print("\nPrimeiras 5 linhas:")
    print(df.head())


# ==================================================================================================
# BLOCO 5 — PREPARAÇÃO DA BASE ANALÍTICA
# Objetivo: filtrar as unidades monitoradas, selecionar o período de análise e manter apenas os
# grupos financeiros utilizados no cálculo do EBITDA.
# ==================================================================================================

def preparar_base_monitorada(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()

    colunas_obrigatorias = [
        "descricao_unidade",
        "codigo_unidade_tratado",
        "grupo_resultado",
        "data_competencia",
        "valor_resultado",
        "status_monitoramento",
        "municipio",
        "uf",
        "regional",
        "grupo_monitoramento",
        "data_inicio_monitoramento",
        "maturacao_unidade",
        "situacao_unidade",
    ]

    faltantes = [col for col in colunas_obrigatorias if col not in df.columns]
    if faltantes:
        raise ValueError(f"Colunas obrigatórias ausentes na base: {faltantes}")

    df["data_competencia"] = pd.to_datetime(df["data_competencia"], errors="coerce")
    df["data_inicio_monitoramento"] = pd.to_datetime(df["data_inicio_monitoramento"], errors="coerce")
    df["valor_resultado"] = pd.to_numeric(df["valor_resultado"], errors="coerce")
    df["status_monitoramento"] = df["status_monitoramento"].astype(str).str.strip().str.upper()

    if "codigo_unidade_join" in df.columns:
        df["codigo_unidade"] = df["codigo_unidade_join"].astype(str).str.strip()
    else:
        df["codigo_unidade"] = (
            df["codigo_unidade_tratado"]
            .fillna(df["descricao_unidade"])
            .astype(str)
            .str.replace(r"[^0-9]", "", regex=True)
            .str.strip()
        )

    df = df[
        (df["status_monitoramento"] == "MONITORADO")
        & (df["codigo_unidade"].notna())
        & (df["codigo_unidade"] != "")
        & (df["data_competencia"].notna())
        & (df["valor_resultado"].notna())
    ].copy()

    df = df[
        (df["data_competencia"] >= pd.to_datetime(PERIODO_INICIO))
        & (df["data_competencia"] <= pd.to_datetime(PERIODO_FIM))
    ].copy()

    grupos_ebitda = [
        "Receita bruta",
        "Deduções da receita bruta",
        "Custos das mercadorias vendidas",
        "Despesas com vendas e administrativas",
    ]

    df["grupo_resultado"] = df["grupo_resultado"].astype(str).str.strip()
    df = df[df["grupo_resultado"].isin(grupos_ebitda)].copy()

    df["ano_mes"] = df["data_competencia"].dt.to_period("M").dt.to_timestamp()
    df["ano"] = df["ano_mes"].dt.year
    df["mes"] = df["ano_mes"].dt.month
    df["ano_mes_texto"] = df["ano_mes"].dt.strftime("%m/%Y")

    return df.reset_index(drop=True)


# Observação: o restante do script foi mantido conforme o texto enviado originalmente no apêndice.
# Caso o objetivo seja executar o código no Python, recomenda-se uma revisão posterior de sintaxe,
# pois há trechos no texto original com nomes de variáveis contendo espaços e acentos.
