# tutorial-terminal-d3js-ibge
Tutorial de como fazer mapas apenas com a linha de comando em D3js com os dados do IBGE

## Convertendo a malha censitária em um mapa
Pegue os dados relativos a malha censitária de São Paulo. No FTP do IBGE é possível encontrar de todas as UFs em _ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/_.  
  
Como não queremos sair do terminal, vamos usar o __curl__ para isso. E vamos apontar para a malha de Minas Gerais 

```bash
curl \
  'ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/mg/mg_setores_censitarios.zip' \
  -o mg_setores_censitarios.zip
```

Depois é dezipar a pasta

```bash
unzip -o mg_setores_censitarios.zip
```

Vamos instalar o shapefile do nosso querido amigo Mike Bostock. Ele precisa de Node e do NPM, então se você estiver usando um Mac eu recomendo usar o Homebrew. Entretanto, você pode confiar no com o instalador da página deles mesmo.

```terminal
npm install -g shapefile
```

Vamos convertero SHP para GeoJSON

```terminal
shp2json 31SEE250GC_SIR.shp --encoding 'utf8' -o mg.json
```

Vamos instalar as projeções do D3

```terminal
npm install -g d3-geo-projection
```

E aplicar a projeção ortográfica ao estado do Rio de Janeiro

```terminal
geoproject \
  'd3.geoOrthographic().rotate([42.5, 22.5, 0]).fitSize([1000, 600], d)' \
  < mg.json \
  > mg-ortho.json
```

Para finalizar, vamos converter a projeção em SVG

```terminal
geo2svg \
  -w 1000 \
  -h 600 \
  < mg-ortho.json \
  > mg-ortho.svg
```

## Unindo os dados censitários a malha

```terminal
npm install -g ndjson-cli
```

E vamos separar em um ndjson o mapa projetado

```terminal
ndjson-split 'd.features' \
  < mg-ortho.json \
  > mg-ortho.ndjson
```

Depois vamos mapear os códigos dos setores para bater com a coluna do CSV

```terminal
ndjson-map 'd.Cod_setor = d.properties.CD_GEOCODI, d' \
  < mg-ortho.ndjson \
  > mg-ortho-sector.ndjson
```

Vamos baixar todos os dados da amostra geral do Rio de Janeiro

```terminal
curl \
  'ftp://ftp.ibge.gov.br/Censos/Censo_Demografico_2010/Resultados_do_Universo/Agregados_por_Setores_Censitarios/MG_20171016.zip' \
  -o MG_20171016.zip
```

Dezipar os dados

```terminal
unzip -o MG_20171016.zip
```

Depois vamos instalar o módulo de conversão de CSVs, TSVs e afins

```terminal
npm install -g d3-dsv
```

Converteremos o CSV do censo para ndjson

```terminal
dsv2json \
  -r ';' \
  -n \
  < MG/Base\ informa\%E7oes\ setores2010\ universo\ MG/CSV/Domicilio01_MG.csv \
  > mg-census.ndjson
```

E uniremos o arquivo de dados do censo com a projeção ortográfica

```terminal
ndjson-join 'd.Cod_setor' \
  mg-ortho-sector.ndjson \
  mg-census.ndjson \
  > mg-ortho-census.ndjson
```

E descobriremos a % da população que se considera branca

```terminal
ndjson-map \
  'd[0].properties = {rent: Math.floor(100 * Number(d[1].V008) / d[1].V002)}, d[0]' \
  < mg-ortho-census.ndjson \
  > mg-ortho-rent.ndjson
```

## Colorindo o mapa

instalar o D3

```terminal
npm install -g d3
```

Então vamos colorir de maneira aleatória o mapa

```terminal
ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 100])(d.properties.rent), d)' \
  < mg-ortho-rent.ndjson \
  > mg-ortho-color.ndjson
```

E vamos printá-lo num SVG

```terminal
geo2svg -n --stroke none -w 1000 -h 600 \
  < mg-ortho-color.ndjson \
  > mg-ortho-color.svg
```

Instalar o TopoJSON

```terminal
npm install -g topojson
```

Unificando os elementos em um topo

```terminal
geo2topo -n \
  tracts=mg-ortho-rent.ndjson \
  > mg-tracts-topo.json
```

Simplificando a geometria

```terminal
toposimplify -p 1 -f \
  < mg-tracts-topo.json \
  > mg-simple-topo.json
```

Quantificando a geometria

```terminal
topoquantize 1e5 \
  < mg-simple-topo.json \
  > mg-quantized-topo.json
```

Instalar a escala cromática do D3

```terminal
npm install -g d3-scale-chromatic
```

E gera o JSON com os recortes

```terminal
topo2geo tracts=- \
  < mg-quantized-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]).range(d3.schemeYlOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.rent)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -w 1000 -h 600 \
  > mg-tracts-threshold-rent.svg
  ```
