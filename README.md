# elasticTitanic

Élements de la présentation Elasticsearch basée sur des données de passagers du Titanic.

Cette présentation a été crée grâce au soutien de Apside.

## Données

Fichier de données by Matthew Brett! 
https://matthew-brett.github.io

https://matthew-brett.github.io/cfd2020/_downloads/b9522d0cb41676fd6241ad70a17f6409/titanic_clean.csv

## Monter un environnement sur Podman

J'ai utilisé Podman sur WSL 2

1. Créer le réseau local
```
   docker network create elasticnetwork
```

2. Démarrer un container Elasticsearch

```
podman run -d \
    --name elasticsearch \
    --net elasticnetwork \
    -p 9200:9200 \
    -e discovery.type=single-node \
    -e ES_JAVA_OPTS="-Xms1g -Xmx1g" \
    -e xpack.security.enabled=false \
    -it \
    docker.elastic.co/elasticsearch/elasticsearch:8.6.0
```

3. Démarrer un container Kibana

```
podman run -d \
    --name kibana \
    --net elasticnetwork \
    -p 5601:5601 \
    docker.elastic.co/kibana/kibana:8.6.0
```

Voir Elasticsearch
http://localhost:9200/

Voir Kibana
http://localhost:5601/

## Explorer Elastic

L'ensemble des appels APIs utilisés dans cette présentation suivent dans ce fichier.

Note : ils sont présentés tels qu'ils sont envoyés directement via la page "Dev tools" de Kibana.

### Nettoyer les index (si vous voulez tout recommencer !)

DELETE titanic-demo-00001
DELETE titanic-passagers-00001
DELETE _ilm/policy/titanic_policy

### Création d'une policy

Gestion du cycle de vie de l'index, etc.

Exemple :

```
PUT _ilm/policy/titanic_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "set_priority": {
            "priority": 100
          },
          "rollover": {
            "max_primary_shard_size": "30gb",
            "max_age": "30d"
          }
        }
      }
    }
  }
}
```

### Création d'un index associé

```
PUT titanic-demo-00001
{
    "settings": {
        "lifecycle": {
            "name": "titanic_policy",
            "rollover_alias": "titanic"
        },
        "number_of_shards": "3",
        "number_of_replicas": "1"
    },
    "aliases": {
        "titanic": {
            "is_write_index": true
        }
    }
}
```

### Visualiser les caractéristiques de l'index 

```
GET titanic-demo-00001
```

### Compter les documents

```
GET titanic-demo-00001/_count
```
Aucun pour le moment ;)

### Chercher

```
GET titanic-demo-00001/_search
```

### Charger les données

Les requêtes suivantes supposent que vous avez importé dans Elastic votre fichier .csv (présent dans ce dépôt, et mentionné au début de ce README). Pour celà, dans Kibana version 8.6, sur la page d'accueil, cliquez sur "Upload a file"

Vous déposez votre .csv dans drag and drop. Import. Import (Vous pouvez modifier le mapping ou les caractéristiques de l'index, dans Advanced, après avoir cliqué pour la première fois sur Import.

Ajouter un document via l'API Elastic :

```
POST /titanic-passagers-00001/_doc/
{
    "country": "France",
    "fare": 7.13,
    "gender": "male",
    "name": "Bechard, Mr. Baptiste",
    "embarked": "Cherbourg",
    "survived": "yes",
    "class": "3rd",
    "age": 36
}
```

### Explorer ces données

Compter les passagers du Titanic (nota : ces données ont été nettoyés, il n'y a donc pas la totalité des passagers réellement embarqués sur le Titanic, seuls les données complètes ont été conservées)

```
GET titanic-passagers-00001/_count
```

### Requêter

Boolean query ([Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html))

Should optionnelle (voir [minimum_should_match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html#bool-min-should-match))

```
GET titanic-passagers-00001/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "match_phrase": {
            "gender": "female"
          }
        },
        {
          "match_phrase": {
            "survived": "yes"
          }
        }
      ],
      "should": [
        {
          "match_phrase": {
            "country": "France"
          }
        },
        {
          "match_phrase": {
            "country": "England"
          }
        }
      ]
    }
  }
}
```

Should embed dans dans un filter context (nested boolean query) = Filter : on off ; Must / should : participation au score.

```
GET titanic-passagers-00001/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "match_phrase": {
            "gender": "female"
          }
        },
        {
          "match_phrase": {
            "survived": "yes"
          }
        },
        {
          "bool": {
            "should": [
              {
                "match_phrase": {
                  "country": "France"
                }
              },
              {
                "match_phrase": {
                  "country": "England"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

### Aggréger

Nous allons maintenant essayer de répondre à des questions plus complexes, via les aggrégations. On commence tranquille.

Nombre d'hommes et de femmes sur le Titanic

```
POST titanic-passagers-00001/_search
{
  "size": 0,
  "aggs": {
    "genre": {
      "terms": {
        "field": "gender",
        "size": 2
      }
    }
  }
}
```

Pyramide des âges par genre

```
POST titanic-passagers-00001/_search
{
  "size": 0,
  "aggs": {
    "genre": {
      "terms": {
        "field": "gender",
        "size": 2
      },
      "aggs": {
        "age": {
          "range": {
            "field": "age",
            "ranges": [
              {
                "from": 0,
                "to": 11
              },
              {
                "from": 11,
                "to": 21
              },
              {
                "from": 21,
                "to": 31
              },
              {
                "from": 31,
                "to": 41
              },
              {
                "from": 41,
                "to": 51
              },
              {
                "from": 51,
                "to": 61
              },
              {
                "from": 61,
                "to": 71
              },
              {
                "from": 71,
                "to": 81
              },
              {
                "from": 81
              }
            ]
          }
        }
      }
    }
  }
}
```

Pourcentage de survivants par sexe

```
POST titanic-passagers-00001/_search
{
  "size": 0,
  "aggs": {
    "genre": {
      "terms": {
        "field": "gender",
        "size": 2
      },
      "aggs": {
        "survived": {
          "terms": {
            "field": "survived"
          }
        },
        "pourcentage_survie": {
          "bucket_script": {
            "buckets_path": {
              "alive": "survived['yes']>_count",
              "dead": "survived['no']>_count"
            },
            "script": "params.alive/(params.alive + params.dead)*100"
          }
        }
      }
    }
  }
}
```

La [documentation des aggrégations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)
