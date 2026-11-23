# Q-Billings: Monitoramento Espacial de Mananciais e Otimização Quântica de Patrulhamento Ambiental em São Bernardo do Campo

## Visão Geral do Projeto

O **Q-Billings** é um sistema de apoio ao monitoramento e patrulhamento ambiental desenvolvido para a região do pós-balsa e entorno da Represa Billings, no município de São Bernardo do Campo (SP). O projeto estrutura um pipeline ponta a ponta, reprodutível e auditável para apoiar equipes de fiscalização e guarda ambiental na identificação de focos de degradação e no planejamento de rotas terrestres:

1. **Sensoriamento Remoto e Segmentação Semântica:** Aquisição on-the-fly de imagens multiespectrais (Sentinel-2 L2A via catálogo STAC) e referência oficial/contingência, seguida de segmentação pixel a pixel com U-Net (ResNet-18) e filtragem hidrológica via NDWI.
2. **Geoprocessamento e Malha Viária Real (GIS):** Vetorização dos focos em coordenadas projetadas métricas **SIRGAS 2000 / UTM zona 23S (`EPSG:31983`)**, extração de centroides e modelagem de grafo viário real com OpenStreetMap (`osmnx` + `networkx`), registrando explicitamente lacunas sem via mapeada por meio de penalidade (`FATOR_FALLBACK`).
3. **Otimização Combinatória e Comparação de Rotas:** Formulação QUBO/Ising e execução do QAOA (Qiskit) sobre a matriz de custos em quilômetros, comparando a **Rota Q-Billings** (`51,673 km`) com estratégias clássicas e operacionais de campo.

---

## Estrutura do Repositório

```text
qbillings/
├── documentacao/       # Proposta de projeto final, especificações técnicas e relatórios acadêmicos
├── mascaras/           # Arquivos de máscaras binárias (rasters GeoTIFF), amostras e saídas do modelo U-Net
├── notebook/           # Jupyter Notebooks completos com o pipeline executável ponta a ponta
└── README.md           # Documentação principal do repositório
```

---

## Detalhamento Técnico do Pipeline

### Fase 1: Sensoriamento Remoto e Segmentação Semântica (Deep Learning)
- **Aquisição On-the-Fly via STAC e Bases Oficiais:** Consulta programática de cenas multiespectrais Sentinel-2 L2A (bandas Red, Green, Blue e NIR) por caixa delimitadora (`BBOX_4326`) via catálogo STAC (`pystac-client` / `planetary-computer`), combinada à consulta WFS de bases oficiais (como INPE/PRODES e MapBiomas) e fallback local de contingência, reprojetando a grade para `4x1024x1024` em `EPSG:31983` (`10 m` de resolução espacial).
- **Normalização Radiométrica e Fatiamento em Patches:** Higienização de valores inválidos, normalização Min-Max por banda (`[0.0, 1.0]`) e recorte em **16 patches** não sobrepostos de `256x256` pixels com rastreio espacial `(i, j)`.
- **Arquitetura U-Net + ResNet-18:** Treinamento e inferência de rede convolucional *U-Net* (`segmentation-models-pytorch`) com codificador *ResNet-18* adaptado para 4 canais espectrais (`in_channels=4`, `classes=1`), perda combinada (`DiceLoss` + `BCEWithLogitsLoss`), otimizador `AdamW` e limiar de sensibilidade (`LIMIAR_DEGRADACAO = 0.35`).
- **Exclusão Hídrica (NDWI) e Exportação GeoTIFF:** Remoção de falsos positivos sobre corpos d'água via índice NDWI (`(Green - NIR) / (Green + NIR) > 0.05`), limpeza morfológica (`binary_opening` e `binary_closing`) e exportação da máscara georreferenciada `mascara_desmatamento_billings.tif` em `uint8` (`EPSG:31983`).

### Fase 2: Geoprocessamento, Vetorização e Grafo Viário Real (GIS)
- **Vetorização e Métricas em `EPSG:31983`:** Extração de polígonos conectados com `rasterio.features.shapes`, `shapely` e `geopandas` em **SIRGAS 2000 / UTM zona 23S (`EPSG:31983`)**, calculando área (`area_m2` e `area_ha`) e centroides geométricos para compor a base operacional e os focos do circuito (`130` pontos totais na cena completa e recorte principal `N0` + `N1` a `N5` para roteamento).
- **Modelagem da Malha Viária Real (`OSMnx` + `NetworkX`):** Extração da rede viária e de trilhas do OpenStreetMap na área do pós-balsa (com margem externa de `1.500 m`) e construção de grafo direcionado e ponderado `G = (V, E)` em quilômetros (`1.243` nós e `3.049` arestas).
- **Transparência em Áreas Sem Via Mapeada:** Cada ponto é conectado ao nó viário OSM mais próximo (`LIMIAR_VIA_M = 250.0 m`). Quando não há via cadastrada dentro do limiar, o grafo cria uma aresta explícita de fallback (`no_road_fallback`) com custo penalizado (`FATOR_FALLBACK = 3.5`), sem criar pontes artificiais.
- **Matriz de Custos Logísticos (`130x130`):** Cálculo dos caminhos mínimos ponderados entre todos os pares de nós operacionais, gerando a matriz quadrada `matriz_distancias_km` validada numericamente.

### Fase 3: Otimização Quântica de Rotas (QUBO / QAOA) e Comparativo Operacional
- **Formulação Combinatória e Mapeamento QUBO/Ising:** Modelagem do circuito fechado de patrulhamento partindo da Base da Guarda Ambiental (`N0`), visitando os focos prioritários (`N1` a `N5`) e retornando à base, convertido em problema QUBO e Hamiltoniano de Ising.
- **Execução Variacional com QAOA (Qiskit):** Otimização variacional com `QAOAAnsatz`, `StatevectorEstimator` e `StatevectorSampler` (`COBYLA`), combinada à avaliação de permutações/heurísticas sobre `matriz_distancias_km` e validação frente à busca exata clássica.
- **Comparativo de Percursos Avaliados:**
  - **Rota Q-Billings (`51,673 km`):** Sequência consolidada `N0 -> N4 -> N5 -> N2 -> N3 -> N1 -> N0`, menor extensão entre os cenários avaliados.
  - **Rota Dijkstra (`51,673 km`):** Encadeamento guloso por menor caminho de Dijkstra a partir de `N0` (`N0 -> N2 -> N1 -> N3 -> N5 -> N4 -> N0`), empatando em distância total.
  - **Estrada / Eixo Viário OSM (`51,673 km`):** Priorização de aderência aos eixos viários cadastrados no OpenStreetMap, somando `51,673 km`.
  - **Ambiental — Maior Área (`52,084 km`):** Ordenação decrescente por tamanho de mancha (`113,0 ha` em `N1` até `30,1 ha` em `N5`), com acréscimo de `+0,411 km`.
  - **Defesa Ambiental (`66,415 km`):** Circuito estendido de fiscalização no Pós-Balsa (Taquacetuba, Tatetos, Santa Cruz, Capivari e retorno), totalizando `+14,742 km` frente à Rota Q-Billings.

---

## Tecnologias Utilizadas

- **Linguagem Principal:** Python 3.10+
- **Sensoriamento Remoto & Catálogo STAC:** `pystac-client`, `planetary-computer`, `rasterio`, `requests`
- **Deep Learning & Visão Computacional:** `torch`, `torchvision`, `segmentation-models-pytorch`, `numpy`, `scipy`
- **Geoprocessamento & Grafos Viários:** `geopandas`, `shapely`, `pyproj`, `osmnx`, `networkx`, `pandas`
- **Computação Quântica & Otimização:** `qiskit`, `qiskit-algorithms`, `qiskit-optimization`, `qiskit-aer`
- **Visualização & Mapeamento:** `matplotlib`, `folium`

---

## Como Executar o Projeto

### 1. Clonar o Repositório

```bash
git clone [https://github.com/joaohppenha/qbillings.git](https://github.com/joaohppenha/qbillings.git)
cd qbillings
```

### 2. Instalar as Dependências

Em um ambiente virtual Python ou na primeira célula do Google Colab:

```bash
pip install rasterio segmentation-models-pytorch torch torchvision matplotlib numpy pystac-client planetary-computer geopandas shapely requests osmnx networkx pandas pyproj folium qiskit qiskit-algorithms qiskit-optimization qiskit-aer scipy
```

### 3. Execução do Pipeline no Notebook

1. Acesse o diretório `notebook/`.
2. Abra o arquivo `.ipynb` no Jupyter Notebook ou carregue-o diretamente no **Google Colab**.
3. Execute as células em ordem sequencial. O pipeline realizará a aquisição da cena multiespectral via STAC, segmentação com U-Net (ResNet-18) e filtro NDWI, vetorização em `EPSG:31983`, extração da malha viária OSM e cálculo da matriz de custos (`130x130`), otimização via QAOA e geração dos mapas comparativos de rotas.

---

## Próximos Passos do Projeto Q-Billings

Para os próximos passos do **Projeto Q-Billings** na região do pós-balsa da Represa Billings (São Bernardo do Campo, `EPSG:31983`), o foco será:

- **Consolidação de fontes oficiais e cache local:** Ajustar e estabilizar a consulta programática a camadas oficiais (como MapBiomas, INPE/PRODES e bases territoriais da APRM-Billings/SEMIL/DataGeo/GeoSampa), mantendo cache e fallback local para evitar falhas de rede ou respostas `400` durante execuções e demonstrações.
- **Complementação da malha viária e detecção de acessos:** Contribuir com o mapeamento colaborativo de vias, estradas vicinais e trilhas no OpenStreetMap (`OSM`) na região do pós-balsa e avaliar modelos de detecção de vias a partir de sensoriamento remoto para reduzir trechos de fallback (`FATOR_FALLBACK`) em áreas de margem e manancial.
- **Ampliação da otimização clássica e quântica:** Comparar o circuito consolidado (`Rota Q-Billings`, `51,673 km`) com solvers clássicos dedicados de roteamento (como TSP/VRP) e expandir os testes do QAOA em Qiskit (incluindo análise de profundidade, ruído ou backends simulados/reais) sobre a matriz `matriz_distancias_km`.
- **Generalização territorial e operabilidade:** Validar a mesma arquitetura em diferentes recortes geográficos e cenários operacionais (incluindo malhas mais adensadas e camadas interativas em `folium`).

---

## Impacto e Resultados Práticos

- **Precisão Geográfica e Cartográfica:** Identificação automatizada de focos com exclusão de corpos hídricos (NDWI) e padronização integral no sistema oficial brasileiro **SIRGAS 2000 / UTM 23S (`EPSG:31983`)**.
- **Roteamento Realista e Auditável:** Integração com a malha viária real do OpenStreetMap e tratamento transparente de trechos sem estrada mapeada (`FATOR_FALLBACK`), evitando rotas irreais em linha reta sobre a represa.
- **Eficiência Operacional:** Consolidação de circuito de patrulhamento com **51,673 km**, reduzindo deslocamentos desnecessários, tempo de equipe em campo, consumo de combustível e emissões operacionais.
- **Alinhamento aos ODS:** Contribuição direta às metas do **ODS 6** (Água Potável e Saneamento), **ODS 11** (Cidades e Comunidades Sustentáveis) e **ODS 15** (Vida Terrestre) da Agenda 2030 da ONU.
