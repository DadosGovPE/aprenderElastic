# Repositório de informações sobre o Elasticsearch

Esse repositório guarda aprendizados em explorações do Elasticsearch:

- [Mapping: definidor da natureza das variáveis](https://github.com/DadosGovPE/aprenderElastic?tab=readme-ov-file#mapping-definidor-da-natureza-das-vari%C3%A1veis)
- [Ingestão de dados aninhados](https://github.com/DadosGovPE/aprenderElastic#ingest%C3%A3o-de-dados-aninhados)
- [Cuidados com dados geoespaciais](https://github.com/DadosGovPE/aprenderElastic#cuidados-com-dados-geoespaciais)
- [Campos ausentes em alguns documentos](https://github.com/DadosGovPE/aprenderElastic#campos-ausentes-em-alguns-documentos)
- [Relatos do processo de implantação do Elastic](https://github.com/DadosGovPE/aprenderElastic#relatos-do-processo-de-implanta%C3%A7%C3%A3o-do-elastic)

## Mapping: definidor da natureza das variáveis

### 🤔Do que se trata o mapping?
A criação do **mapping** é etapa que imediatamente sucede a criação de um index. Ele define simultaneamente a estrutura e natureza (**type**) das variáveis (**fields**) de um banco de dados (**index**).

### ✍🏾Como criar um mapping?
A criação do mapping pode ser feita no **Console** do **Kibana Dev Tools** usando a estrutura usual dos mappings como se vê no exemplo abaixo. Caso o index ainda não exista, esse comando também o gera automaticamente.

```
PUT /index-teste
{
  "mappings": {
    "properties": {
      "fatorOrdenado": {
        "type": "keyword"
      },
      "texto": {
        "type": "text"
      },
      "numero": {
        "type": "double"
      },
      "inteiro": {
        "type": "integer"
      },
      "coordenadas": {
        "type": "geo_point"
      },
      "listaVariaveis": {
        "properties": {
          "var1": {
            "type": "keyword"
          },
          "var2": {
            "type": "text"
          }
        }
      }
    }
  }
}
```

Uma alternativa é criar o mapping usando o pacote **{elastic}** do **R**. O exemplo abaixo reproduz o mapping acima:

```
# Define o mapping usando listas (ou JSON)
body <- list(
  properties = list(
    fatorOrdenado = list(type = "keyword"),
    texto = list(type = "text"),
    numero = list(type = "double"),
    inteiro = list(type = "integer"),
    coordenadas = list(type = "geo_point"),
    listaVariaveis = list(
      properties = list(
        var1 = list(type = "keyword"),
        var2 = list(type = "text")
      )
    )
  )
)

# Cria o index (obrigatório, caso ele ainda não exista)
elastic::index_create(conn, index = "index-teste")

# Passa o mapping para o index
elastic::mapping_create(conn, index = "index-teste", body = body)
```

### ⚠️ Efeitos importantes:
#### O mapping restringe quais os valores e formatos são aceitáveis em um determinado field
No index-teste acima temos fields numéricos como "numero" e "inteiro". Uma ingestão de documentos que contenham textos nessas variáveis irá falhar. Similarmente, os fields de data ou geo_point precisam ter formato(s) padronizado(s). Documentos que tenham data "YYYY-mm-dd" em um field com formato "dd-mm-YYYY" irão falhar.

#### O mapping não impede a criação de novos fields
É possível realizar uma ingestão sem obedecer a estrutura do mapping, todavia o Elastic irá se encarregar de definir o type dessas novas variáveis.

#### O mapping auxilia a definir a natureza de variáveis como geo_point e date
Quando um type dessa natureza é definido o Elastic busca interpretar (parse) os fields dessa forma. Se o formato da coluna estiver dentro de um dos padrões a ingestão acontece com sucesso. A documentação do Elastic indica alguns dos formatos aceitáveis para ingestão de geo_point e date.

#### O mapping pode evitar tarefas de reindex
A definição prévia do type de uma variável evita a necessidade de reindexar um index para meramente converter uma variável de um tipo a outro. Isso é especialmente importante para variáveis aninhadas que são particularmente difíceis de manejar com a linguagem Painless.

## Ingestão de dados aninhados
### 🤔Por que usar dados aninhados?
Para o Elastic é importante a definição de uma unidade de observação do banco de dados. **Cada documento deve idealmente conter apenas uma observação.** Infelizmente alguns bancos contém múltiplas unidades de medida possíveis. Há dois casos emblemáticos de como esse problema se manifesta:

Em um caso, o banco pode ter **múltiplos níveis de agregação.** Por exemplo, um banco de dados de boletins de ocorrência que contém todos os registros históricos de um mesmo BO conforme o mesmo é atualizado. O mesmo banco pode ser apresentado a nível de registro ou a nível de BO (ao agregar todos os registros em uma única linha).

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/baedf756-c28b-4418-854a-5e50d9a6873c)

Em outro caso, a **união de bancos** de dados pode gerar um banco com **unidade problemática.** Por exemplo, dados dois bancos de dados de uma escola:

um contém todos os estudantes por turma

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/1a0f87d4-52e6-4da4-ab81-024301ff74b1)

e outro contém todas as notas de cada turma em diferentes disciplinas.

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/4c5c7f84-2285-4040-bc71-672465ac72e7)

Se a única chave que une os bancos for a turma, não é possível associar o estudante a uma nota. A união desses bancos teria uma relação "múltiplos a múltiplos" e cada aluno seria associado a todas as notas de sua turma. O que não corresponde à realidade.

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/9778ab11-499f-4212-b0c2-bf851b87469a)

Uma solução para tais problemas é agregar os dados em listas de sorte a alcançar uma unidade de medida satisfatória.

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/c6f94641-90de-48bc-9383-f513ac35263e)

![image](https://github.com/DadosGovPE/aprenderElastic/assets/7217965/a2ee15d5-84e9-4e28-bbcc-296e2f6de929)

### ✍🏾Como realizar a ingestão de uma base com dados aninhados?
Construir bancos de dados com colunas-lista é uma tarefa simples em R. A ingestão no Elastic também é trivial através da função **elastic::docs_bulk()**. O exemplo abaixo mostra como criar os bancos de dados estudantis, agregar as observações, unir os bancos e fazer a ingestão.

```
# Gera o banco de estudantes da turma
dadosEstudantes <- dplyr::tibble(
  turma = c("A", "A"),
  estudante = c("Zeus", "Athena")
)

# Gera o banco de notas da turma
dadosNotas <- dplyr::tibble(
  turma = c("A", "A", "A", "A"),
  nota = c(5, 10, 7, 6),
  curso = c("Matemática", "Português", "Matemática", "Português")
)

# Colapsa o banco de estudantes por turma
dadosEstudantes <- dadosEstudantes |> 
  dplyr::summarise(.by = turma, estudante = list(estudante))

# Colapsa o banco de notas por turma
dadosNotas <- dadosNotas |> 
  tidyr::nest(.by = turma, .key = "desempenho")

# Une os bancos pela chave "turma"
dados <- dplyr::full_join(dadosEstudantes, dadosNotas)

# Cria o index (obrigatório, caso ele ainda não exista)
elastic::index_create(conn, index = "index-teste")

# Realiza a ingestão
elastic::docs_bulk(conn, dados, index = "index-teste")
```
O Elastic irá construir fields aninhados automaticamente, mesmo se você não passar previamente o mapping (embora é importante passar o mapping conforme comentado no último parágrafo em [Mapping: definidor da natureza das variáveis](https://github.com/DadosGovPE/aprenderElastic?tab=readme-ov-file#mapping-definidor-da-natureza-das-vari%C3%A1veis)). O nome da variável aninhada (**nested**) é construído referenciando as variáveis nas quais ela está aninhada. Por exemplo, a variável **"curso"**, aninhada na variável **"desempenho"** passa a se chamar **"desempenho.curso"** no Kibana. Abaixo temos o mapping gerado automaticamente pelo Elastic para o banco do exemplo:

```
{
  "mappings": {
    "_doc": {
      "properties": {
        "desempenho": {
          "properties": {
            "curso": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "nota": {
              "type": "long"
            }
          }
        },
        "estudante": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "turma": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      }
    }
  }
}
```
## Cuidados com dados geoespaciais
Mapas aceitam fields com listas de geo_point e exibem cada um individualmente. Todavia as tooltips sempre exibem informações do documento por inteiro.

## Campos ausentes em alguns documentos
O Elastic usualmente lida bem com o caso onde os vários documentos a serem ingeridos tem diferentes quantidades de campos. Novos campos que surgem vão sendo acrescentados ao mapping. Campos do mapping ausentes em documentos são ignorados.

Todavia há um curioso fenômeno quando se constrói um **index pattern** com **timestamp**: documentos sem a variável usada como timestamp são omitidas do Discover do Kibana. Mesmo estando presentes no index.

Por exemplo: dado um index com 100 documentos, 10 deles sem timestamp. Ao observar tal index no Discover só poderemos observar 90 documentos. Mesmo que seja realizada na ferramenta uma busca por documentos onde o campo não existe, tais documentos não serão exibidos.

## Relatos do processo de implantação do Elastic
Na medida que a máquina com o Elastic foi implantada e operada por nós, surgiram alguns problemas:
- Pouca **memória alocada** para o docker do elasticsearch. Resolvido com maior alocação;
- **Armazenamento cheio** na partição var. Nela fica o docker e também os bancos de dados do Elastic. Resolvido com expansão do armazenamento;
- **Armazenamento cheio** na partição home. Nela ficam as pastas dos usuários. Realizamos os processos de carregamento dos dados, conversão de csv a json e ingestão no elastic a partir dela. Resolvido com expansão do armazenamento;
- Queda do docker e **volta não-automática**. Resolvido com automatização da volta do docker em caso de queda. Caso a queda persista pode-se usar o comando **"docker start NAME"** em qualquer local, onde NAME é o nome do docker a reiniciar;
- Arquivos de **.yml** de configuração do Elastic e Kibana tem **localização incerta**. Geralmente há duas instâncias em pastas geradas aleatoriamente dento do docker. Solucionado através do **mapeamento do .yml** para outro arquivo .yml fora do docker em local fixo.
