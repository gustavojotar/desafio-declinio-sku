
```markdown
# Desafio Declínio de SKU 

## Visão Geral

Solução preditiva para o **gerenciamento inteligente do ciclo de vida de produtos** do Grupo Boticário, capaz de identificar SKUs em risco de declínio e recomendar ações proativas de proteção de margem.

O projeto responde a duas perguntas de negócio:
1. **Este SKU vai entrar em declínio nos próximos 25 ciclos?** (probabilidade)
2. **Em qual ciclo isso vai acontecer?** (timing para ação)

---

## Resultados Principais

| Métrica | Valor |
|---------|-------|
| Modelo | Gradient Boosting Survival |
| C-Index Treino | 0.7512 |
| C-Index Teste | 0.7512 |
| Overfitting | Zero (gap = 0) |
| Horizonte | 25 ciclos (~16 meses) |

---

## Estrutura do Repositório

```
desafio-declinio-sku/
├── data/                        # Dados (não versionados)
│   └── base-de-dados.csv
├── notebooks/
│   ├── 01_analise_exploratoria.ipynb          # Diagnóstico de negócio
│   ├── 02_modelo_gradient_boosting_survival.ipynb  # Modelo: evento declínio estrutural
│   ├── 03_modelo_gbs_ativo_inativo.ipynb      # Modelo: evento ativo → inativo
│   ├── 04_modelo_gbs_evento_picos.ipynb       # Modelo final + proteção de margem + ROI
│   └── 05_comparacao_modelos.ipynb            # Comparação: Cox PH vs RSF vs GBS
├── src/
│   └── modelagem_preditiva.py
├── models/
├── reports/figures/
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Abordagem

### 1. Diagnóstico de Negócio (Notebook 01)

Análise exploratória que identificou os padrões que antecedem o declínio:

- **Capilaridade:** Perda de PDVs novos é o sinal mais precoce (até 80 ciclos antes)
- **Vendas:** Receita cruza o baseline ~10 ciclos antes do fim (ponto sem retorno)
- **Ruptura:** Colapso logístico nos últimos 30 ciclos (produção descontinuada antes da desativação oficial)
- **Ticket Médio:** Sobe no final — indica abandono de lojas, não de público fiel
- **Marketing:** Estatisticamente irrelevante para diferenciar Ativos de Inativos (Kruskal-Wallis p = 0.45)

### 2. Modelagem Preditiva (Notebooks 02-05)

**Modelo escolhido:** Gradient Boosting Survival Analysis (scikit-survival)

**Por que Análise de Sobrevivência?**
- Responde **SE** e **QUANDO** (não apenas sim/não)
- Lida com **censura** (produtos ativos que ainda não declinaram)
- Gera uma **curva de probabilidade** ao longo do tempo
- Previsão **condicional**: dado que o produto está vivo hoje, qual a chance de declinar nos próximos 25 ciclos?

**Definição do Evento de Declínio:**
```
Um SKU entra em declínio quando simultaneamente:
  1. Receita cai abaixo de 20% do pico histórico
  2. Não renova o pico há 15+ ciclos
  3. Base de PDVs está encolhendo (diff < 0)
  4. Esses sinais persistem por pelo menos 4 ciclos consecutivos
```

**Previsão Condicional:**
```
P(declinar nos próximos 25 ciclos | vivo hoje) = 1 - S(t+25) / S(t)
```

**Features selecionadas (~30):**

| Pilar | Features | Justificativa |
|-------|----------|---------------|
| Capilaridade | PDVs novos, PDVs total (média + slope) | Sinal mais precoce (H = 13.470 e 16.547) |
| Vendas | Receita diária corrigida (média + slope) | Maior poder discriminativo (H = 21.354) |
| Velocidade | Slopes últimos 5 e 10 ciclos | Velocidade da mudança > valor absoluto |
| Aceleração | slope_ult5 - slope_ult10 | Detecta se a queda está se intensificando |
| Distância do pico | % do pico atual, ciclos desde o pico | Critério direto de obsolescência |
| Ticket Médio | Receita/PDV (média + slope) | Sobe no final = declínio escondido |
| Ruptura | Proporção global e últimos 10 ciclos | Colapso logístico |
| Médias Móveis | MM3 e MM5 | Suaviza ruído sazonal |
| Lags | Último valor observado | Padrão recente |
| Idade | Ciclos desde o lançamento | Maturidade como fator de risco |

**Comparação de Modelos (Notebook 05):**

| Modelo | Tipo | Uso |
|--------|------|-----|
| Cox PH | Linear | Baseline interpretável |
| Random Survival Forest | Bagging | Robustez sem tuning |
| Gradient Boosting Survival | Boosting | Melhor performance |

### 3. Ação Proativa (Notebook 04, Seções 14-16)

**Proteção de Margem — Política de preços baseada no modelo:**

| Probabilidade | Ação |
|---------------|------|
| > 50% + ciclo < 5 | URGENTE: Phase-out imediato, vender estoque a preço cheio |
| > 50% | ALTO: Redução gradual 10-15%, suspender produção |
| 30-50% | MÉDIO: Sem novos lotes, sem repacking, reavaliar em 5 ciclos |
| < 30% | BAIXO: Manter preço e investimento |

**Simulação de ROI — Proteção de Margem:**

| Cenário | Desconto SEM modelo | Desconto COM modelo | Margem salva |
|---------|:-------------------:|:-------------------:|:------------:|
| Conservador | 30% | 10% | +20 p.p. |
| Realista | 45% | 10% | +35 p.p. |
| Otimista | 55% | 5% | +50 p.p. |

**ROI de Realocação de Marketing:**

O modelo identifica SKUs condenados que ainda recebem investimento em marketing (comprovadamente ineficaz para produtos em declínio). A verba é redirecionada para categorias com maior sobrevivência.

| Cenário | % Realocada | ROI no destino |
|---------|:-----------:|:--------------:|
| Conservador | 50% | 1.5x |
| Realista | 70% | 2.5x |
| Otimista | 90% | 4.0x |

---

## Como Executar

### Pré-requisitos
- Python 3.9+
- Google Colab (recomendado)

### Google Colab (recomendado)
1. Abrir o notebook desejado no Colab
2. Na primeira célula, o Google Drive será montado automaticamente
3. Ajustar o caminho do CSV: `/content/drive/MyDrive/Case SKU/data/base-de-dados.csv`
4. Executar todas as células em sequência

### Local
```bash
git clone https://github.com/gustavojotar/desafio-declinio-sku.git
cd desafio-declinio-sku
pip install -r requirements.txt
```

---

## Stack Tecnológica

| Componente | Ferramenta |
|------------|-----------|
| Linguagem | Python 3.9+ |
| Manipulação | pandas, numpy, scipy |
| Visualização | matplotlib, seaborn |
| Modelagem | scikit-survival (Gradient Boosting Survival) |
| Pré-processamento | scikit-learn |
| Ambiente | Google Colab |

---

## Resumo Executivo

O declínio de um SKU **não é um evento súbito** — ele segue um padrão previsível:

| Fase | Ciclos antes do fim | Sinal |
|------|:-------------------:|-------|
| Alerta precoce | -80 a -60 | PDVs novos desaparecem |
| Estagnação | -60 a -15 | Receita para de crescer, base PDVs encolhe |
| Ponto sem retorno | -15 a -10 | Receita cruza 20% do pico |
| Queda acelerada | -10 a 0 | Todos indicadores despencam |

**Janela de ação proativa:** entre ciclo -15 e -10 (antes do ponto sem retorno, quando ainda é possível descontinuar sem descontos agressivos).

---

## Autor

**Gustavo** — [@gustavojotar](https://github.com/gustavojotar)


```

