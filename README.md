# Q-Billings: Monitoramento Espacial de Mananciais e Otimização Quântica de Patrulhamento Ambiental em São Bernardo do Campo

## Visão Geral do Projeto

O **Q-Billings** é uma plataforma inteligente e automatizada de monitoramento ambiental e gestão logística desenvolvida para a proteção da Área de Preservação Permanente (APP) da Represa Billings, no município de São Bernardo do Campo (SBC). O sistema resolve o gargalo de resposta rápida a infrações ambientais (desmatamento ilegal, ocupação de margens e supressão de Mata Atlântica) por meio de um pipeline ponta a ponta integrando três pilares tecnológicos:

1. **Sensoriamento Remoto & Inteligência Artificial:** Detecção automática de degradação vegetal a partir de imagens orbitais.
2. **Geoprocessamento Espacial (GIS):** Vetorização de áreas críticas, mapeamento em coordenadas reais e construção de grafos de deslocamento.
3. **Computação Quântica Variacional:** Roteamento otimizado de patrulhamento terrestre e náutico com foco na minimização de distância, redução de custos e menor pegada de carbono.

---

## Estrutura do Repositório

```text
qbillings/
├── documentacao/       # Proposta de projeto final, especificações técnicas e relatórios acadêmicos
├── mascaras/           # Arquivos de máscaras binárias (rasters), amostras e saídas do modelo U-Net
├── notebook/           # Jupyter Notebooks completos com o pipeline executável ponta a ponta
└── README.md           # Documentação principal do repositório

```
---

## Detalhamento Técnico do Pipeline

### Fase 1: Sensoriamento Remoto & Segmentação Semântica (Deep Learning)
- **Ingestão de Dados Multiespectrais:** Importação e pré-processamento de imagens orbitais do satélite Sentinel-2 sobre o pós-balsa e margens do reservatório em SBC, divididas em patches de 256x256 pixels.
- **Arquitetura U-Net + ResNet-18:** Implementação de uma rede neural convolucional profunda (*U-Net*) acoplada ao extrator de características pré-treinado (*backbone ResNet-18*). A estrutura encoder-decoder preserva bordas e detalhes espaciais finos de degradação.
- **Segmentação Pixel a Pixel:** Classificação contínua do terreno para isolar alterações em faixas de preservação permanente (APP) e áreas de supressão recente de Mata Atlântica.
- **Saída Binária:** Geração de matrizes binárias (*rasters*) onde os pixels ativos (`1`) delimitam com precisão os focos de intervenção prioritária.

### Fase 2: Geoprocessamento e Estruturação do Grafo Espacial (GIS)
- **Vetorização com GeoPandas:** Conversão dos agrupamentos de pixels ativos da máscara binária em polígonos vetoriais delimitados.
- **Sistemas de Referência Espacial (CRS):** Projetado e padronizado no sistema de coordenadas oficiais **UTM SIRGAS 2000 / Fuso 23S (EPSG:31983)** para garantir precisão métrica de terreno.
- **Extração de Centroides:** Cálculo do centro geométrico ponderado dos polígonos para definir a localização exata dos focos de fiscalização no mapa:
  - **Nó 0 (N0):** Base Operacional da Guarda Ambiental de São Bernardo do Campo.
  - **Nós 1 e 2 (N1, N2):** Centroides dos focos de degradação ambiental identificados pela U-Net.
- **Grafo Topológico no NetworkX:** Construção de um grafo ponderado e direcionado G=(V, E) alimentado pela matriz de distâncias geodésicas/euclidianas em quilômetros entre todos os pontos do circuito.

### Fase 3: Otimização Combinatória Quântica (QUBO / QAOA)
- **Formulação do Problema do Caixeiro Viajante (TSP):** Modelação da rota de fiscalização restrita a visitar todos os alvos vetorizados (N1, N2) com partida e retorno garantidos à Base (N0).
- **Mapeamento QUBO:** Conversão do TSP para o formato *Quadratic Unconstrained Binary Optimization*, codificando a matriz de distâncias espaciais como termo de custo e impondo penalidades quadráticas para restrições de rota.
- **Solver Variacional QAOA:** Execução do *Quantum Approximate Optimization Algorithm* via `Qiskit Optimization` e `Qiskit Aer Simulator`. O algoritmo explora a superfície de energia do Hamiltoniano através do otimizador clássico `COBYLA` até alcançar o estado fundamental de menor custo.
- **Resultado Operacional:** Determinação da sequência ótima de patrulhamento de menor consumo logístico: **Base (N0) -> N2 -> N1 -> Base (N0)**.

---

## Tecnologias Utilizadas

- **Linguagem Principal:** Python 3.10+
- **Deep Learning & Visão Computacional:** PyTorch, Torchvision, NumPy
- **Engenharia Geoespacial & Grafos:** GeoPandas, Shapely, Rasterio, NetworkX, SciPy
- **Computação Quântica & Otimização:** Qiskit, Qiskit Optimization, Qiskit Algorithms, Qiskit Aer Simulator
- **Visualização & Mapeamento:** Folium (com plugins AntPath e Esri World Imagery), Matplotlib, Seaborn

---

## Como Executar o Projeto

### 1. Clonar o Repositório

git clone https://github.com/joaohppenha/qbillings.git
cd qbillings

### 2. Instalar as Dependências

Em um ambiente virtual Python ou célula inicial do Colab:

pip install torch torchvision geopandas rasterio networkx qiskit qiskit-algorithms qiskit-optimization qiskit-aer folium matplotlib scipy

### 3. Execução do Pipeline no Notebook

1. Acesse o diretório `notebook/`.
2. Abra o arquivo `.ipynb` no Jupyter Notebook ou carregue-o diretamente no **Google Colab**.
3. Execute as células em ordem sequencial. O pipeline fará o carregamento dos patches, inferência com U-Net, geoprocessamento em UTM, otimização quântica via QAOA e renderização do mapa interativo final com a rota de patrulhamento animada.

---

## Impacto e Resultados Práticos

- **Precisão Geográfica:** Identificação pontual e automatizada de supressões vegetais em Áreas de Preservação Permanente (APP) ao longo do manancial da Billings.
- **Eficiência Logística:** Minimização comprovada do tempo e da distância percorrida pelas equipes terrestres e náuticas da Guarda Ambiental.
- **Sustentabilidade Operacional:** Redução direta no consumo de combustível e diminuição da pegada de carbono dos veículos de fiscalização municipal.
- **Contribuição aos ODS:** Alinhamento direto às metas do **ODS 6** (Água Potável e Saneamento) e **ODS 15** (Vida Terrestre) da Agenda 2030 das Nações Unidas.
