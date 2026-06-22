# Case Declínio de SKU - Grupo Boticário

## Objetivo

Consolidar uma solução técnica estruturada e reproduzível para o **gerenciamento inteligente do ciclo de vida de produtos** do Grupo Boticário.

O projeto abrange três frentes:

### 1. Diagnóstico de Negócio
Análise exploratória (EDA) que identifica padrões históricos — comportamento de vendas, rupturas e expansão de pontos de venda — que antecedem a fase de declínio.

### 2. Modelagem Preditiva (Análise de Sobrevivência)
Modelo **Gradient Boosting Survival** capaz de prever se um SKU entrará em declínio e em qual ciclo isso ocorrerá, dentro de um horizonte de **25 ciclos**.

### 3. Ação Proativa (Proteção de Margem)
Ferramenta para que o time de negócios possa ajustar preços ou interromper a produção antecipadamente, evitando descontos agressivos que prejudicam a margem financeira.

---

## Estrutura do Repositório

```
declinio_sku/
├── data/                        # Dados (adicionar manualmente)
│   └── base-de-dados.csv
├── notebooks/
│   ├── 01_analise_exploratoria.ipynb          # EDA completa
│   ├── 02_modelo_gradient_boosting_survival.ipynb  # GBS - Evento: Declínio Estrutural
│   ├── 03_modelo_gbs_ativo_inativo.ipynb      # GBS - Evento: Ativo → Inativo
│   └── 04_modelo_gbs_evento_picos.ipynb       # GBS - Evento: Distância do Pico (v3)
├── src/
│   └── modelagem_preditiva.py   # Pipeline auxiliar
├── models/                      # Modelos serializados
├── reports/
│   └── figures/                 # Gráficos gerados
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Notebooks

### 01 - Análise Exploratória
- Visão geral, estatísticas descritivas, valores ausentes
- Distribuições, correlações, análise temporal
- Comparação Ativos vs Inativos (vendas, ruptura, PDVs)
- Análise de distância do pico e tempo desde o pico
- Análise de evento (repacking)

### 02 - GBS: Declínio Estrutural
- **Evento:** Expansão fraca de PDVs + retração da base por 5 ciclos
- Modelo: Gradient Boosting Survival Analysis
- Previsão condicional: P(T < t+25 | T > t)

### 03 - GBS: Ativo → Inativo
- **Evento:** `des_status_atual_agrup == "Inativo"`
- Mesmo pipeline do notebook 02

### 04 - GBS: Distância do Pico (Versão Final)
- **Evento:** Receita < 20% do pico + >=15 ciclos sem renovar pico + PDVs caindo
- **Confirmação:** 4 ciclos consecutivos
- **Features (~30):** Slopes, médias, pico, aceleração, médias móveis, lags, ruptura, idade
- **C-Index:** 0.75 treino / 0.75 teste (zero overfitting)
- **Previsão:** P(declinar nos próximos 25 ciclos | vivo hoje) = 1 - S(t+25)/S(t)
- **Proteção de Margem:** Política de preços baseada no output do modelo
- **Simulação de ROI:** Cenários conservador, realista e otimista

---

## Metodologia

### Definição do Evento de Declínio 
```
sinal_declinio = (
    receita < 20% do pico histórico     AND
    >= 15 ciclos sem renovar o pico      AND
    base de PDVs caindo (diff < 0)
)
evento = sinal confirmado por 4 ciclos consecutivos
```

### Features do Modelo
- **Capilaridade:** `ind_cpfs_novos`, `ind_cpfs_total` (médias + slopes)
- **Vendas:** `ind_vlr_receita_real_dia_corrigido`, `ind_vlr_receita_real_corrigido`
- **Velocidade:** Slopes últimos 5 e 10 ciclos
- **Aceleração:** slope_ult5 - slope_ult10
- **Pico:** `pct_pico_atual`, `ciclos_desde_pico_atual`
- **Ticket Médio:** Receita/PDV (média + slope)
- **Ruptura:** Proporção global e últimos 10 ciclos
- **Médias Móveis:** MM3 e MM5 das variáveis-chave
- **Lags:** Último valor observado
- **Idade:** Ciclos desde o lançamento (phase_in)

### Previsão Condicional
```
P(declinar nos próximos 25 ciclos | vivo hoje) = 1 - S(t+25) / S(t)
```
Onde S(t) é a função de sobrevivência estimada pelo Gradient Boosting Survival.

### Proteção de Margem
| Probabilidade (modelo) | Ação recomendada |
|------------------------|------------------|
| > 50% + ciclo < 5 | URGENTE: Phase-out imediato |
| > 50% | ALTO: Redução 10-15% + suspender produção |
| 30-50% | MÉDIO: Sem novos lotes, sem repacking |
| < 30% | BAIXO: Manter preço e investimento |

---

## Resultados

| Métrica | Valor |
|---------|-------|
| C-Index Treino | 0.7512 |
| C-Index Teste | 0.7512 |
| Gap (overfitting) | 0.0000 |
| Horizonte | 25 ciclos |

---

## Instalação e Uso

### Pré-requisitos
- Python 3.9+
- Google Colab (recomendado)

### Setup Local

```bash
git clone https://github.com/gustavojotar/declinio_sku.git
cd declinio_sku
pip install -r requirements.txt
```

### Google Colab
1. Abrir o notebook desejado no Colab
2. Montar o Google Drive
3. Ajustar o caminho do arquivo CSV em: `/content/drive/MyDrive/Case SKU/data/base-de-dados.csv`

---

## Stack Tecnológica

- **Linguagem:** Python 3.9+
- **Manipulação:** pandas, numpy, scipy
- **Visualização:** matplotlib, seaborn
- **Modelagem:** scikit-survival (Gradient Boosting Survival)
- **Pré-processamento:** scikit-learn (StandardScaler, train_test_split)
- **Ambiente:** Google Colab

---

## Autor

**Gustavo** — [@gustavojotar](https://github.com/gustavojotar)

---

## Licença

Este projeto é de uso público para fins educacionais e profissionais.
