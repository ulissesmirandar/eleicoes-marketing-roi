# 🗳️ ROI e Eficiência do Marketing Político Digital no Brasil (2012–2026)

Este repositório documenta um projeto autoral de **Análise de Dados e Business Intelligence** focado na evolução dos gastos eleitorais em plataformas digitais e na relação entre investimento financeiro (Meta, Google, Kwai) e conversão de votos no Brasil.

O estudo utiliza dados públicos do **Tribunal Superior Eleitoral (TSE)** tratados pela iniciativa **Base dos Dados**, processados via **Google BigQuery (SQL)** e visualizados de forma interativa no **Looker Studio**.

---

## 📌 Contexto & Problema de Negócio

Com a minirreforma eleitoral de 2017 (Lei nº 13.488/2017), o impulsionamento pago em redes sociais passou a ser expressamente permitido a partir das Eleições Gerais de 2018. Desde então, centenas de milhões de reais migraram do modelo tradicional de campanha para as *Big Techs*.

O objetivo deste projeto é responder a duas questões centrais:
1. **Visão Macro (Linha do Tempo):** Qual foi o salto financeiro real das plataformas digitais de 2012 a 2026, separando Eleições Gerais e Municipais?
2. **Visão Micro (Funil de Eficiência):** O investimento em anúncios pagos compra votos? Qual o perfil de retorno das campanhas eleitas frente aos candidatos que não converteram verba em cadeiras?

---

## 🛠️ Stack Tecnológica

* **Banco de Dados:** Google BigQuery (SQL Standard)
* **Fonte de Dados:** `basedosdados.br_tse_eleicoes` (TSE / Base dos Dados)
* **Engenharia/Camada Intermediária:** Google Sheets (camada de cache leve para o BI)
* **Visualização:** Google Looker Studio
* **Metodologia:** *Build in Public* com uso assistido de LLMs (Gemini/ChatGPT) para prototipagem ágil e depuração analítica.

---

## 🔍 Principais Descobertas Técnicas & Metodológicas

### 1. Auditoria e os "R$ 100 Milhões Invisíveis"
Uma consulta direta pelos CNPJs tradicionais da Meta (*13.347.016/0001-17*) ignora uma fatia expressiva dos gastos reais. A auditoria das declarações revelou que milhares de campanhas prestam contas utilizando o CNPJ das instituições de pagamento contratadas pela plataforma para processar boletos e PIX (*dLocal, Adyen, EBANX, PayU*). 
* A inclusão das intermediadoras elevou a apuração real da Meta de valores parciais de ~R$ 290 milhões para **R$ 402,1 milhões** em 2026.
* O recuo do Google após as eleições de 2022 evidenciou o efeito da política interna de descontinuação de anúncios políticos no país em 2024.

### 2. Tratamento de Outliers e Escala Logarítmica
Na matriz de candidatos (15.054 registros de 2018 e 2022), a discrepância de ordens de magnitude em votos (de dezenas a centenas de milhares) e gastos exigiu a implementação de **Escala Logarítmica (Log-Log)** nos eixos X e Y para eliminar o *overplotting* e permitir a visualização de quadrantes com base nas médias.

### 3. Independência de Eixos e Eliminação de Correlação Espúria
A modelagem inicial utilizava `custo_por_voto` ($Gasto / Votos$) no Eixo Y, o que acoplava matematicamente os dois eixos e forçava uma distribuição em rampa descendente. A versão definitiva adotou `gasto_impulsionamento` (Eixo Y) versus `votos` (Eixo X), deslocando o `custo_por_voto` para métricas de *hover* (`Tooltip/Dimensão Composta`) e rankings detalhados de tabela.

---

## 💻 Códigos SQL (Google BigQuery)

<details>
<summary><b>Query 1: Visão Macro Histórica (2012–2026)</b></summary>

WITH Gastos_Impulsionamento AS (
  SELECT 
    ano,
    sequencial_candidato,
    SUM(valor_despesa) AS total_gasto
  FROM `basedosdados.br_tse_eleicoes.despesas_candidato`
  WHERE ano IN (2018, 2022)
    AND cpf_cnpj_fornecedor IN (
      '13347016000117', '13347016000389', '25021356000132', 
      '14796606000190', '24080493000185', '08965639000113',
      '06990590000123', '40225615000130'
    )
  GROUP BY ano, sequencial_candidato
),
Total_Votos AS (
  SELECT
    ano,
    sequencial_candidato,
    SUM(CASE WHEN turno = 1 THEN votos ELSE 0 END) AS total_votos,
    MAX(resultado) AS status_eleicao
  FROM `basedosdados.br_tse_eleicoes.resultados_candidato`
  WHERE ano IN (2018, 2022)
  GROUP BY ano, sequencial_candidato
)
SELECT 
  c.ano,
  c.sigla_uf AS estado,
  c.cargo,
  c.nome_urna AS candidato,
  c.sigla_partido AS partido,
  ROUND(g.total_gasto, 2) AS gasto_impulsionamento,
  v.total_votos AS votos,
  ROUND(SAFE_DIVIDE(g.total_gasto, v.total_votos), 2) AS custo_por_voto,
  CASE 
    WHEN v.status_eleicao IN ('eleito', 'eleito por qp', 'eleito por media') THEN 'Eleito'
    WHEN v.status_eleicao IN ('suplente', 'nao eleito') THEN 'Não Eleito'
    ELSE 'Outros (Indeferidos/Renúncia)'
  END AS status_final
FROM `basedosdados.br_tse_eleicoes.candidatos` AS c
INNER JOIN Gastos_Impulsionamento AS g 
  ON c.sequencial = g.sequencial_candidato AND c.ano = g.ano
INNER JOIN Total_Votos AS v 
  ON c.sequencial = v.sequencial_candidato AND c.ano = v.ano
WHERE c.ano IN (2018, 2022)
  AND c.sigla_uf != 'BR'
  AND v.total_votos > 0
ORDER BY custo_por_voto DESC;

📊 Dashboard Interativo
Acesse o painel completo no Looker Studio com filtros dinâmicos por Ano, Estado, Cargo e Partido:

[🔗 Visualizar Painel Interativo](https://datastudio.google.com/s/im7-DIrd0Kc)

✍️ Autor
Ulisses Miranda — [LinkedIn](www.linkedin.com/in/ulisses-miranda/) | [Medium](https://medium.com/@ulissesmiranda)
