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



//////////////////////////////////////////////



put http://192.168.6.29:9200/cit3_contractcenter_se_contract_sc

consignee_name
consigneeMobile
outSid
createTime
updateTime




{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "ngram",
          "min_gram": 4,
          "max_gram": 30
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
        "consignee_name": {"type": "text", "analyzer": "autocomplete", "search_analyzer": "standard"} ,
        "consigneeMobile": {"type": "text", "analyzer": "autocomplete", "search_analyzer": "standard"} ,
        "outSid": {"type": "text", "analyzer": "autocomplete", "search_analyzer": "standard"} ,
        "createTime": {"type": "date"} ,
        "updateTime": {"type": "date"} 
      }
  }
  }
}
