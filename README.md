
# Topicos 
- **[IndexMS](#IndexMS)**
- **[Docker](#Docker)**
- **[Kibana](#Kibana)**
- **[Implementação](#Implementação)**

# IndexMS 

- Análises aqui sobre um processo prático para restaurar um índice localmente, porém, por questões de modelo de infra local, o melhor é fazer a carga a partir de um CSV.
-  É importante que mantenham o nome do índice igual ao nome do arquivo (modelo_exemplo), e também o envio do padrão de criação do índice (mapping) para preservar o campo georrefenciado - centróide do municipío. 
- Este índice tem em torno de X mil linhas(documentos) e o CSV a ser utilizado possui o header com o nome dos campos. Por ser um CSV, os campos q são do tipo texto estão quotados ("").  
- O método de carga utilizado para o índice é usando logstash, configurando um arquivo do tipo pipeline. Após a carga, para conferência  e estudo do dado, e um exemplo do painel a ser construído no kibana como exercício.

# Docker

- Utilização de containers para subir localmente o __ElasticSearch__ e o __Kibana__ 
- Versao Utilizada: 7.4.2 
- Disponiveis no arquivo docker-compose.yml

``` yml
version: '3.0'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.4.2
    container_name: elasticsearchMS
    environment:
      - discovery.type=single-node
    volumes:
      - MS_kibana:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.4.2
    environment:
      SERVER_NAME: kibana.example.org
      ELASTICSEARCH_URL: http://localhost:9200
    ports:
      - "5601:5601"
volumes:
  MS_kibana:
```

- Com o Docker Compose instalado na maquina, via linha de terminal executar 
``` sh 
docker-compose up
```





# Kibana

Usando a ferrametnta Dev Tools do Kibana, é bem simples criar o índice, conforme o print acima
Para este índice, segue abaixo o modelo de criação com PUT

``` http 
DELETE modelo_exemplo

GET modelo_exemplo

PUT modelo_exemplo
{
  "mappings": {
    "properties": {
      "pt_municipio": {
        "type": "geo_point"
      }
    }
  }
}

```

# Implementação

- Através da ferramenta LogStash, será realizada a tranformação do modelo_exemplo.csv em um formato indexavel no ElasticSearch, e para isso, deve-se configurar o arquivo.conf com os dados de entrada, filtro e saida.
- Neste caso, a entrada está no formato CSV.
- E para execução do mesmo, é importante descrever no arquivo, as colunas presentes no mesmo.

- Configuração do Arquivo  .conf

```sh
input {
  file {
    #Caminho do CSV 
    path => "/home/higor/Documents/Index/modelo_exemplo.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => multiline {
      pattern => "^(\d){6};"
      negate => true
      what => "previous"
      auto_flush_interval => 1
    }
  }
}
filter {
  csv {
    #Colunas Presentes no CSV
    columns => [ "id","ano","ano_mes","co_mes","co_municipio_ibge","co_uf_ibge","municipio","sg_uf","situacao_implantacao_cenario1","situacao_implantacao_cenario2","sg_mes","mes_ano","qtde","data_competencia","pt_municipio","co_regiao_ibge","regiao_ibge","tipologia_ibge_mun","uf"]
    skip_header => true
    separator => ";"
  }

  mutate {
    #Formato das tabelas
    convert => {
      "id" => "integer"
      "ano" => "integer"
      "ano_mes" => "integer"
      "co_mes" => "integer"
      "co_municipio_ibge" => "integer"
      "co_uf_ibge" => "integer"
      "municipio" => "string"
      "sg_uf" => "string"
      "situacao_implantacao_cenario1" => "string"
      "situacao_implantacao_cenario2" => "string"
      "sg_mes" => "string"
      "mes_ano" => "string"
      "qtde" => "integer"
      "data_competencia" => "string"
      "co_regiao_ibge" => "integer"
      "regiao_ibge" => "string"
      "tipologia_ibge_mun" => "string"
      "uf" => "string"
    }
  }

  mutate {
  remove_field => [ "message", "path", "host" ]
  }
}
output {
    stdout { codec => dots }
    
    stdout { codec => rubydebug }

    elasticsearch {
     hosts => ["localhost:9200"]
     index => "modelo_exemplo"
    }
}
```


- Rodar o *LogStash* com o conf relacionada com a tabela CSV desejada.


 ```sh 
 bin/logstash -f log_municipio.conf
 ```

**[⬆ Subir para o Topo](#Topicos)**



