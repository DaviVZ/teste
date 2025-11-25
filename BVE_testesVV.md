# BVE — Testes de Verificação e Validação (Lab 10)

## 1. Identificação do Grupo

-   **Grupo:**  106
-   **Integrantes:** Beatriz Molinari Satil de Souza, Davi Vieira Zandomeneghi, Samuel Roizenblatt Davidovici
-   **Data:**  27/10/2025
-   **Versão do modelo (commit ou data):** V6


## 2. Plano de Verificação

O objetivo geral da verificação foi garantir que o código dos notebooks e a integração com os arquivos de parâmetros e de saída estivessem corretos, produzindo resultados consistentes com os cenários definidos de teste. Para isso, foi elaborado um conjunto de casos de teste, com foco em aspectos de determinismo, variação de parâmetros e comportamento extremo da frota.

## 2.1 Casos de Teste

| ID | Descrição do teste                                                                 | Objetivo principal                                                       | Entradas principais                                                                                                           | Saídas / indicadores observados principais                                                          | Status (OK / Falha) |
|----|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|---------------------|
| T1 | Determinístico simples — demanda fixa e frota 3 barcaças, 10 dias                  | Validar funcionamento básico do modelo sem aleatoriedade (sanidade)      | Frota fixa de 3 barcaças, demanda de 500 t/dia para 3 navios, 10 dias de simulação                                          | Número de serviços realizados, volume total atendido, backlog mínimo esperado (zero ou próximo de zero)| OK                  |
| T2 | Extremos de frota — 1 barcaça vs 10 barcaças                                       | Verificar impacto de frota mínima e máxima na capacidade de atendimento  | Dois cenários: (a) frota de 1 barcaça, (b) frota de 10 barcaças, mesma demanda e horizonte de tempo                         | Diferença significativa de backlog entre os dois cenários; volume atendido maior com frota ampliada   | OK                  |
| T3 | Aumento da demanda — cenário de stress de demanda                                   | Avaliar comportamento do sistema sob demanda elevada                     | Demanda maior para os mesmos navios (por exemplo, 3 navios com mais de 500 t/dia ou número maior de navios)                 | Backlog crescente, limitação de recurso barcaça clara nos indicadores                               | OK                  |
| T4 | Teste de consistência dos logs                                                      | Conferir se os arquivos de log são gerados de forma padronizada          | Execução dos notebooks de simulação com parâmetros padrão                              | Arquivos CSV contendo timestamps de início e fim, volumes atendidos, identificação de viagens        | OK                  |
| T5 | Consistência estatística (replicações)                                              | Verificar estabilidade estatística com replicações                        | Múltiplas rodadas para um cenário aleatório (se aplicável)                            | Convergência de médias de indicadores, pouca variação relativa entre replicações                     | Pendente            |

## 2.2 Principais Resultados e Evidências

As evidências de verificação foram coletadas a partir dos arquivos de log gerados pelas simulações (`abastecimento_log.csv`) e, quando disponível, de outros arquivos auxiliares (por exemplo, arquivos com parâmetros de frota ou de demanda). A seguir, são apresentados os scripts utilizados para sumarizar os resultados dos testes, bem como os indicadores calculados e gráficos produzidos.

```python
import pandas as pd
import glob
import os
import matplotlib.pyplot as plt

# Identifica automaticamente todas as subpastas de teste (nome iniciando por "teste_") no diretório atual.
base_dir = os.getcwd()
test_dirs = [d for d in os.listdir(base_dir) if d.startswith('teste_') and os.path.isdir(os.path.join(base_dir,d))]

results = []
duration_data = {}
volume_data = {}

for test_name in sorted(test_dirs):
    full_path = os.path.join(base_dir, test_name)
    att_logs = glob.glob(os.path.join(full_path, 'abastecimento_log_*.csv'))

    if not att_logs:
        continue

    duration_list = []
    volume_list = []

    for att_file in sorted(att_logs):
        run_id = att_file.split('_')[-1].split('.')[0]
        df_att = pd.read_csv(att_file)
        if df_att.empty:
            continue

        num_services = len(df_att)
        total_volume = df_att['volume_atendido'].sum()
        mean_duration = (df_att['fim_abastecimento'] - df_att['ini_abastecimento']).mean()
        total_demanda = df_att['demanda'].sum()
        backlog = total_demanda - total_volume

        scenario = run_id
        if test_name.startswith('teste_T2'):
            scenario = '1 barcaça' if run_id == '000' else '10 barcaças'

        label = f"{test_name} ({scenario})"

        results.append({
            'Teste': test_name,
            'Cenário': scenario,
            'Label': label,
            'Serviços realizados': num_services,
            'Volume atendido (t)': total_volume,
            'Duração média (min)': mean_duration,
            'Demanda total (t)': total_demanda,
            'Backlog (t)': backlog
        })

        duration_list.extend((df_att['fim_abastecimento'] - df_att['ini_abastecimento']).tolist())
        volume_list.extend(df_att['volume_atendido'].tolist())

    duration_data[test_name] = duration_list
    volume_data[test_name] = volume_list

summary_df = pd.DataFrame(results)

# Monta um DataFrame-resumo com os principais indicadores calculados para cada cenário de teste.
summary_df['Teste_Cenario'] = summary_df['Label']

# Calcula métricas derivadas (taxa de atendimento e volume médio por serviço) a partir dos indicadores básicos.
summary_df['Taxa de atendimento (%)'] = (summary_df['Volume atendido (t)'] / summary_df['Demanda total (t)']) * 100
summary_df['Volume médio por serviço (t/serviço)'] = summary_df['Volume atendido (t)'] / summary_df['Serviços realizados']

# Registra, em formato de dicionário, a descrição formal de cada caso de teste para compor o plano de verificação.
cases_data = [
    {
        'ID': 'T1',
        'Descrição': 'Determinístico simples',
        'Objetivo': 'Sanidade do modelo',
        'Resultado esperado': 'Backlog próximo de zero',
        'Resultado obtido': 'Backlog próximo de zero, volume atendido compatível com demanda',
        'Status': 'OK'
    },
    {
        'ID': 'T2',
        'Descrição': 'Extremos de Frota',
        'Objetivo': 'Impacto de 1 vs 10 barcaças',
        'Resultado esperado': 'Maior backlog com 1 barcaça; atendimento quase total com 10 barcaças',
        'Resultado obtido': 'Maior backlog com 1 barcaça; redução clara de backlog com 10 barcaças',
        'Status': 'OK'
    },
    {
        'ID': 'T3',
        'Descrição': 'Aumento da Demanda',
        'Objetivo': 'Stress de demanda',
        'Resultado esperado': 'Backlog crescente',
        'Resultado obtido': 'Backlog de centenas de toneladas, indicando saturação da frota',
        'Status': 'OK'
    },
    {
        'ID': 'T4',
        'Descrição': 'Consistência dos Logs',
        'Objetivo': 'Padrão de arquivos de saída',
        'Resultado esperado': 'Arquivos gerados com colunas corretas e sem linhas vazias',
        'Resultado obtido': 'Logs gerados com colunas esperadas e dados consistentes',
        'Status': 'OK'
    },
    {
        'ID': 'T5',
        'Descrição': 'Consistência estatística',
        'Objetivo': 'Verificar estabilidade com replicações',
        'Resultado esperado': 'Indicadores estáveis',
        'Resultado obtido': '32 serviços, volume 32 900 t; replicação única',
        'Status': 'Pendente'
    }
]

cases_df = pd.DataFrame(cases_data)

# Exibe, ao final, a tabela de indicadores e o quadro-resumo dos casos de teste para conferência dos resultados.
summary_df, cases_df
```

### 2.2.1. Teste 1: Determinístico Controlado (Teste de Sanidade)

O primeiro teste (T1) consistiu em um cenário determinístico de sanidade, com uma frota de 3 barcaças e três navios (`NAV001`, `NAV002`, `NAV003`), com demandas de 500 t cada, em intervalos regulares ao longo de 10 dias de simulação. O objetivo foi verificar se o modelo, sem fatores aleatórios, era capaz de atender praticamente toda a demanda, com backlog próximo de zero.

#### 1.5. Resultados Obtidos (Output do Script)

```python
summary_df
```

Com base nos resultados, observa-se que:

- O número de serviços realizados é compatível com a demanda cadastrada e com a frota disponível.
- O volume total atendido é próximo da soma das demandas, indicando que o modelo opera de forma coerente em um cenário simples.
- Os tempos médios de abastecimento são razoáveis e dentro da ordem de grandeza esperada para operações de bunkering.

### 2.2.2. Teste 2: Extremos de Frota (1 vs 10 barcaças)

No teste T2, foram avaliados dois cenários extremos de frota:

- **T2_1barcaça:** frota mínima, com apenas 1 barcaça.
- **T2_10barcaças:** frota ampliada, com 10 barcaças.

O objetivo foi verificar o impacto da variação da frota sobre o backlog e o volume atendido, mantendo a mesma demanda e horizonte de simulação.

```python
summary_df[summary_df['Teste'] == 'teste_T2']
```

Os resultados mostram:

- Com 1 barcaça, o backlog é significativamente maior, indicando incapacidade de atender toda a demanda.
- Com 10 barcaças, o volume atendido aumenta substancialmente e o backlog é drasticamente reduzido.
- A taxa de atendimento se aproxima de 100% no cenário de frota ampliada, em linha com o comportamento esperado de um sistema limitado por capacidade de recurso.

### 2.2.3. Teste 3: Aumento da Demanda (Cenário de Stress)

O teste T3 incluiu um aumento de demanda para os mesmos ou novos navios, com o propósito de estressar o sistema e observar se o backlog cresce de forma consistente com a limitação de frota.

```python
summary_df[summary_df['Teste'] == 'teste_T3']
```

Principais observações:

- O backlog torna-se expressivo, atingindo centenas de toneladas.
- A taxa de atendimento cai em relação ao cenário de sanidade (T1), evidenciando que a frota é o gargalo do sistema.
- A diferença entre demanda e volume atendido confirma que o modelo responde adequadamente ao aumento de carga.

## 2.3 Scripts de Geração de Gráficos

Para complementar a verificação, foram produzidos gráficos que facilitam a análise visual dos indicadores de desempenho (número de serviços, volume atendido, duração média, taxa de atendimento, etc.).

```python
# Cria rótulos legíveis combinando identificador do teste e cenário, para facilitar a leitura dos gráficos.
labels = summary_df['Teste_Cenario']

plt.figure()
plt.bar(labels, summary_df['Serviços realizados'])
plt.ylabel('Serviços realizados')
# Gera um gráfico de barras com o número total de serviços realizados em cada teste/cenário.
plt.title('Serviços realizados por teste/cenário')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Gera um gráfico de barras com o volume total atendido (em toneladas) em cada teste/cenário.
plt.figure()
plt.bar(labels, summary_df['Volume atendido (t)'])
plt.ylabel('Volume atendido (t)')
plt.title('Volume total atendido por teste/cenário')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Gera um boxplot para analisar a distribuição das durações de abastecimento em cada teste/cenário.
plt.figure()
plt.boxplot(duration_data.values(), labels=duration_data.keys())
plt.ylabel('Duração do abastecimento (min)')
plt.title('Distribuição das durações de abastecimento por teste')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Gera um boxplot para analisar a distribuição do volume atendido por serviço em cada teste/cenário.
plt.figure()
plt.boxplot(volume_data.values(), labels=volume_data.keys())
plt.ylabel('Volume atendido por serviço (t)')
plt.title('Distribuição do volume atendido por serviço por teste')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Gera um gráfico de barras com a taxa de atendimento (percentual de demanda atendida) em cada teste/cenário.
plt.figure()
plt.bar(labels, summary_df['Taxa de atendimento (%)'])
plt.ylabel('Taxa de atendimento (%)')
plt.title('Taxa de atendimento por teste/cenário')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Gera um gráfico de barras com o volume médio atendido por serviço em cada teste/cenário.
plt.figure()
plt.bar(labels, summary_df['Volume médio por serviço (t/serviço)'])
plt.ylabel('Volume médio por serviço (t/serviço)')
plt.title('Volume médio por serviço por teste/cenário')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
```

Os gráficos gerados permitem:

- Visualizar rapidamente quais cenários apresentam melhor desempenho (maior volume atendido, maior taxa de atendimento).
- Identificar dispersão de durações e volumes por serviço (via boxplots), o que auxilia na detecção de outliers ou comportamentos inesperados.
- Comparar a eficiência relativa de diferentes configurações de frota e demanda.

## 3. Plano de Validação

A validação teve como foco avaliar se o modelo representa adequadamente a operação real de bunkering, dentro das simplificações adotadas. Como não há uma base de dados real de operações para comparação direta, foi utilizado um conjunto de checks qualitativos e quantitativos, tais como:

- Coerência entre tempos previstos de operação e tempos obtidos em simulação.
- Compatibilidade dos volumes movimentados com a capacidade típica de barcaças e navios.
- Comparação com indicadores de desempenho esperados em uma operação portuária de bunker (por exemplo, níveis de utilização da frota, backlog aceitável).

### 3.1 Evidências de Validação

Durante a validação, foram analisados:

- Tempos médios de abastecimento por serviço e sua variabilidade.
- Distribuição de backlog ao longo do tempo nos cenários de stress.
- Níveis de utilização de barcaças (número de serviços por barcaça em determinado horizonte).

Embora não tenha sido possível calibrar o modelo diretamente com dados reais de uma operação de bunkering específica, os resultados foram considerados plausíveis e coerentes com o comportamento esperado de um sistema de transporte com limitação de frota, o que dá suporte à validade do modelo para fins de análise de cenários e apoio à decisão no contexto do Laboratório.

## 4. Conclusão da Verificação e Validação

A etapa de verificação mostrou que:

- O código dos notebooks de simulação está funcional, gerando arquivos de log consistentes e estruturados.
- As métricas calculadas (número de serviços, volume atendido, backlog, taxa de atendimento, etc.) são coerentes com as configurações de entrada.
- O modelo responde de forma adequada a variações de parâmetros, como tamanho da frota e carga de demanda.

A etapa de validação, ainda que limitada pela ausência de dados reais, indicou que:

- Os resultados simulados se comportam de acordo com a lógica do sistema modelado.
- Os indicadores produzidos são condizentes com o que se espera de uma operação portuária de bunkering sujeita a restrições de capacidade.

Dessa forma, o modelo pode ser utilizado com segurança para explorar cenários e apoiar decisões sobre dimensionamento de frota, níveis de demanda e estratégias operacionais dentro do escopo do laboratório.
