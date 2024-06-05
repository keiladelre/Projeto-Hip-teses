# PROJETO HIPÓTESES
Neste projeto trabalhei com as liguagens SQL e Python, e com as ferramentas BigQuery e Power BI, com o objetivo de validar ou refutar hipóteses, por meio da análise exploratória dos dados e cálculos estatísticos de correlação das variáveis da base de dados do Spotify de 2023.

## Objetivo:

Este projeto foi elaborado para responder a um case criado onde uma gravadora que deseja lançar um novo artista, pergunta: Quais as recomendações estratégicas para que a gravadora e o novo artista possam tomar as decisões, que aumentem suas chances de alcançar o "sucesso"?

Para responder a essa pergunta, utilizamos a base de dados do Spotify, com todas as músicas mais escutadas em 2023, e a partir desses dados a gravadora formulou algumas hipóteses, que foram consideradas no processo de análise dos dados, onde iremos confirmar ou refutar cada uma delas.

## Hipóteses:

A gravadora levantou uma série de hipóteses sobre o que faz uma música seja mais ouvida. Essas hipóteses incluem:

- Músicas com BPM (Batidas Por Minuto) mais altos fazem mais sucesso em termos de número de streams no Spotify.
- As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas, como a Deezer.
- A presença de uma música em um maior número de playlists está correlacionada com um maior número de streams.
- Artistas com um maior número de músicas no Spotify têm mais streams.
- As características da música influenciam o sucesso em termos de número de streams no Spotify.

## Base de dados utilizada:

[https://drive.google.com/file/d/11W1wfljCoRKy1Uk5R65LHWmh2mtCtMGV/view]

## 1.Pré

**1.1-** Criar o bigquery por sandbox.

[https://www.youtube.com/watch?v=z32438Yehl4&list=PL5TJqBvpXQv5n1N15kcK1m9oKJm_cv-m6&index=3](https://www.youtube.com/watch?v=z32438Yehl4&list=PL5TJqBvpXQv5n1N15kcK1m9oKJm_cv-m6&index=3)

**1.2-** Baixar as planilhas na plataforma em zip e extrair arquivos no desktop

## **2. Processar e preparar a base de dados**

### 2.1- Conectar/importar dados para as ferramentas

**2.1.1-** Subir as planilhas no bigquery, preservando seus nomes, colocando-as no dataset Spotify.

### 2.2- Identificar e tratar valores nulos

**2.2.1-** Usar COUNT(*) WHERE e IS NULL no BigQuery de pasta a pasta pra testar cada uma delas e formular tabelas de nulos.
** Após verificarf todas as tabelas encontrado apenas 50 campos nulos na tabela track_in_competition, do dataset projeto-2-hipoteses-420502, na coluna in_shazam_charts, e 95 campos nulos na coluna Key:

```sql
SELECT
  COUNT (*) AS `quantidade`
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_competition`
WHERE
  in_shazam_charts IS NULL
```

```sql
SELECT
  COUNT (*) AS `quantidade`
FROM
  `projeto-2-hipoteses-420502.Spotify.track_technical_info`
WHERE
  key IS NULL
```

### 2.3- Identificar e tratar valores duplicados

**2.3.1-**  ****No BigQuery verifiquei na tabela track_in_Spotify se constava alguma música duplicada, com o mesmo track_id, mas não foi encontrado track_id duplicado:

```sql
SELECT
  track_id,
  COUNT (*) AS quantidade
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
GROUP BY
  track_id
HAVING
  COUNT (*)>1
```

No BigQuery abri uma nova Query para verificar se poderia ter duas músicas com mesmo nome e artista, com track_id diferente, usando GROUP BY, e encontrei 4 músicas repetidas:

```sql
SELECT
  track_name,
  artist_s__name,
  COUNT (*) AS quantidade
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
GROUP BY
  track_name,
  artist_s__name
HAVING
  COUNT (*)>1
```

E tive o seguinte resultado, confirmando 4 músicas, sendo do mesmo artista, duplicadas:

| Linha | track_name | artist_s__name | f0_ |
| --- | --- | --- | --- |
| 1 | SNAP | Rosa Linn | 2 |
| 2 | About Damn Time | Lizzo | 2 |
| 3 | Take My Breath | The Weeknd | 2 |
| 4 | SPIT IN MY FACE! | ThxSoMch | 2 |

Abri uma nova Query para verificar outras informações das quatro músicas que foram encontradas duplicadas, para analisar se poderiam ser versões diferentes, lançadas em datas diferentes, e encontrei apenas a música About Damn Time com data de lançamento diferente:


```sql
SELECT
  track_id,
  track_name,
  artist_s__name,
  released_year,
  released_month,
  artist_count,
  streams,
  in_spotify_charts,
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
WHERE
  track_name IN ('SNAP',
    'About Damn Time',
    'Take My Breath',
    'SPIT IN MY FACE!');
```

Apenas para garantir que não se tratava de uma versão lançada talvesz no mesmo mês, abre uma nova query pra consultar as músicas duplicadas e suas informações técnicas da tabela track_technical_info, e não foram observadas grandes diferenças nas categorias técnicas das músicas:

```sql
SSELECT
  t1.track_id,
  t1.track_name,
  t1.artist_s__name,
  t1.released_year,
  t1.released_month,
  t1.artist_count,
  t1.streams,
  t1.in_spotify_charts,
  t2.*  -- seleciona todas as colunas da tabela track_technical_info
FROM 
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify` AS t1
JOIN
  `projeto-2-hipoteses-420502.Spotify.track_technical_info` AS t2
ON
  t1.track_id = t2.track_id
WHERE
  t1.track_name IN ('SNAP', 'About Damn Time', 'Take My Breath', 'SPIT IN MY FACE!');
```


Para retirar os valores duplicados, de uma forma genérica que permiti-se retirar novas músicas duplicadas caso surgissem na base de dados, construi com o auxílio da AI o comando abaixo que tomou como critério, caso uma mesma música de um mesmo artista estivesse duplicada, deveria verificar qual a música mais bem colocada no ranking do Spotify e caso nesse critério tivesse empate, deveria levar em considera a segunda variável que é o número total de streams, criando uma nova tabela Powerbi_corrigida :

```sql
CREATE OR REPLACE TABLE `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida`
SELECT *
FROM   
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify` ts
WHERE 
  ts.track_id NOT IN (
    SELECT t0.track_id
    FROM `projeto-2-hipoteses-420502.Spotify.track_in_spotify` AS t0
    INNER JOIN 
    (
        SELECT t1.track_name_1, t1.artist_s__name_1, t1.in_spotify_charts, MIN(t1.streams_1) AS min_streams
        FROM `projeto-2-hipoteses-420502.Spotify.track_in_spotify` AS t1
          INNER JOIN 
            (
              SELECT track_name_1, artist_s__name_1, MIN(COALESCE(in_spotify_charts,0)) AS min_charts
              FROM `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
              GROUP BY track_name_1, artist_s__name_1
              HAVING COUNT(*) > 1
            ) AS t2
            ON t1.track_name_1 = t2.track_name_1 AND t1.artist_s__name_1 = t2.artist_s__name_1 AND t1.in_spotify_charts = t2.min_charts
        GROUP BY t1.track_name_1, t1.artist_s__name_1, t1.in_spotify_charts
    ) AS t1
    ON t0.track_name_1 = t1.track_name_1 AND t0.artist_s__name_1 = t1.artist_s__name_1 AND t0.in_spotify_charts = t1.in_spotify_charts 
    AND t0.streams_1 = t1.min_streams
  )
```

### 2.4 - Identificar e gerenciar dados fora do escopo de análise

2.4.1- Identifiquei como dados fora do escopo da análise: key e mode em track_technical_info e in_shazam_charts que tinham 50 nulos, mas que também não faria tanta diferença na análise por não ser um app de stream. Por isso não retirei da planilha apenas ignorei esses dois campos nas análise futuras.

### 2.5- Identificar e tratar dados discrepantes em variáveis categoricas.

2.5.1- Utilizei o comando abaixo para tratar caracteres especiais na coluna track_name, onde temos os nomes das músicas:

```sql
SELECT
track_name,
  REGEXP_REPLACE(track_name, r'[^\w\s]', '') AS track_name_1
FROM
  `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida`
```

2.5.2- Atulizei a tabela com o campo track_name_1:

```sql
CREATE OR REPLACE TABLE ´projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida´ AS
SELECT *, 
track_name,
  REGEXP_REPLACE(track_name, r'[^\w\s]', '') AS track_name_1
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
```
### 2.6-Identificar e tratar dados discrepantes em variáveis numéricas

2.6.1- Ao realizar a consulta abaixo, encontrei um campo na coluna streams correspondete a uma string, sendo ele: BPM110KeyAModeMajorDanceability53Valence75Energy69Acousticness7Instrumentalness0Liveness17Speechiness3

```sql
SELECT
  MAX (streams),
  MIN (streams),
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
```
2.6.2- Para alterar a coluna streams para numérica e corrigir caso algum campo seja string, pesquisei um comando para filtrar esses casos e trazer somente registros onde a coluan streams seja numerica, após isso foi alterado o tipo da coluna de String para Integer

```sql
SELECT *,
   CAST(ts.streams AS INT64) AS streams_1
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
WHERE REGEXP_CONTAINS(ts.streams, r'^\d+$');
```

2.6.3- Para atualizar a tabela projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida, utilizei o código abaixo: 

```sql
CREATE OR REPLACE TABLE  `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida` AS
SELECT *,
   CAST(ts.streams AS INT64) AS streams_1,
FROM
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify`
WHERE REGEXP_CONTAINS(ts.streams, r'^\d+$');
```
### 2.7- Criar novas variáveis

2.7.1- Criei uma nova variável concatenando o ano-mês-dia de lançamento das músicas:

```sql
SELECT
  DATE(CONCAT(released_year, "-", released_month, "-", released_day)) AS data_completa
FROM
  `projeto-2-hipoteses-420502.Spotify.new_track_in_spotify`
```

2.7.2- Para atualizar a tabela projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida, utilizei o código abaixo:

```sql
CREATE OR REPLACE TABLE  `projeto-2-hipoteses-420502.Spotify.new_track_in_spotify` AS
SELECT *,
  DATE(CONCAT(released_year, "-", released_month, "-", released_day)) AS data_completa
FROM
  `projeto-2-hipoteses-420502.Spotify.new_track_in_spotify`
```
### 2.9- Unir (join) as tabelas de dados

2.9.1- Abri um nova query, e useI LEFT JOIN para uni-las. Para realizar esta etapa, pesquisei formulações do código no CHATGPT:

```sql
SELECT ts.*,
tc.in_apple_playlists,
tc.in_apple_charts,
tc.in_deezer_playlists,
tc.in_deezer_charts,
tc.in_shazam_charts,
ti.bpm,
ti.key,
ti.mode,
ti.danceability__,
ti.valence__,
ti.energy__,
ti.acousticness__,
ti.instrumentalness__,
ti.liveness__,
ti.speechiness
FROM      
  `projeto-2-hipoteses-420502.Spotify.track_in_spotify` ts
LEFT JOIN 
  `projeto-2-hipoteses-420502.Spotify.track_in_competition` tc ON tc.track_id = ts.track_id
LEFT JOIN 
  `projeto-2-hipoteses-420502.Spotify.track_technical_info` ti ON ti.track_id = ts.track_id
```

### 2.10- Construir tabelas de dados auxiliares

2.10.1- Criei tabelaS temporárias, para verificar quantidade de músicas por artista e total de streams: 

```sql
WITH tabela_temporaria_artista_solo AS (
  SELECT
    artist_s__name_1,
    COUNT(DISTINCT track_id) AS total_tracks,
    SUM(streams_1) AS total_streams_1
  FROM
    `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida`
  WHERE
    artist_count = 1
  GROUP BY
    artist_s__name_1
)

SELECT
  artist_s__name_1,
  total_tracks,
  tabela_temporaria_artista_solo.total_streams_1
FROM
  tabela_temporaria_artista_solo
ORDER BY
  total_tracks DESC;
```

## **3. Fazer uma análise exploratória**

### 3.1- Pré

3.1.1- Conectei as informações da base de dados já tratadas pelo BigQuery ao Power BI seguindo passo a passo o seguinte vídeo: [https://www.youtube.com/watch?v=OPrjsidrXQI&t=94s](https://www.youtube.com/watch?v=OPrjsidrXQI&t=94s)

### 3.2- Agrupar dados de acordo com variáveis categóricas

3.2.1- Agrupei os dados partindo dos pedidos de análise do case, usando matrizes do Power BI:

- A) Músicas com BPM (Batidas Por Minuto) mais altos fazem mais sucesso em termos de número de streams no Spotify.
- B) As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas, como a Deezer.
- C) A presença de uma música em um maior número de playlists está correlacionada com um maior número de streams.
- D) Artistas com um maior número de músicas no Spotify têm mais streams.
- E) As características da música influenciam o sucesso em termos de número de streams no Spotify.

3.2.1-Formato das matrizes:

A) BPM x Streams:

![Untitled](./images/04300203-cfba-401b-9e0e-edb7b9688455.png)

B) Charts: spotify/deezer/apple:

![Untitled](./images/41016d59-2b39-42f1-8e42-a73b884f9a34.png)

C) Streams x Playlists: 

![Untitled](./images/285e539b-6918-4b77-859c-1589da471d41.png)

D)Contagem de músicas x streams: 

![Untitled](./images/1456f482-c130-42d2-aa39-c2541e2a7d16.png)

### 3.4. Visualizar variáveis categóricas

3.4.1- Criei gráficos de barra e linha no Power BI e visualizei como alguns deles já afirmam ou refutam as hipóteses. Como, alto BPM não garante que uma música tenha mais streams.

3.4.2- Um pouco das dificuldades com as visualizações: Essa base de dados traz alguns outliers. E tornam os números de streams muito discrepantes. Alguns na casa dos bilhões, outros na casa do milhares. Vejo que é necessário segmentar, para ter visualizações melhores, e comparar artistas com maiores números em suas categorias. 

### 3.5. Aplicar medidas de tendência central

3.5.1- A partir de matrizes, fui criando colunas com a soma, média, e, mediana. Decidi por aplicar as medidas de tendência central nas variáveis: streams, bpm, playlists, charts por artista, e, média de playlists por música. Minha página de matrizes ficou assim:

![Untitled](./images/df783c17-0f63-4f67-bdd6-e02be4e0e785.png)

### 3.6. Visualizar a distribuição dos dados

3.6.1- Para visualizar a distribuição das variáveis foram usados histogramas; Para usá-los, é preciso instalar e acionar o python.

3.6.2- Com python já instalado e acionado no Power BI, criei o histograma para a variávels streams. Utilizando o código no script python no power bi:

```python
# O código a seguir para criar um dataframe e remover as linhas duplicadas sempre é executado e age como um preâmbulo para o script: 

# dataset = pandas.DataFrame(undefined)
# dataset = dataset.drop_duplicates()

# Cole ou digite aqui seu código de script:

import matplotlib.pyplot as plt
import pandas as pd

# Obtenha os dados do Power BI - você só precisa alterar essas informações de todo o código
# Substitua 'dataset' pelo nome do seu DataFrame e 'YOUR VARIABLE' pelo nome da sua variável
data = dataset[['bpm']]

#data = dataset[['streams_1']]

# Crie o histograma
plt.hist(data['streams_1'], bins=10, color='blue', alpha=0.7)
plt.xlabel('Value')
plt.ylabel('Frequency')
plt.title('Histogram')

# Mostre o histograma
plt.show()
```

O histograma ficou assim:


COLOCAR IMAGEM

### 3.7. Aplicar medidas de dispersão

3.7.1- As medidas de dispersão usadas foram: Desvio padrão e variância. Ambas foram feitas diretamente no Power BI usando tabelas matriz:


3.7.2- Conceitos importantes: 

**Desvio padrão**: O desvio padrão mede a que distância os valores individuais estão da média (média) de um conjunto de dados. Um desvio padrão mais alto indica maior dispersão.

**Variância**: A variância é o quadrado do desvio padrão. Ao elevar ao quadrado os desvios, os sinais positivos e negativos são eliminados, tornando-o útil para o cálculo. É outra medida de dispersão em torno da média.

Interpretação do desvio padrão:

**O desvio padrão é interpretado da seguinte forma:**

1. Quanto maior for o desvio padrão, maior será a dispersão dos dados. Isso significa que os valores individuais tendem a estar mais distantes da média. Portanto, um desvio padrão elevado indica maior variabilidade nos dados.

2. Quanto menor for o desvio padrão, menor será a dispersão dos dados. Isso significa que os valores individuais tendem a estar mais próximos da média. Um desvio padrão baixo indica menos variabilidade nos dados.

3. O desvio padrão pode ser interpretado como uma medida de risco ou incerteza em determinados contextos. Por exemplo, em finanças, um maior desvio padrão nos retornos de um investimento indica maior risco.

4. O desvio padrão é útil para comparar a dispersão entre diferentes conjuntos de dados. Pode ajudar a determinar qual dos dois conjuntos de dados tem maior variabilidade.

### 3.8.Visualizar o comportamento dos dados ao longo do tempo

3.8.1- Usando as ferramentas do Power BI, criei gráfico de linha, com eixo X data_completa_ano e eixo Y soma_streams_1 , e o gráfico  ficou assim:

COLOCAR IMAGEM

### 3.9. Calcular quartis, decis ou percentis

3.9.1- Decidi fazer passo a passo, criando novas tabelas por quartil e categoria para cada variável:

```sql
--criando tabela quartil:streams
CREATE OR REPLACE TABLE `projeto-2-hipoteses-420623.projeto2.quartil_streams`
AS
  WITH Quartile AS (
  SELECT
    track_id,
    streams_1,
    NTILE(4) OVER (ORDER BY streams_1) AS quartile_streams_alto_baixo
  FROM `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida`
)
SELECT
  a.*,
  q.quartile_streams_alto_baixo,
  IF(q.quartile_streams_alto_baixo = 4, "alto", "baixo") AS quartile_classification
FROM `projeto-2-hipoteses-420502.Spotify.Powerbi_corrigida` AS a
LEFT JOIN Quartile AS q
ON a.track_id = q.track_id
  

--categorizando os quartis:streams
SELECT
  streams,
  quartil_streams,
  IF(quartil_streams = 4, "alto", IF(quartil_streams = 3, "médio", "baixo")) AS categoria_streams
FROM `projeto2.quartil_streams`;

```

3.9.2- Depois uni cada uma delas a minha tabela principal nomeada uniao_tabelas:

```sql
--unindo tabela de categorias e quartis a tabela uniao_tabelas

--unindo quartil_streams e categoria_streams
CREATE OR REPLACE TABLE `projeto2.uniao_tabelas` AS
SELECT
  t1.*,
  t2.* EXCEPT (streams)
FROM `projeto2.uniao_tabelas` AS t1
LEFT JOIN `projeto2.categoria_streams` AS t2
ON t1.streams = t2.streams;

--unindo quartil_bpm e categoria_bpm
CREATE OR REPLACE TABLE `projeto2.uniao_tabelas` AS
SELECT
  t1.*,
  t2.* EXCEPT (bpm)
FROM `projeto2.uniao_tabelas` AS t1
LEFT JOIN `projeto2.categoria_bpm` AS t2
ON t1.bpm = t2.bpm;

--unindo quartil_in_spotify_playlists e categoria_in_spotify_playlists
CREATE OR REPLACE TABLE `projeto2.uniao_tabelas` AS
SELECT
  t1.*,
  t2.* EXCEPT(in_spotify_playlists)
FROM `projeto2.uniao_tabelas` AS t1
LEFT JOIN `projeto2.categoria_in_spotify_playlists` AS t2
ON t1.in_spotify_playlists = t2.in_spotify_playlists;

--unindo quartil_track_count e categoria_track_count
CREATE OR REPLACE TABLE `projeto2.uniao_tabelas` AS
SELECT
  t1.*,
  t2.* EXCEPT(track_count)
FROM `projeto2.uniao_tabelas` AS t1
LEFT JOIN `projeto2.categoria_track_count` AS t2
ON t1.track_count = t2.track_count;

```

### 3.10. Calcular correlação entre variáveis

3.10.1- Calculei as correlações:

```sql
-- calculando correlação de pearson

SELECT
CORR(streams,in_spotify_playlists) AS corr_spotify_playlists,
CORR(streams,in_spotify_charts) AS corr_spotfy_charts,
CORR(streams,in_deezer_playlists) AS corr_deezer_playlists,
CORR(streams,in_deezer_charts) AS corr_deezer_charts,
CORR(streams,in_apple_playlists) AS corr_apple_playlists,
CORR(streams,in_apple_charts) AS corr_apple_charts,
CORR (streams,danceability__) AS corr_danceability,
CORR(streams,energy__) AS corr_energy,
CORR(streams,liveness__) AS corr_liveness,
CORR(streams,bpm) AS corr_bpm,
CORR(streams,speechiness__) AS corr_speechiness,
CORR(streams,valence__) AS corr_valence,
CORR(streams,acousticness__) AS corr_acousticness,
CORR(streams,instrumentalness__) AS corr_instrumentalness
FROM `projeto2.uniao_tabelas`;

```

3.10.2- Resultados da correlação de Pearson:



## **4. Aplicar técnica de análise**

### 4.1. Aplicar segmentação

4.1.1- Para segmentar meus dados pra uma análise, os separei por quartis, especificamente, o quartil de streams. Onde minha base de dados foi segmentada em 4 grupos, de acordo com o número de streams. Sabendo que, o 4 quartil é o que tem o maior número de streams. Minhas matrizes usaram como linha o quartil de 1 a 4, e, como valores, a média das variáveis:

## 4. Conclusões e validação das hipóteses

4.1 - Músicas com BPM (Batidas Por Minuto) mais altos fazem mais sucesso em termos de número de streams no Spotify. Hipótese refutada. Ao realizar os testes, e observar os gráficos de dispersão.



4.2- As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas, como a Deezer. Hipótese validada, temos sim comportamentos moderadamente semelhantes entre as plataforma.


4.3- A presença de uma música em um maior número de playlists está correlacionada com um maior número de streams. Sim, hipotése validada. São 



4.4- Artistas com um maior número de músicas no Spotify têm mais streams. Hipótese refutada. Além disso, artistas com streams tem músicas que alcançam mais números de streams do que artistas com grande número de músicas.



4.5- As características da música influenciam o sucesso em termos de número de streams no Spotify. Hipótese refutada. Os cálculos e gráficos mostram que não há correlação.


