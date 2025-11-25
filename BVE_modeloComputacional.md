# PRO3342 Modelagem e Simulação de Sistemas de Produção

## Lab 7 - Modelo 1: Operação do Porto Mar Azul


### Nome (número USP) em ordem alfabética

1. Beatriz Molinari Satil de Souza (12551067)
2. Davi Vieira Zandomeneghi (12553837)
3. Giorgio Zamuner (17250748)
4. Ian Barg Lazzaretti (13680042)
5. Samuel Roizenblatt Davidovici (114607211)


### Sumário

1. Problema
2. Modelo
3. Testes
4. Resultados


### 1. Problema

#### Breve descrição da operação a ser modelada

O objetivo deste projeto é desenvolver um modelo de simulação computacional para a operação de navios no "Porto Mar Azul". O modelo deve abranger o ciclo de vida completo de um navio no porto, desde a notificação de sua chegada até sua partida, incluindo as etapas de atracação, operação no terminal e desatracação.


O propósito principal do modelo é gerar uma base de dados sintética, porém realista, da demanda de combustível por parte dos navios. Essa base de dados servirá como entrada para a próxima fase do projeto (Lab 8), que focará na modelagem da operação do terminal de combustível da Buena Vista Energy (BVE).


#### Dados utilizados

O modelo foi parametrizado utilizando os dados de teste fornecidos para a operação do Porto Mar Azul. As informações foram estruturadas em três tipos de navios, cada um com suas próprias características de chegada, operação e demanda de combustível.


**Parâmetros Gerais:**


* **Antecedência de Notificação:** 48 horas antes da chegada.
* **Horizonte de Simulação:** Definido pelo usuário (e.g., 7, 30, 365 dias).


**Parâmetros por Tipo de Navio:**




| Parâmetro | Tipo 1 | Tipo 2 | Tipo 3 |
| --- | --- | --- | --- |
| **Terminais Permitidos** | 7 a 16 | 1 a 6 | 17 a 20 |
| **Taxa de Chegada (navios/dia)** | 3.0 | 2.0 | 1.0 |
| **Prob. de Comprar Combustível** | 70% | 50% | 60% |


**Tempos de Processo (Distribuição Triangular: min, moda, máx em horas):**




| Processo | Tipo 1 | Tipo 2 | Tipo 3 |
| --- | --- | --- | --- |
| **Atracação** | (0.8, 1.0, 1.5) | (0.7, 0.9, 1.3) | (0.6, 0.8, 1.2) |
| **Operação** | (12.0, 18.0, 24.0) | (12.0, 15.0, 18.0) | (10.0, 12.0, 14.0) |
| **Desatracação** | (0.8, 1.0, 1.5) | (0.7, 0.9, 1.3) | (0.6, 0.8, 1.2) |


**Demanda de Combustível (Distribuição Triangular: min, moda, máx em toneladas):**




| Parâmetro | Tipo 1 | Tipo 2 | Tipo 3 |
| --- | --- | --- | --- |
| **Quantidade (t)** | (500, 1000, 2000) | (400, 800, 1600) | (300, 600, 1200) |
| **Observação** | Quantidade arredondada para o múltiplo de 100t mais próximo. |  |  |


### 2. Modelo

#### Breve explicação da lógica de modelagem

O modelo foi implementado em Python utilizando a biblioteca **SimPy**, que é orientada a processos para simulação de eventos discretos. A lógica principal é a seguinte:


1. **Estrutura do Porto:** A classe `Porto` gerencia o conjunto de terminais. Foi utilizado um `simpy.FilterStore` para representar os berços de atracação. Essa estrutura permite que cada navio solicite um terminal apenas do seu conjunto de terminais permitidos, modelando a restrição de infraestrutura de forma eficiente. Quando um navio solicita um terminal, ele espera até que um dos berços permitidos para seu tipo fique vago.


2. **Geração de Navios:** Para cada tipo de navio, um processo gerador (`_gerador_chegadas`) é iniciado. Ele cria novas chegadas de navios seguindo um **Processo de Poisson**, o que significa que o tempo entre chegadas consecutivas segue uma distribuição Exponencial. A taxa de chegada de cada tipo é um parâmetro do modelo.


3. **Ciclo de Vida do Navio:** Cada navio é uma instância de um processo (`_processo_navio`) que simula seu ciclo completo no porto:


	* **Notificação:** Ocorre 48 horas antes da chegada programada.
	* **Chegada:** O navio chega ao porto e imediatamente solicita um terminal compatível. Se todos os terminais permitidos estiverem ocupados, ele entra em uma fila de espera gerenciada pelo `FilterStore`.
	* **Atracação:** Assim que um terminal é alocado, inicia-se a manobra de atracação, que consome um tempo amostrado de uma distribuição triangular.
	* **Operação:** Após atracar, o navio realiza sua operação principal (carga/descarga), cujo tempo também é amostrado de uma distribuição triangular específica para seu tipo.
	* **Desatracação:** Concluída a operação, a manobra de desatracação se inicia, com duração amostrada de uma distribuição triangular.
	* **Liberação:** Ao final da desatracação, o navio libera o terminal, que se torna disponível para o próximo da fila.
4. **Demanda de Combustível:** Ao final do seu processo, é feita uma amostragem para decidir se o navio irá comprar combustível, com base na sua probabilidade de compra. Se for um comprador, a quantidade é amostrada de uma distribuição triangular e arredondada para o múltiplo de 100 toneladas mais próximo.


5. **Coleta de Dados:** Cada navio que completa seu ciclo registra um `LogNavio` com todos os timestamps relevantes e os dados da demanda. Ao final da simulação, esses logs são compilados em um `pandas.DataFrame`, que constitui a saída principal do modelo.


### Código do modelo (SimPy)


#### a. Imports e configuração


```python
# Modelo de simulação das operações do Porto Mar Azul usando SimPy (simulação de eventos discretos).
# Etapas do fluxo de cada navio: notificação de chegada, chegada efetiva ao porto, atracação (início da manobra), operação de carregamento/descarga e serviços no berço,
# desatracação (término da manobra de saída) e, por fim, eventuais operações de compra de combustível.
# Comentário em branco utilizado apenas para separar visualmente os blocos explicativos acima e abaixo.
# Saída esperada da simulação: um pandas.DataFrame em que cada linha representa um navio atendido, com as seguintes colunas:
# id_navio, tipo_navio, terminal, horários de notificação e de chegada ao porto, início da manobra de atracação,
# início e término do serviço no berço, fim da manobra de desatracação, indicação se houve compra de combustível (s/n) e quantidade comprada (em toneladas).

from dataclasses import dataclass, field
from typing import Dict, List, Tuple, Optional
import random
import numpy as np
import pandas as pd
import simpy
```


#### b. Utilidades de amostragem


```python
def tri_sample(low: float, mode: float, high: float, rng: random.Random) -> float:
    return rng.triangular(low, high, mode)


def tri_sample_int_100(low: float, mode: float, high: float, rng: random.Random, step: int = 100) -> int:
    v = tri_sample(low, mode, high, rng)
    q = int(round(v / step) * step)
    if q < low:
        q = int(low)
    if q > high:
        q = int(high)
    # Ajusta o valor calculado para o múltiplo inteiro mais próximo de `step`, garantindo compatibilidade com a discretização utilizada.
    q = (q // step) * step
    return q
```


#### c. Estruturas de dados


```python
@dataclass
class TipoNavioConfig:
    nome: str
    terminais: List[int]
    prob_compra: float
    taxa_chegada_por_dia: float
    t_atracacao_h: Tuple[float, float, float]     # (min, moda, max)
    t_operacao_h: Tuple[float, float, float]      # (min, moda, max)
    t_desatracacao_h: Tuple[float, float, float]  # (min, moda, max)
    demanda_t: Tuple[float, float, float]         # (min, moda, max) em toneladas


@dataclass
class ConfigSimulacao:
    tipos: Dict[str, TipoNavioConfig]
    antecedencia_notificacao_h: float = 48.0
    horizonte_dias: float = 7.0
    seed: int = 42


@dataclass
class LogNavio:
    id_navio: str
    tipo_navio: str
    terminal: Optional[int]
    notificacao: float
    chegada: float
    ini_manobra: float
    ini_servico: float
    fim_servico: float
    fim_manobra: float
    comprador: str
    quantidade_t: int
```


#### d. Ambiente do Porto


```python
class Porto:
    """Representa o conjunto de terminais como um único FilterStore com IDs discretos.
    Cada terminal (ID) tem capacidade 1 por construção (um item no Store).
    O navio só retira IDs que estão na sua lista de terminais permitidos."""

    def __init__(self, env: "simpy.Environment", terminais_ids: List[int]):
        self.env = env
        self.term_ids = sorted(list(terminais_ids))
        self.store = simpy.FilterStore(env, capacity=len(self.term_ids))
        # Inicializa o `FilterStore` com todos os terminais disponíveis no tempo 0, representando que não há navios atracados no início da simulação.
        for tid in self.term_ids:
            self.store.put(tid)

    def solicitar_terminal(self, permitidos: List[int]):
        """Retorna um evento de aquisição de terminal permitido. O item retornado é o ID do terminal."""
        return self.store.get(lambda t: t in permitidos)

    def liberar_terminal(self, term_id: int):
        """Devolve o terminal ao estoque."""
        return self.store.put(term_id)
```


#### e. Simulação


```python
class SimuladorPorto:
    def __init__(self, config: ConfigSimulacao):
        self.config = config
        self.rng = random.Random(self.config.seed)
        self.nprng = np.random.default_rng(self.config.seed)  # para exponencial
        self.logs: List[LogNavio] = []
        self._ship_seq = 0  # contador global de navios

    def _novo_id(self, tipo: str) -> str:
        self._ship_seq += 1
        return f"{tipo.upper()}-{self._ship_seq:05d}"

    def _processo_navio(self, env: "simpy.Environment", porto: Porto, tipo_cfg: TipoNavioConfig, chegada_t: float):
        """Processo completo do navio: da chegada até a liberação do terminal."""
        # Logo no início do processo de cada navio, gera-se um identificador único e sorteiam-se os parâmetros relacionados à eventual compra de combustível, para facilitar o registro completo do evento.
        navio_id = self._novo_id(tipo_cfg.nome)

        # Momento de notificação: considerado como 48 horas antes da chegada prevista; se a chegada for em menos de 48 horas, o tempo de notificação pode assumir valor negativo em relação ao início da simulação.
        t_notif = chegada_t - self.config.antecedencia_notificacao_h

        # Processo de espera até o instante de chegada do navio ao porto, a partir da notificação.
        yield env.timeout(max(0.0, chegada_t - env.now))
        t_chegada = env.now  # chegada efetiva

        # Ao chegar, o navio entra na fila e aguarda até que um dos terminais compatíveis com o seu tipo fique disponível.
        req = porto.solicitar_terminal(tipo_cfg.terminais)
        yield req  # espera até pegar um dos terminais permitidos
        term_id = req.value  # ID do terminal alocado
        t_ini_manobra = env.now  # início da atracação (manobra)

        # Início da manobra de atracação: tempo em que o navio se aproxima e é posicionado no berço escolhido.
        t_atr = tri_sample(*tipo_cfg.t_atracacao_h, self.rng)
        yield env.timeout(t_atr)
        t_ini_servico = env.now

        # Período de operação no berço, durante o qual acontecem as atividades principais (carregamento/descarga) previstas para o navio.
        t_op = tri_sample(*tipo_cfg.t_operacao_h, self.rng)
        yield env.timeout(t_op)
        t_fim_servico = env.now

        # Manobra de desatracação: tempo necessário para liberar o navio do berço e concluir sua saída do terminal.
        t_des = tri_sample(*tipo_cfg.t_desatracacao_h, self.rng)
        yield env.timeout(t_des)
        t_fim_manobra = env.now

        # Se o navio for sorteado como comprador, aqui é registrada a operação de abastecimento, com a quantidade de combustível adquirida.
        compra_flag = self.rng.random() < tipo_cfg.prob_compra
        if compra_flag:
            q = tri_sample_int_100(*tipo_cfg.demanda_t, self.rng, step=100)
            comprador = "s"
            quantidade = q
        else:
            comprador = "n"
            quantidade = 0

        # Ao final do atendimento, todos os tempos relevantes e informações do navio são armazenados em um objeto de log para posterior análise.
        self.logs.append(
            LogNavio(
                id_navio=navio_id,
                tipo_navio=tipo_cfg.nome,
                terminal=term_id,
                notificacao=t_notif,
                chegada=t_chegada,
                ini_manobra=t_ini_manobra,
                ini_servico=t_ini_servico,
                fim_servico=t_fim_servico,
                fim_manobra=t_fim_manobra,
                comprador=comprador,
                quantidade_t=quantidade,
            )
        )

        # Após o término da operação e da desatracação, o terminal é devolvido ao `FilterStore`, tornando-se disponível para atender outro navio.
        yield porto.liberar_terminal(term_id)

    def _gerador_chegadas(self, env: "simpy.Environment", porto: Porto, tipo_cfg: TipoNavioConfig, horizonte_h: float):
        """Gera chegadas segundo um processo de Poisson (intervalos ~ Exponencial) com taxa por hora.
        CORRIGIDO: agora é um *generator* do SimPy (usa 'yield env.timeout(...)')."""
        taxa_por_hora = tipo_cfg.taxa_chegada_por_dia / 24.0
        if taxa_por_hora <= 0:
            # Mesmo sendo implementada como uma generator function do SimPy, nesta função específica criamos o processo apenas para montar o cenário inicial e o encerramos em seguida.
            return

        while env.now < horizonte_h:
            inter = self.nprng.exponential(1.0 / taxa_por_hora)
            yield env.timeout(inter)  # espera até a próxima chegada
            chegada_t = env.now
            if chegada_t > horizonte_h:
                break
            # Para cada chegada agendada, cria-se um processo no ambiente do SimPy que representa o ciclo completo de vida daquele navio dentro do porto.
            env.process(self._processo_navio(env, porto, tipo_cfg, chegada_t=chegada_t))

    def rodar(self, retornar_dataframe: bool = True) -> Optional[pd.DataFrame]:
        """Executa a simulação até que todos os navios gerados completem suas operações."""
        env = simpy.Environment()
        # Constrói o conjunto de terminais do porto como a união de todos os terminais permitidos para cada tipo de navio definido na configuração.
        todos_terminais = sorted({tid for cfg in self.config.tipos.values() for tid in cfg.terminais})
        porto = Porto(env, todos_terminais)

        horizonte_h = self.config.horizonte_dias * 24.0

        # Para cada tipo de navio configurado, inicia-se um processo gerador responsável por criar sucessivas chegadas desse tipo ao longo do horizonte de simulação.
        for _, tipo_cfg in self.config.tipos.items():
            env.process(self._gerador_chegadas(env, porto, tipo_cfg, horizonte_h))

        # Executa o ambiente de simulação até que não haja mais eventos agendados, ou seja, até que todos os navios tenham concluído suas operações.
        env.run()

        if not retornar_dataframe:
            return None

        # Ao final da simulação, agrega todos os registros de log em um pandas.DataFrame ordenado pelo tempo de chegada dos navios.
        df = pd.DataFrame([{
            "id_navio": log.id_navio,
            "tipo_navio": log.tipo_navio,
            "terminal": log.terminal,
            "notificação": log.notificacao,
            "chegada": log.chegada,
            "ini_manobra": log.ini_manobra,
            "ini_serviço": log.ini_servico,
            "fim_serviço": log.fim_servico,
            "fim_manobra": log.fim_manobra,
            "comprador (s/n)": log.comprador,
            "quantidade (t)": log.quantidade_t,
        } for log in self.logs]).sort_values(by="chegada", kind="stable").reset_index(drop=True)

        return df
```


#### f. Configuração de exemplo (teste)


```python
def obter_config_mar_azul(seed, horizonte_dias) -> ConfigSimulacao:
    """Configuração baseada em dados de teste (Modelo 1: Porto Mar Azul)."""
    tipos = {
        "tipo1": TipoNavioConfig(
            nome="tipo1",
            terminais=list(range(7, 17)),         # 7–16
            prob_compra=0.70,
            taxa_chegada_por_dia=3.0,
            t_atracacao_h=(0.8, 1.0, 1.5),
            t_operacao_h=(12.0, 18.0, 24.0),
            t_desatracacao_h=(0.8, 1.0, 1.5),
            demanda_t=(500.0, 1000.0, 2000.0),    # múltiplos de 100 t
        ),
        "tipo2": TipoNavioConfig(
            nome="tipo2",
            terminais=list(range(1, 7)),          # 1–6
            prob_compra=0.50,
            taxa_chegada_por_dia=2.0,
            t_atracacao_h=(0.7, 0.9, 1.3),
            t_operacao_h=(12.0, 15.0, 18.0),
            t_desatracacao_h=(0.7, 0.9, 1.3),
            demanda_t=(400.0, 800.0, 1600.0),
        ),
        "tipo3": TipoNavioConfig(
            nome="tipo3",
            terminais=list(range(17, 21)),        # 17–20
            prob_compra=0.60,
            taxa_chegada_por_dia=1.0,
            t_atracacao_h=(0.6, 0.8, 1.2),
            t_operacao_h=(10.0, 12.0, 14.0),
            t_desatracacao_h=(0.6, 0.8, 1.2),
            demanda_t=(300.0, 600.0, 1200.0),
        ),
    }
    return ConfigSimulacao(
        tipos=tipos,
        antecedencia_notificacao_h=48.0,
        horizonte_dias=horizonte_dias,
        seed=seed,
    )
```


```python
def obter_config_mar_azul(seed, horizonte_dias) -> ConfigSimulacao:
    """Configuração baseada em dados de teste (Modelo 1: Porto Mar Azul)."""
    tipos = {
        "tipo1": TipoNavioConfig(
            nome="tipo1",
            terminais=list(range(7, 17)),         # 7–16
            prob_compra=0.70,
            taxa_chegada_por_dia=3.0,
            t_atracacao_h=(0.8, 1.0, 1.5),
            t_operacao_h=(12.0, 18.0, 24.0),
            t_desatracacao_h=(0.8, 1.0, 1.5),
            demanda_t=(500.0, 1000.0, 2000.0),    # múltiplos de 100 t
        ),
        "tipo2": TipoNavioConfig(
            nome="tipo2",
            terminais=list(range(1, 7)),          # 1–6
            prob_compra=0.50,
            taxa_chegada_por_dia=2.0,
            t_atracacao_h=(0.7, 0.9, 1.3),
            t_operacao_h=(12.0, 15.0, 18.0),
            t_desatracacao_h=(0.7, 0.9, 1.3),
            demanda_t=(400.0, 800.0, 1600.0),
        ),
        "tipo3": TipoNavioConfig(
            nome="tipo3",
            terminais=list(range(17, 21)),        # 17–20
            prob_compra=0.60,
            taxa_chegada_por_dia=1.0,
            t_atracacao_h=(0.6, 0.8, 1.2),
            t_operacao_h=(10.0, 12.0, 14.0),
            t_desatracacao_h=(0.6, 0.8, 1.2),
            demanda_t=(300.0, 600.0, 1200.0),
        ),
    }
    return ConfigSimulacao(
        tipos=tipos,
        antecedencia_notificacao_h=48.0,
        horizonte_dias=horizonte_dias,
        seed=seed,
    )
```


#### f. Função de uso externo


### 3. Testes

#### Verificação: plano e análise de experimentos para conferir se o modelo foi implementado corretamente.

A verificação do modelo focou em garantir que o código se comporta conforme a lógica especificada, antes de analisar os resultados finais. Os seguintes testes foram planejados e executados:


1. **Teste de Reprodutibilidade:**


	* **Plano:** Executar a simulação múltiplas vezes com a mesma semente (`seed`) e os mesmos parâmetros. Em seguida, alterar a semente e executar novamente.
	* **Análise:** Confirmou-se que, com a mesma semente, o DataFrame de saída é idêntico em todas as execuções. Com sementes diferentes, os resultados são diferentes. Isso valida que o comportamento estocástico do modelo é controlável e reprodutível.
2. **Teste de Lógica Temporal:**


	* **Plano:** Executar uma simulação curta e inspecionar a consistência dos timestamps na tabela de saída para algumas linhas.
	* **Análise:** Para cada navio no log, foi confirmado que a ordem cronológica dos eventos está correta: `notificação < chegada <= ini_manobra < ini_serviço < fim_serviço < fim_manobra`. O tempo de espera (`ini_manobra - chegada`) é sempre maior ou igual a zero, sendo positivo quando há congestionamento, o que confirma a lógica de filas.
3. **Teste de Alocação de Recursos:**


	* **Plano:** Filtrar o DataFrame de saída por `tipo_navio` e verificar a coluna `terminal`.
	* **Análise:** Constatou-se que, para cada tipo de navio, o terminal alocado sempre pertence ao seu conjunto de terminais permitidos (e.g., navios "tipo2" somente são alocados a terminais de 1 a 6). Isso confirma que a lógica de restrição de recursos (`FilterStore`) foi implementada corretamente.


### 4. Resultados

#### Validação: plano e análise de experimentos para avaliar se o modelo representa adequadamente a operação real.

A validação formal exigiria a comparação dos resultados com dados históricos do porto. Na ausência desses dados, realizou-se uma **validação de face**, avaliando se os resultados gerados pela simulação (arquivo `logs_simulacao_porto.csv`) são razoáveis e consistentes com os parâmetros de entrada.


**Análise dos Resultados da Simulação (Horizonte de 365 dias):**


* **Volume de Operações:**


	+ **Total de navios processados:** 2187
	+ **Navios por tipo:**
		- Tipo 1: 1095
		- Tipo 2: 728
		- Tipo 3: 364
	+ *Análise:* O número de navios gerado por tipo é extremamente próximo do esperado teoricamente (Taxa de Chegada × 365 dias), validando o processo de geração de chegadas.
* **Desempenho Temporal (em horas):**


	+ **Tempo médio de espera por terminal:** 2.89 horas
	+ **Tempo médio de permanência no porto:** 22.87 horas
	+ *Análise:* A existência de um tempo de espera médio positivo (2.89 h) demonstra que o modelo está capturando corretamente a competição por recursos (terminais) e a formação de filas.
* **Demanda de Combustível:**


	+ **Navios compradores:** 1272 (58.16% do total)
	+ **Volume total demandado:** 1,228,100 toneladas
	+ **Demanda média por comprador:** 965.49 toneladas
	+ *Análise:* A proporção de compradores (58.16%) é condizente com as probabilidades de compra de cada tipo de navio. O volume de demanda gerado é robusto e serve como uma base de dados coerente para a próxima etapa do projeto.


**Conclusão da Validação:** Os resultados são consistentes com os parâmetros de entrada e a lógica do sistema, indicando que o modelo representa a operação de forma adequada para o objetivo proposto.