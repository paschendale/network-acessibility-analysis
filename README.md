Este tutorial emprega a versão 13 do PostgreSQL, junto das versões mais atualizadas do PostGIS e pgRouting. Para realizar a instalação das dependências, utilize seu gerenciador de pacotes (em SOs Linux ou Mac) ou os pacotes de instalação da EnterpriseDB (em SOs Windows). Recomenda-se o uso do pgAdmin 4 (instalado junto com os pacotes da EnterpriseDB ou separadamente em Linux e Mac) para rodar os códigos a seguir na base de dados.

Na pasta `sql` há um arquivo `routing.sql`, contendo um dump completo da base de dados utilizada neste artigo, opcionalmente, você pode recuperá-la através da linha de comando do `psql`.

## Preparação da base de dados

### 1. Criar base de dados e adicionar extensões ao banco

`CREATE EXTENSION postgis; 
CREATE EXTENSION pgrouting;`

### 2. Faça upload da rede no PostGIS e configure as topologias de rede

**NOTA**: renomeie o arquivo como rede_l antes ou depois do upload. O upload pode ser feito tanto pelo QGIS arrastando para a conexão da base criada quanto pela ferramenta de upload de shapefiles para o PostGIS. Em `example` você encontrará um shapefile de redes que pode ser usado para este tutorial


` -- Criando campos para armazenar os ids dos vértices no início e no fim dos arcos
ALTER TABLE rede_l ADD source INT4;
ALTER TABLE rede_l ADD target INT4;`

` -- Alterando o nome da coluna de geometria para the_geom para compatibilizar com o padrão do pgRouting. Esta etapa é opcional caso o nome da colune de geometria no banco já seja 'the_geom'
ALTER TABLE rede_l RENAME COLUMN geom TO the_geom;`

` -- Criando a topologia da rede
SELECT pgr_createTopology('rede_l',1);`

Ao final do procedimento a query enviará uma mensagem **OK**, indicando que tudo ocorreu com sucesso. Caso a query envie uma mensagem **FAIL** procure na aba *Messages* do pgAdmin qual a razão do erro.

**NOTA**: A execução de `pgr_createTopology` cria uma segunda tabela com o template `nome_tabela_arcos_vertices_pgr`

### 3. Encontrar os nós centrais dos arcos 

No QGIS, abra a caixa de ferramentas através do atalho `CTRL` + `Alt` + `T`.
Procure pela ferramenta no menu *GDAL > Geoprocessamento de Vetor > Pontos ao longo das linhas*.

Preencha os campos com os seguintes parâmetros:

**Camada de entrada:** rede_l
**Nome da coluna de geometria:** the_geom
**Distancia a partir da linha:** 0.5

Salve a camada criada como **rede_mid_nodes_p** em disco no formato *.gpkg*, e faça upload na base de dados posteriormente.

**NOTA**: O uso do *.gpkg* para o salvamento de arquivos em disco é importante para que não se desconfigure nomes de colunas e codificação das tabelas. 

### 4. Dividir os arcos (**rede_l**) nos locais dos nós centrais (**rede_mid_nodes_p**)

**NOTA**: Para este passo, será necessária a instalação complemento do QGIS *Networks*

Salve a camada **rede_l** em disco no formato *.gpkg* como **rede_renoded_l**.
No menu de ferramentas do QGIS, procure por *Networks > Connect Nodes to Lines*

Preencha os campos com os seguintes parâmetros:

**Network**: rede_renoded_l
**Nodes**: rede_mid_nodes_p

A ferramenta irá realizar a edição do arquivo **rede_renoded_l** criando vértices no centro de cada arco e irá tentar salvar a edição no arquivo. Neste momento, o QGIS apontará um erro indicando que não foi possível realizar o salvamento do arquivo devido à um conflito de chaves primárias, isso não importará no momento. Arraste o arquivo **rede_renoded_l** ainda em edição para dentro da base de dados no QGIS.

### 5. Configurar topologias na tabela **rede_renoded_l**

`-- Recria os campos de source e target para a nova rede
ALTER TABLE rede_renoded_l DROP source;
ALTER TABLE rede_renoded_l DROP target;
ALTER TABLE rede_renoded_l ADD source INT4;
ALTER TABLE rede_renoded_l ADD target INT4;`

`-- Caso a tabela possua id's anteriores esta parte do código recria os IDs da tabela mantendo o id de rede_l como id_edge
ALTER TABLE rede_renoded_l RENAME COLUMN id TO id_edge;
ALTER TABLE rede_renoded_l DROP IF EXISTS fid; 
ALTER TABLE rede_renoded_l DROP IF EXISTS id_0; 
ALTER TABLE rede_renoded_l ADD COLUMN id SERIAL PRIMARY KEY;`

`-- Reconstruindo a topologia
ALTER TABLE rede_renoded_l RENAME COLUMN geom TO the_geom;
SELECT pgr_createTopology('rede_renoded_l',1);`

### 6. Atualizar a tabela de arcos originais (**rede_l**) com os vértices médios que os representam

`ALTER TABLE rede_l ADD vertice_medio INT4;
UPDATE rede_l SET vertice_medio = subq.id_vertice
FROM (
	SELECT b.id AS id_vertice, a.id AS id_edge FROM
	rede_mid_nodes_p a,
	rede_renoded_l_vertices_pgr b
	WHERE a.geom = b.the_geom
) subq
WHERE subq.id_edge = rede_l.id;`

### 7. Atualizar a tabela nova de vértices com os arcos que representam, caso sejam vértices médios

`ALTER TABLE rede_renoded_l_vertices_pgr ADD id_edge INT4;
UPDATE rede_renoded_l_vertices_pgr SET id_edge = rede_l.id_edge
FROM (SELECT id AS id_edge, vertice_medio FROM rede_l) rede_l
WHERE vertice_medio = id;`

## Análises

A partir deste ponto são realizadas as análises da matriz de custos de **rede_renoded_l**, as análises realizadas são recombinações e junções da tabela resultante da função `pgr_dijkstraCostMatrix` do pgRouting (ver [referência](https://docs.pgrouting.org/3.1/en/pgr_dijkstraCostMatrix.html)) com as demais tabelas do banco e podem ser feitas separadamente ou em uma query que cria uma vista materializada denominada `rede_l_analises`, que pode ser acessada via QGIS. Caso seu interesse seja somente o resultado final avance para o item `12. Criando uma camada com todos os atributos de acessibilidade para cada arco`.

### 8. Criando a matriz de proximidade entre arcos (matriz resultante da função `pgr_dijkstraCostMatrix`)

`SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
FROM 
pgr_dijkstraCostMatrix(
	'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
	(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
) matrix
LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid;`

### 9. Extraindo a matriz de ligações diretas para cada arco

`SELECT id_edge_start, SUM(distancia) ligacoes_diretas FROM
(
	SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
	FROM 
	pgr_dijkstraCostMatrix(
		'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
		(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
	) matrix
	LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
	LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
WHERE distancia <= 1
GROUP BY id_edge_start`

### 10. Extraindo a matriz de distancia até o arco mais distante da rede

`SELECT DISTINCT ON (start_vid.id_edge) start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
FROM 
pgr_dijkstraCostMatrix(
	'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
	(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
) matrix
LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
ORDER BY start_vid.id_edge, agg_cost/2 DESC`

### 11. Extraindo a matriz de conexões ponderadas para *n* ordens de um arco

a. (OPCIONAL) Obter a matriz de conexões por arco para cada ordem

`SELECT id_edge_start, distancia AS ordem, COUNT(id_edge_start) AS n_ligacoes, COUNT(id_edge_start)*distancia AS n_ligacoes_ponderadas
FROM (	SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
	FROM 
	pgr_dijkstraCostMatrix(
		'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
		(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
	) matrix
	LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
	LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
GROUP BY id_edge_start, distancia
ORDER BY id_edge_start, distancia`

b. Obter a matriz de conexões ponderadas por arco

`SELECT id_edge_start, SUM(n_ligacoes_ponderadas) AS total_ligacoes FROM
(SELECT id_edge_start, distancia AS ordem, COUNT(id_edge_start) AS n_ligacoes, COUNT(id_edge_start)*distancia AS n_ligacoes_ponderadas
FROM (	SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
	FROM 
	pgr_dijkstraCostMatrix(
		'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
		(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
	) matrix
	LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
	LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
GROUP BY id_edge_start, distancia
ORDER BY id_edge_start, distancia) subq
GROUP BY id_edge_start`

### 12. Criando uma camada com todos os atributos de acessibilidade para cada arco

`CREATE MATERIALIZED VIEW rede_l_analises AS
SELECT rede_l.*, ligacoes_diretas.ligacoes_diretas, mais_distante.distancia AS passos_mais_distante, total_ligacoes.total_ligacoes  
FROM rede_l 
LEFT JOIN (
	SELECT id_edge_start, SUM(distancia) ligacoes_diretas FROM
	(
		SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
		FROM 
		pgr_dijkstraCostMatrix(
			'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
			(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
		) matrix
		LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
		LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
	) subq
	WHERE distancia <= 1
	GROUP BY id_edge_start
) ligacoes_diretas ON ligacoes_diretas.id_edge_start = rede_l.id
LEFT JOIN (
	SELECT DISTINCT ON (start_vid.id_edge) start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
	FROM 
	pgr_dijkstraCostMatrix(
		'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
		(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
	) matrix
	LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
	LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
	ORDER BY start_vid.id_edge, agg_cost/2 DESC
) mais_distante ON mais_distante.id_edge_start = rede_l.id
LEFT JOIN (
SELECT id_edge_start, SUM(n_ligacoes_ponderadas) AS total_ligacoes FROM
(SELECT id_edge_start, distancia AS ordem, COUNT(id_edge_start) AS n_ligacoes, COUNT(id_edge_start)*distancia AS n_ligacoes_ponderadas
FROM (	SELECT start_vid.id_edge AS id_edge_start, end_vid.id_edge AS id_edge_end, agg_cost/2 AS distancia 
	FROM 
	pgr_dijkstraCostMatrix(
		'SELECT id, source::INT4, target::INT4, 1 AS cost, 1 AS reverse_cost FROM rede_renoded_l',
		(SELECT array_agg(vertice_medio) FROM rede_l WHERE vertice_medio IS NOT NULL)
	) matrix
	LEFT JOIN rede_renoded_l_vertices_pgr start_vid ON start_vid.id = start_vid
	LEFT JOIN rede_renoded_l_vertices_pgr end_vid ON end_vid.id = end_vid
) subq
GROUP BY id_edge_start, distancia
ORDER BY id_edge_start, distancia) subq
GROUP BY id_edge_start
) total_ligacoes ON total_ligacoes.id_edge_start = rede_l.id;
`

## Referências

[pgRouting Manuals](https://docs.pgrouting.org/)

## Autoria e Citações

Este tutorial é resultado de um artigo elaborado pelos estudantes de pós-graduação em Engenharia Civil MSc. Engº. Victor Marotta, Engº. Luís Santos Nascimento, Engº. Augusto Franco e o professor DSc. MSc. Engº. Marcus Vínicius Abreu. 

Para referências, favor citar: (a publicar)
