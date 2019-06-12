put http://192.168.6.29:9200/test_hy4

{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "ngram",
          "min_gram": 4,
          "max_gram": 11
        }
      },
      "analyzer": {
        "autocomplete": { 
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "_doc": {
      "properties": {
        "text": {
          "type": "text",
          "analyzer": "autocomplete", 
          "search_analyzer": "standard" 
        }
      }
    }
  }
}