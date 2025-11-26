# G106 - Lab 12 - Execução e Análise dos Experimentos

**Disciplina:** Laboratório de Modelagem e Simulação  
**Projeto:** Abastecimento de Navios por Barcaças - Simulação Discreta  
**Objetivo:** Avaliar o impacto de diferentes configurações operacionais (capacidade de berço, nível de demanda e configuração de frota de barcaças) sobre o desempenho do sistema de abastecimento da BVE, por meio de um plano fatorial 2×3×4 com replicações.

**Integrantes do Grupo:**
- Beatriz Molinari Satil de Souza (12551067)  
- Davi Vieira Zandomeneghi (12553837)  
- Samuel Roizenblatt Davidovici (14607211)
---

#1. Análise fatorial 2×3×4 – Projeto BVE

Este notebook em R lê os arquivos `log_atendimentos_barcacas_XXX.csv`,
monta o plano fatorial 2×3×4 com 5 réplicas e executa:
- ANOVA fatorial para as respostas `makespan`, `utilizacao_media` e `atraso_acum`;
- gráficos de efeitos principais e de interação para os fatores `berco`, `demanda` e `frota`;
- gráfico de Pareto dos efeitos padronizados;
- intervalos de confiança por cenário, acompanhados de boxplots;
- testes Tukey HSD para comparação múltipla de médias;
- diagnóstico de resíduos (normalidade e homocedasticidade).

Os fatores considerados são:
- **Berço (`berco`)**: Sem / Com capacidade adicional de atracação;
- **Demanda (`demanda`)**: Base / +20% / +50%;
- **Frota (`frota`)**: diferentes combinações de número de barcaças e capacidade de bomba (F1–F4).


#2. Resultados visuais (gráficos e tabelas)

Nesta seção são explorados, de forma gráfica, os resultados obtidos a partir da base consolidada `dados_exp`. Os principais recursos utilizados são:

- **Histogramas** das respostas, para inspecionar a distribuição das métricas ao longo de todas as execuções;
- **Boxplots por fator** (por exemplo, `demanda` vs. `utilizacao_media`), para comparar níveis de um mesmo fator;
- **Boxplots de interação** (por exemplo, `frota` × `berco`), para verificar efeitos combinados;
- **Tabelas resumo** das métricas por cenário.

Os códigos nas células seguintes permitem regenerar todos os gráficos sempre que novos arquivos de simulação forem adicionados.


#3. Análises de resultados

A análise estatística é baseada na ANOVA fatorial aplicada às métricas de resposta `makespan`, `utilizacao_media` e, quando disponível, `atraso_acum`.

Antes de interpretar os efeitos dos fatores, é necessário verificar as premissas do modelo de ANOVA, a partir dos testes e gráficos gerados pelo código:

### 3.1. Validação das premissas da ANOVA

- **Normalidade dos resíduos**: avaliada pelo teste de Shapiro–Wilk e pelo gráfico QQ-plot dos resíduos;
- **Homogeneidade das variâncias**: avaliada pelo teste de Levene e pelo gráfico de resíduos vs. valores ajustados.

Caso essas premissas sejam atendidas (p‑valores acima do nível de significância adotado e ausência de padrões sistemáticos nos gráficos), os resultados da ANOVA podem ser utilizados para:

- identificar quais fatores (`berco`, `demanda`, `frota`) têm efeito significativo em cada resposta;
- verificar a presença de interações relevantes entre os fatores;
- apoiar a escolha das combinações mais adequadas aos objetivos do sistema (reduzir `makespan`, aumentar `utilizacao_media` ou reduzir `atraso_acum`).

Os valores numéricos específicos (estatísticas F, p‑valores e intervalos de confiança) devem ser lidos diretamente dos outputs impressos quando as funções de análise são executadas.


# 4. Código e outputs "crus" do modelo:
## Leitura e agregação dos arquivos CSV das simulações

Nas células de código a seguir está implementado o fluxo de análise em R:

1. Localização e leitura de todos os arquivos `log_atendimentos_barcacas_XXX.csv` gerados pelo simulador;
2. Cálculo, para cada execução (combinação de cenário e réplica), das métricas de desempenho:
   - `makespan` (tempo total da operação);
   - `utilizacao_media` (taxa média de ocupação das barcaças);
   - `atraso_acum` (quando registrado nos logs).
3. Associação de cada execução aos fatores do plano fatorial 2×3×4 (`berco`, `demanda`, `frota`), produzindo a base consolidada `dados_exp`.

A partir dessa base, são executadas automaticamente:

- a ANOVA fatorial;
- os gráficos de efeitos principais e de interação;
- o gráfico de Pareto dos efeitos padronizados;
- os boxplots e intervalos de confiança por cenário;
- os gráficos de diagnóstico de resíduos.

Sempre que novos logs forem gerados, basta manter o mesmo padrão de nomes dos arquivos e reexecutar o notebook para atualizar toda a análise.


```r
# Script principal: análise fatorial 2×3×4 do projeto Buena Vista Energy (BVE)
# Lê os arquivos `log_atendimentos_barcacas_XXX.csv` e calcula as métricas de desempenho por execução
# Associa cada execução ao respectivo cenário do plano fatorial (berço, demanda, frota) e, em seguida, aplica:
# - ANOVA fatorial para cada métrica de resposta
# - Gráficos de efeitos principais e de interação entre os fatores
# - Gráfico de Pareto com os efeitos padronizados
# - Intervalos de confiança por cenário e boxplots das respostas
# - Testes de comparação múltipla (Tukey HSD)
# - Diagnóstico de resíduos (normalidade e homocedasticidade)

# Pacotes necessários ----------------------------------------------------
packages <- c("tidyverse", "broom", "car")
inst <- packages %in% rownames(installed.packages())
if (any(!inst)) {
  install.packages(packages[!inst])
}
invisible(lapply(packages, library, character.only = TRUE))

```

```r
# Análise gráfica exploratória das métricas de resposta -----------------

# Carregamento dos pacotes de visualização (já incluídos no tidyverse)
library(ggplot2)

# Neste exemplo, assumimos duas métricas principais: `utilizacao_media` e `makespan`
# Se o nome da coluna for diferente, substitua `utilizacao_media` pela métrica desejada.

# -------------------------------------------------------------------
# Histograma da métrica escolhida (visualiza a distribuição dos valores)
# -------------------------------------------------------------------
# O histograma ajuda a verificar a distribuição geral dos resultados
# de uma métrica específica (por exemplo, `utilizacao_media`) ao longo de todas as
# replicações e cenários simulados.

ggplot(dados_exp, aes(x = utilizacao_media)) +
  geom_histogram(
    binwidth = 0.01, # Ajuste a largura das 'barras' (bins)
    fill = "steelblue",
    color = "black",
    alpha = 0.8
  ) +
  labs(
    title = "Distribuição da Métrica: Utilização Média",
    x = "Utilização Média (Todas as Execuções)",
    y = "Frequência (Contagem)"
  ) +
  theme_minimal()

# -------------------------------------------------------------------
# Boxplots para comparar o efeito dos fatores sobre a métrica
# -------------------------------------------------------------------
# O boxplot é útil para comparar o impacto dos fatores de projeto
# (por exemplo, `demanda`) sobre a métrica de resposta (por exemplo, `utilizacao_media`).

# Boxplot simples: métrica de resposta em função do fator `demanda`
ggplot(dados_exp, aes(x = demanda, y = utilizacao_media, fill = demanda)) +
  geom_boxplot() +
  labs(
    title = "Impacto da Demanda na Utilização Média",
    x = "Nível de Demanda",
    y = "Utilização Média"
  ) +
  theme_minimal() +
  theme(legend.position = "none") # Legenda desnecessária aqui

# -------------------------------------------------------------------
# Boxplot avançado: visualização de possível interação entre fatores
# -------------------------------------------------------------------
# Este gráfico permite observar de forma visual a possível interação
# entre dois fatores (por exemplo, `frota` e `berco`) na métrica analisada.

ggplot(dados_exp, aes(x = frota, y = utilizacao_media, fill = berco)) +
  geom_boxplot() +
  # O `facet_wrap` divide o gráfico em painéis separados
  # exibindo um painel para cada nível do fator `berco`.
  facet_wrap(~ berco) +
  labs(
    title = "Interação: Utilização Média vs. Frota e Berço",
    x = "Configuração de Frota",
    y = "Utilização Média",
    fill = "Configuração de Berço"
  ) +
  theme_minimal()
```

> **Leitura dos gráficos:** o primeiro gráfico mostra a distribuição da métrica selecionada (histograma),
> o segundo compara a métrica entre os níveis de `demanda` (boxplot simples) e o terceiro explora a
> interação entre `frota` e `berco` (boxplot facetado). Use esses gráficos para identificar padrões,
> assimetrias e possíveis interações relevantes.

> (*Histograma da métrica de resposta. Reexecute o código no R/Jupyter para visualizar a figura.*)

> (*Boxplot da métrica por nível de `demanda`. Reexecute o código no R/Jupyter para visualizar a figura.*)

> (*Boxplot com interação entre `frota` e `berco` (facetado). Reexecute o código no R/Jupyter para visualizar a figura.*)

## Cálculo das métricas de desempenho (respostas)

Com os dados carregados, extraímos e calculamos as métricas de desempenho selecionadas:

- **Atraso acumulado**: volume de carga não atendido ou atendido com atraso, refletindo a qualidade de serviço.
- **Makespan**: tempo total até o término do atendimento, refletindo a eficiência operacional do sistema.
- **Utilização média** (ou taxa de ocupação das barcaças): importante para avaliar o uso de recursos disponíveis.

Essas métricas são calculadas por cenário e replicação, para permitir comparações robustas e análise estatística posterior.

```r

# Parâmetros gerais da análise ----------------------------------------
# Caminho da pasta onde estão os arquivos de log exportados pela simulação
caminho_dados   <- "."
# Padrão de nomes dos arquivos de log (ajuste se estiver diferente)
padrao_arquivos <- "^log_atendimentos_barcacas_\\d+\\.csv$"

# Número de réplicas por cenário no experimento (plano atual: 5)
n_replicas <- 5

# Nível de significância adotado para os testes estatísticos (α)
alpha <- 0.05

```

## Visualização dos resultados dos experimentos

Utilizamos representações gráficas para comparar os diferentes cenários e níveis dos fatores do plano fatorial. Entre os principais gráficos, destacam‑se:

- **Histogramas** das respostas para inspeção visual das distribuições;
- **Boxplots por fator** (por exemplo, `demanda` vs. `utilizacao_media`);
- **Boxplots de interação** (por exemplo, `frota` × `berco`);
- **Gráficos de efeito principal** e **gráficos de interação** gerados pela função `analisar_resposta()`;
- **Boxplots por cenário** com indicação de variabilidade.

Essas visualizações ajudam a interpretar como cada fator e suas combinações afetam o desempenho do sistema.


```r
# Definição do plano fatorial 2×3×4 -----------------------------------
# Fator 1 – Capacidade de berço: cenários **Sem** e **Com** berço adicional
# Fator 2 – Demanda de abastecimento: níveis **Base**, **+20%** e **+50%**
# Fator 3 – Frota de barcaças (configuração de número de embarcações e vazão):
#   F1 – 6 barcaças (BVE-01 a BVE-06) com bombas de 200 m³/h
#   F2 – 6 barcaças (BVE-01 a BVE-06) com bombas de 240 m³/h
#   F3 – 7 barcaças (BVE-01 a BVE-07) com bombas de 200 m³/h
#   F4 – 7 barcaças (BVE-01 a BVE-07) com bombas de 240 m³/h
# Cada combinação de níveis define um cenário do experimento fatorial.
# O plano completo resulta em 2×3×4 = 24 cenários distintos.

design <- expand.grid(
  berco   = c("Sem", "Com"),
  demanda = c("Base", "+20%", "+50%"),
  frota   = c("F1", "F2", "F3", "F4"),
  KEEP.OUT.ATTRS = FALSE,
  stringsAsFactors = FALSE
) %>%
  as_tibble() %>%
  arrange(berco, demanda, frota) %>%
  mutate(cenario = dplyr::row_number()) %>%
  select(cenario, berco, demanda, frota)

# Cada cenário é replicado `n_replicas` vezes para estimar a variabilidade das respostas.
design_rep <- design %>%
  slice(rep(dplyr::row_number(), each = n_replicas)) %>%
  mutate(replica = rep(seq_len(n_replicas), times = nrow(design)))
```

## Conclusão preliminar da análise

O plano fatorial 2×3×4 permite avaliar, de forma sistemática, o efeito conjunto de:

- adicionar ou não capacidade de **berço**;
- variar a **demanda** (Base, +20%, +50%);
- alterar a configuração da **frota** de barcaças (F1–F4).

A partir dos resultados de ANOVA, dos gráficos de efeitos principais e de interação e dos boxplots por cenário, é possível:

- identificar quais fatores têm maior impacto sobre `makespan`, `utilizacao_media` e `atraso_acum`;
- verificar se há interações importantes entre `berco`, `demanda` e `frota`;
- selecionar combinações operacionais mais adequadas ao objetivo do sistema (por exemplo, minimizar `makespan` sem sobrecarregar a frota, ou maximizar a utilização dentro de limites aceitáveis de atraso).

As conclusões numéricas específicas (cenário considerado mais favorável, ganhos percentuais, etc.) devem ser extraídas diretamente dos outputs gerados quando o notebook é executado com os logs definitivos.


```r

# Função auxiliar para calcular métricas de desempenho por execução ----
# Espera-se que cada arquivo CSV contenha, no mínimo, as colunas:
# `id_navio`, `terminal`, `demanda`, `id_barca`,
# `ini_abastecimento`, `fim_abastecimento`, `volume_atendido`, `status`
# (e, opcionalmente, alguma coluna relacionada a atraso/tempo de espera)

calcular_metricas_execucao <- function(caminho_arquivo) {
  df <- readr::read_csv(caminho_arquivo, show_col_types = FALSE)

  # Mantém apenas os atendimentos concluídos, caso a coluna de status exista
  if ("status" %in% names(df)) {
    df <- df %>% dplyr::filter(status == "concluido")
  }

  if (nrow(df) == 0) {
    warning("Arquivo sem registros concluídos: ", caminho_arquivo)
    return(tibble(
      arquivo          = basename(caminho_arquivo),
      makespan         = NA_real_,
      utilizacao_media = NA_real_,
      atraso_acum      = NA_real_
    ))
  }

  # Confere se as colunas de tempo existem antes de calcular as durações
  if (!all(c("ini_abastecimento", "fim_abastecimento") %in% names(df))) {
    stop("Colunas 'ini_abastecimento' e/ou 'fim_abastecimento' não encontradas em ",
         caminho_arquivo)
  }

  tempo_ini <- min(df$ini_abastecimento, na.rm = TRUE)
  tempo_fim <- max(df$fim_abastecimento, na.rm = TRUE)
  makespan  <- tempo_fim - tempo_ini

  duracao_sim <- makespan

  # Calcula a utilização média da frota de barcaças ao longo da simulação
  util_por_barca <- df %>%
    group_by(id_barca) %>%
    summarise(
      tempo_ocupado = sum(fim_abastecimento - ini_abastecimento, na.rm = TRUE),
      .groups = "drop"
    ) %>%
    mutate(utilizacao = tempo_ocupado / duracao_sim)

  utilizacao_media <- mean(util_por_barca$utilizacao, na.rm = TRUE)

  # Calcula o atraso acumulado tentando identificar automaticamente a coluna de espera/atraso, se existir
  possiveis_colunas_atraso <- intersect(
    c("atraso", "tempo_espera", "tempo_fila", "waiting_time"),
    names(df)
  )

  atraso_acum <- if (length(possiveis_colunas_atraso) > 0) {
    sum(df[[possiveis_colunas_atraso[1]]], na.rm = TRUE)
  } else {
    NA_real_
  }

  tibble(
    arquivo          = basename(caminho_arquivo),
    makespan         = as.numeric(makespan),
    utilizacao_media = as.numeric(utilizacao_media),
    atraso_acum      = atraso_acum
  )
}


```

```r
# Leitura de todos os arquivos de log e montagem da base consolidada ----
arquivos_log <- list.files(
  path       = caminho_dados,
  pattern    = padrao_arquivos,
  full.names = TRUE
) %>%
  sort()

if (length(arquivos_log) == 0) {
  stop("Nenhum arquivo encontrado com o padrão: ", padrao_arquivos)
}

# Para cada arquivo de log, calcula as métricas por execução e empilha os resultados
metricas_exec <- purrr::map_dfr(
  arquivos_log,
  calcular_metricas_execucao
) %>%
  dplyr::mutate(execucao = dplyr::row_number())

# Verificação de consistência: deve haver 24 cenários × `n_replicas` linhas na base
if (nrow(metricas_exec) != nrow(design_rep)) {
  warning(
    "Número de arquivos (", nrow(metricas_exec),
    ") diferente do nº de linhas do plano (",
    nrow(design_rep), "). A associação será feita pela ordem."
  )
}

# Junta a base de métricas com o plano fatorial (berço, demanda e frota)
dados_exp <- design_rep %>%
  dplyr::mutate(execucao = dplyr::row_number()) %>%
  dplyr::left_join(metricas_exec, by = "execucao") %>%
  dplyr::mutate(
    berco   = factor(berco,   levels = c("Sem", "Com")),
    demanda = factor(demanda, levels = c("Base", "+20%", "+50%")),
    frota   = factor(frota,   levels = c("F1", "F2", "F3", "F4")),
    cenario = factor(cenario)
  )
```

```r

# Funções auxiliares para geração de gráficos e execução da ANOVA -------

plot_efeito_principal <- function(df, resposta, fator) {
  resumo <- df %>%
    group_by_at(fator) %>%
    summarise(
      media = mean(get(resposta), na.rm = TRUE),
      sd    = sd(get(resposta), na.rm = TRUE),
      n     = dplyr::n(),
      se    = sd / sqrt(n),
      .groups = "drop"
    )

  ggplot(resumo, aes_string(x = fator, y = "media", group = 1)) +
    geom_line() +
    geom_point(size = 2) +
    geom_errorbar(
      aes(ymin = media - 1.96 * se,
          ymax = media + 1.96 * se),
      width = 0.1
    ) +
    labs(
      title = paste("Efeito principal de", fator, "em", resposta),
      x = fator,
      y = paste("Média de", resposta)
    ) +
    theme_minimal()
}

plot_interacao <- function(df, resposta, fator_x, fator_traco) {
  resumo <- df %>%
    group_by_at(c(fator_x, fator_traco)) %>%
    summarise(
      media = mean(get(resposta), na.rm = TRUE),
      .groups = "drop"
    )

  ggplot(resumo,
         aes_string(x = fator_x,
                    y = "media",
                    color = fator_traco,
                    group = fator_traco)) +
    geom_line() +
    geom_point() +
    labs(
      title = paste("Interação", fator_x, "x", fator_traco, "em", resposta),
      x = fator_x,
      y = paste("Média de", resposta),
      color = fator_traco
    ) +
    theme_minimal()
}

analisar_resposta <- function(dados, resposta, alpha = 0.05) {
  cat("\n\n==============================\n")
  cat("Análise da resposta:", resposta, "\n")
  cat("==============================\n")

  # 1) Se todos os valores da resposta forem NA, a análise dessa métrica é pulada
  if (all(is.na(dados[[resposta]]))) {
    cat("Variável", resposta, "está completamente NA. Análise ignorada.\n")
    return(invisible(NULL))
  }

  # 2) Trabalha apenas com as linhas que têm resposta observada (não-NA)
  df <- dados %>%
    dplyr::filter(!is.na(.data[[resposta]]))

  if (nrow(df) < 3) {
    cat("Poucas observações com valor não-NA para", resposta,
        "(n =", nrow(df), "). ANOVA ignorada por enquanto.\n")
    return(invisible(NULL))
  }

  # 3) Verifica quantos níveis cada fator possui na base filtrada
  fatores <- c("berco", "demanda", "frota")

  n_niveis <- sapply(fatores, function(f) {
    if (!f %in% names(df)) return(0L)
    nlevels(droplevels(df[[f]]))
  })

  fatores_validos <- fatores[n_niveis >= 2]

  if (length(fatores_validos) == 0) {
    cat("Nenhum fator possui 2 ou mais níveis entre as observações válidas.\n")
    cat("Níveis por fator:\n")
    print(n_niveis)
    cat("Quando houver mais cenários/CSV com variação nos fatores, a ANOVA passa a funcionar.\n")
    return(invisible(NULL))
  }

  if (length(fatores_validos) < length(fatores)) {
    cat("Atenção: alguns fatores não têm 2 níveis e serão removidos do modelo:\n")
    print(n_niveis)
  }

  # 4) Ajusta um modelo fatorial incluindo apenas os fatores que possuem dois ou mais níveis
  rhs <- paste(fatores_validos, collapse = " * ")
  formula_exp <- as.formula(
    paste(resposta, "~", rhs)
  )

  modelo_aov <- aov(formula_exp, data = df)

  cat("\n--- ANOVA fatorial ---\n")
  print(summary(modelo_aov))

  # 5) Gera gráficos de efeitos principais apenas para os fatores válidos
  plots_main <- lapply(fatores_validos, function(f) {
    plot_efeito_principal(df, resposta, f)
  })
  names(plots_main) <- fatores_validos

  # 6) Gera gráficos de interação 2 a 2 apenas entre fatores válidos
  if (length(fatores_validos) >= 2) {
    comb <- combn(fatores_validos, 2, simplify = FALSE)
    plots_inter <- lapply(comb, function(par) {
      plot_interacao(df, resposta, par[1], par[2])
    })
    names(plots_inter) <- sapply(comb, paste, collapse = " x ")
  } else {
    plots_inter <- list()
  }

  # 7) Calcula os efeitos padronizados a partir de um modelo linear (base para o Pareto)
  modelo_lm <- lm(formula_exp, data = df)
  efeitos <- broom::tidy(modelo_lm) %>%
    dplyr::filter(term != "(Intercept)") %>%
    dplyr::mutate(
      efeito        = estimate,
      efeito_std    = abs(estimate / std.error),
      termo_legivel = term
    ) %>%
    dplyr::arrange(dplyr::desc(efeito_std))

  cat("\n--- Efeitos padronizados (para Pareto) ---\n")
  print(efeitos)

  plot_pareto <- ggplot(efeitos,
                        aes(x = reorder(termo_legivel, efeito_std),
                            y = efeito_std)) +
    geom_bar(stat = "identity") +
    coord_flip() +
    labs(
      title = paste("Pareto dos efeitos padronizados –", resposta),
      x = "Termo (fator / interação)",
      y = "|t-valor|"
    ) +
    theme_minimal()

  # 8) Calcula, para cada cenário, média, desvio padrão e IC de 95% da resposta
  resumo_cenarios <- df %>%
    dplyr::group_by(cenario, berco, demanda, frota) %>%
    dplyr::summarise(
      media  = mean(.data[[resposta]], na.rm = TRUE),
      sd     = sd(.data[[resposta]], na.rm = TRUE),
      n      = dplyr::n(),
      se     = sd / sqrt(n),
      t_crit = qt(1 - alpha/2, df = n - 1),
      ic_inf = media - t_crit * se,
      ic_sup = media + t_crit * se,
      .groups = "drop"
    )

  cat("\n--- Resumo por cenário (IC 95%) ---\n")
  print(resumo_cenarios)

  box_cenario <- ggplot(df,
                        aes_string(x = "cenario", y = resposta)) +
    geom_boxplot() +
    labs(
      title = paste("Distribuição de", resposta, "por cenário"),
      x = "Cenário",
      y = resposta
    ) +
    theme_minimal()

  # 9) Aplica Tukey HSD apenas para os fatores que permaneceram no modelo ajustado
  tukey <- lapply(fatores_validos, function(f) {
    TukeyHSD(modelo_aov, which = f)
  })
  names(tukey) <- fatores_validos

  # 10) Realiza o diagnóstico de resíduos (QQ-plot, gráfico de resíduos vs. ajustados, Shapiro e Levene)
  residuos  <- residuals(modelo_aov)
  ajustados <- fitted(modelo_aov)

  diag_df <- data.frame(ajustados = ajustados, residuos = residuos)

  plot_resid <- ggplot(diag_df,
                       aes(x = ajustados, y = residuos)) +
    geom_point() +
    geom_hline(yintercept = 0, linetype = 2) +
    labs(
      title = paste("Resíduos vs ajustados –", resposta),
      x = "Valores ajustados",
      y = "Resíduos"
    ) +
    theme_minimal()

  plot_qq <- ggplot(data.frame(residuos = residuos),
                    aes(sample = residuos)) +
    stat_qq() +
    stat_qq_line() +
    labs(
      title = paste("QQ-plot dos resíduos –", resposta),
      x = "Quantis teóricos",
      y = "Quantis amostrais"
    ) +
    theme_minimal()

  teste_shapiro <- shapiro.test(residuos)
  teste_levene  <- car::leveneTest(formula_exp, data = df)

  cat("\n--- Teste de normalidade (Shapiro-Wilk) ---\n")
  print(teste_shapiro)
  cat("\n--- Teste de homocedasticidade (Levene) ---\n")
  print(teste_levene)

  invisible(list(
    modelo_aov       = modelo_aov,
    modelo_lm        = modelo_lm,
    efeitos          = efeitos,
    plots_main       = plots_main,
    plots_inter      = plots_inter,
    plot_pareto      = plot_pareto,
    resumo_cenarios  = resumo_cenarios,
    boxplot_cenario  = box_cenario,
    tukey            = tukey,
    diagnostico      = list(
      residuos      = residuos,
      ajustados     = ajustados,
      plot_residuos = plot_resid,
      plot_qq       = plot_qq,
      shapiro       = teste_shapiro,
      levene        = teste_levene
    )
  ))
}

```

```r
metricas <- c("makespan", "utilizacao_media", "atraso_acum")
resultados <- lapply(metricas, function(m) {
  analisar_resposta(dados_exp, resposta = m, alpha = alpha)
})

```

> **Leitura das saídas a seguir:** para cada métrica (`makespan`, `utilizacao_media`, `atraso_acum`),
> o script imprime a ANOVA fatorial, os efeitos padronizados (base para o gráfico de Pareto) e mensagens
> de diagnóstico (por exemplo, avisos sobre colunas com muitos valores ausentes). Essas informações são
> a base numérica para interpretar quais fatores são mais relevantes para cada resposta.

```text


==============================
Análise da resposta: makespan 
==============================

--- ANOVA fatorial ---
                    Df Sum Sq Mean Sq F value  Pr(>F)   
berco                1   11.6   11.63   1.752 0.18880   
demanda              2   12.6    6.28   0.947 0.39168   
frota                3   29.8    9.94   1.497 0.22025   
berco:demanda        2   95.4   47.72   7.189 0.00123 **
berco:frota          3    9.8    3.25   0.490 0.69013   
demanda:frota        6   17.3    2.88   0.433 0.85501   
berco:demanda:frota  6   34.2    5.71   0.860 0.52754   
Residuals           96  637.3    6.64                   
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

```

```text
Warning message:
“[1m[22m`aes_string()` was deprecated in ggplot2 3.0.0.
[36mℹ[39m Please use tidy evaluation idioms with `aes()`.
[36mℹ[39m See also `vignette("ggplot2-in-packages")` for more information.”

```

```text

--- Efeitos padronizados (para Pareto) ---
[90m# A tibble: 23 × 8[39m
   term     estimate std.error statistic p.value efeito efeito_std termo_legivel
   [3m[90m<chr>[39m[23m       [3m[90m<dbl>[39m[23m     [3m[90m<dbl>[39m[23m     [3m[90m<dbl>[39m[23m   [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m      [3m[90m<dbl>[39m[23m [3m[90m<chr>[39m[23m        
[90m 1[39m frotaF2     -[31m2[39m[31m.[39m[31m78[39m      1.63    -[31m1[39m[31m.[39m[31m71[39m   0.091[4m4[24m  -[31m2[39m[31m.[39m[31m78[39m      1.71  frotaF2      
[90m 2[39m frotaF4     -[31m2[39m[31m.[39m[31m40[39m      1.63    -[31m1[39m[31m.[39m[31m47[39m   0.144   -[31m2[39m[31m.[39m[31m40[39m      1.47  frotaF4      
[90m 3[39m bercoCo…    -[31m4[39m[31m.[39m[31m0[39m[31m7[39m      3.26    -[31m1[39m[31m.[39m[31m25[39m   0.215   -[31m4[39m[31m.[39m[31m0[39m[31m7[39m      1.25  bercoCom:dem…
[90m 4[39m demanda…     2.68      2.30     1.16   0.248    2.68      1.16  demanda+20%:…
[90m 5[39m demanda…    -[31m1[39m[31m.[39m[31m78[39m      1.63    -[31m1[39m[31m.[39m[31m0[39m[31m9[39m   0.278   -[31m1[39m[31m.[39m[31m78[39m      1.09  demanda+20%  
[90m 6[39m demanda…     2.47      2.30     1.07   0.286    2.47      1.07  demanda+50%:…
[90m 7[39m bercoCo…     2.44      2.30     1.06   0.292    2.44      1.06  bercoCom:fro…
[90m 8[39m frotaF3     -[31m1[39m[31m.[39m[31m71[39m      1.63    -[31m1[39m[31m.[39m[31m0[39m[31m5[39m   0.297   -[31m1[39m[31m.[39m[31m71[39m      1.05  frotaF3      
[90m 9[39m bercoCo…     2.13      2.30     0.924  0.358    2.13      0.924 bercoCom:fro…
[90m10[39m bercoCo…    -[31m2[39m[31m.[39m[31m87[39m      3.26    -[31m0[39m[31m.[39m[31m881[39m  0.381   -[31m2[39m[31m.[39m[31m87[39m      0.881 bercoCom:dem…
[90m# ℹ 13 more rows[39m

--- Resumo por cenário (IC 95%) ---
[90m# A tibble: 24 × 11[39m
   cenario berco demanda frota media    sd     n    se t_crit ic_inf ic_sup
   [3m[90m<fct>[39m[23m   [3m[90m<fct>[39m[23m [3m[90m<fct>[39m[23m   [3m[90m<fct>[39m[23m [3m[90m<dbl>[39m[23m [3m[90m<dbl>[39m[23m [3m[90m<int>[39m[23m [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m
[90m 1[39m 1       Com   +20%    F1     162.  1.29     5 0.577   2.78   161.   164.
[90m 2[39m 2       Com   +20%    F2     162.  1.51     5 0.676   2.78   160.   164.
[90m 3[39m 3       Com   +20%    F3     163.  1.19     5 0.533   2.78   161.   164.
[90m 4[39m 4       Com   +20%    F4     162.  2.56     5 1.15    2.78   159.   165.
[90m 5[39m 5       Com   +50%    F1     161.  2.89     5 1.29    2.78   158.   165.
[90m 6[39m 6       Com   +50%    F2     161.  3.13     5 1.40    2.78   157.   165.
[90m 7[39m 7       Com   +50%    F3     160.  4.12     5 1.84    2.78   154.   165.
[90m 8[39m 8       Com   +50%    F4     159.  2.46     5 1.10    2.78   156.   162.
[90m 9[39m 9       Com   Base    F1     163.  1.79     5 0.802   2.78   161.   165.
[90m10[39m 10      Com   Base    F2     162.  1.27     5 0.570   2.78   161.   164.
[90m# ℹ 14 more rows[39m

--- Teste de normalidade (Shapiro-Wilk) ---

	Shapiro-Wilk normality test

data:  residuos
W = 0.94037, p-value = 4.537e-05


--- Teste de homocedasticidade (Levene) ---
Levene's Test for Homogeneity of Variance (center = median)
      Df F value Pr(>F)
group 23   0.608 0.9137
      96               


==============================
Análise da resposta: utilizacao_media 
==============================

--- ANOVA fatorial ---
                    Df  Sum Sq Mean Sq F value   Pr(>F)    
berco                1 0.13304 0.13304 100.071  < 2e-16 ***
demanda              2 0.08384 0.04192  31.530 2.98e-11 ***
frota                3 0.12856 0.04285  32.233 1.68e-14 ***
berco:demanda        2 0.07284 0.03642  27.395 3.86e-10 ***
berco:frota          3 0.00310 0.00103   0.777   0.5096    
demanda:frota        6 0.00483 0.00081   0.606   0.7253    
berco:demanda:frota  6 0.02778 0.00463   3.483   0.0037 ** 
Residuals           96 0.12763 0.00133                     
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

--- Efeitos padronizados (para Pareto) ---
[90m# A tibble: 23 × 8[39m
   term    estimate std.error statistic p.value  efeito efeito_std termo_legivel
   [3m[90m<chr>[39m[23m      [3m[90m<dbl>[39m[23m     [3m[90m<dbl>[39m[23m     [3m[90m<dbl>[39m[23m   [3m[90m<dbl>[39m[23m   [3m[90m<dbl>[39m[23m      [3m[90m<dbl>[39m[23m [3m[90m<chr>[39m[23m        
[90m 1[39m bercoC…  -[31m0[39m[31m.[39m[31m163[39m     0.032[4m6[24m     -[31m5[39m[31m.[39m[31m0[39m[31m1[39m 2.49[90me[39m[31m-6[39m -[31m0[39m[31m.[39m[31m163[39m        5.01 bercoCom:dem…
[90m 2[39m bercoC…   0.157     0.046[4m1[24m      3.40 9.75[90me[39m[31m-4[39m  0.157        3.40 bercoCom:dem…
[90m 3[39m demand…  -[31m0[39m[31m.[39m[31m0[39m[31m78[4m1[24m[39m    0.023[4m1[24m     -[31m3[39m[31m.[39m[31m39[39m 1.03[90me[39m[31m-3[39m -[31m0[39m[31m.[39m[31m0[39m[31m78[4m1[24m[39m       3.39 demanda+20%  
[90m 4[39m frotaF4  -[31m0[39m[31m.[39m[31m0[39m[31m71[4m8[24m[39m    0.023[4m1[24m     -[31m3[39m[31m.[39m[31m12[39m 2.43[90me[39m[31m-3[39m -[31m0[39m[31m.[39m[31m0[39m[31m71[4m8[24m[39m       3.12 frotaF4      
[90m 5[39m bercoC…   0.139     0.046[4m1[24m      3.01 3.35[90me[39m[31m-3[39m  0.139        3.01 bercoCom:dem…
[90m 6[39m frotaF3  -[31m0[39m[31m.[39m[31m0[39m[31m61[4m5[24m[39m    0.023[4m1[24m     -[31m2[39m[31m.[39m[31m67[39m 8.99[90me[39m[31m-3[39m -[31m0[39m[31m.[39m[31m0[39m[31m61[4m5[24m[39m       2.67 frotaF3      
[90m 7[39m bercoC…  -[31m0[39m[31m.[39m[31m0[39m[31m64[4m9[24m[39m    0.032[4m6[24m     -[31m1[39m[31m.[39m[31m99[39m 4.93[90me[39m[31m-2[39m -[31m0[39m[31m.[39m[31m0[39m[31m64[4m9[24m[39m       1.99 bercoCom:fro…
[90m 8[39m demand…  -[31m0[39m[31m.[39m[31m0[39m[31m57[4m3[24m[39m    0.032[4m6[24m     -[31m1[39m[31m.[39m[31m76[39m 8.21[90me[39m[31m-2[39m -[31m0[39m[31m.[39m[31m0[39m[31m57[4m3[24m[39m       1.76 demanda+50%:…
[90m 9[39m demand…   0.038[4m8[24m    0.023[4m1[24m      1.68 9.54[90me[39m[31m-2[39m  0.038[4m8[24m       1.68 demanda+50%  
[90m10[39m demand…  -[31m0[39m[31m.[39m[31m0[39m[31m42[4m9[24m[39m    0.032[4m6[24m     -[31m1[39m[31m.[39m[31m32[39m 1.92[90me[39m[31m-1[39m -[31m0[39m[31m.[39m[31m0[39m[31m42[4m9[24m[39m       1.32 demanda+50%:…
[90m# ℹ 13 more rows[39m

--- Resumo por cenário (IC 95%) ---
[90m# A tibble: 24 × 11[39m
   cenario berco demanda frota media     sd     n      se t_crit ic_inf ic_sup
   [3m[90m<fct>[39m[23m   [3m[90m<fct>[39m[23m [3m[90m<fct>[39m[23m   [3m[90m<fct>[39m[23m [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m [3m[90m<int>[39m[23m   [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m  [3m[90m<dbl>[39m[23m
[90m 1[39m 1       Com   +20%    F1    0.361 0.061[4m1[24m     5 0.027[4m3[24m    2.78  0.285  0.437
[90m 2[39m 2       Com   +20%    F2    0.300 0.050[4m4[24m     5 0.022[4m5[24m    2.78  0.237  0.362
[90m 3[39m 3       Com   +20%    F3    0.309 0.033[4m1[24m     5 0.014[4m8[24m    2.78  0.268  0.350
[90m 4[39m 4       Com   +20%    F4    0.262 0.024[4m1[24m     5 0.010[4m8[24m    2.78  0.232  0.292
[90m 5[39m 5       Com   +50%    F1    0.290 0.040[4m4[24m     5 0.018[4m1[24m    2.78  0.240  0.341
[90m 6[39m 6       Com   +50%    F2    0.323 0.031[4m6[24m     5 0.014[4m1[24m    2.78  0.283  0.362
[90m 7[39m 7       Com   +50%    F3    0.271 0.036[4m6[24m     5 0.016[4m4[24m    2.78  0.226  0.316
[90m 8[39m 8       Com   +50%    F4    0.253 0.027[4m2[24m     5 0.012[4m2[24m    2.78  0.219  0.287
[90m 9[39m 9       Com   Base    F1    0.415 0.037[4m0[24m     5 0.016[4m5[24m    2.78  0.369  0.461
[90m10[39m 10      Com   Base    F2    0.351 0.019[4m3[24m     5 0.008[4m6[24m[4m1[24m   2.78  0.327  0.375
[90m# ℹ 14 more rows[39m

--- Teste de normalidade (Shapiro-Wilk) ---

	Shapiro-Wilk normality test

data:  residuos
W = 0.98496, p-value = 0.2039


--- Teste de homocedasticidade (Levene) ---
Levene's Test for Homogeneity of Variance (center = median)
      Df F value Pr(>F)
group 23  0.7625 0.7677
      96               


==============================
Análise da resposta: atraso_acum 
==============================
Variável atraso_acum está completamente NA. Análise ignorada.

```

