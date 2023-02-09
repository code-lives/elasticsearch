
## 注意
⚠️  kibana连接es的账号密码 对应的kibana.yml以及第三步里面kibana的密码。

```php
elasticsearch.username: "kibana"
elasticsearch.password: "123456"
```
##### 启动命令
    docker-compose up -d
###### 第一步 进入es容器
```php
docker exec -it elasticsearch sh
```
###### 第二步 进入es容器
```php
bin/elasticsearch-setup-passwords interactive
```
###### 第三步 输入y 下一步，进行密码设置
```php
    Enter password for [elastic]:
    Reenter password for [elastic]:
    Enter password for [apm_system]:
    Reenter password for [apm_system]:
    Enter password for [kibana_system]:
    Reenter password for [kibana_system]:
    Enter password for [logstash_system]:
    Reenter password for [logstash_system]:
    Enter password for [beats_system]:
    Reenter password for [beats_system]:
    Enter password for [remote_monitoring_user]:
    Reenter password for [remote_monitoring_user]:
    Changed password for user [apm_system]
    Changed password for user [kibana_system]
    Changed password for user [kibana]
    Changed password for user [logstash_system]
    Changed password for user [beats_system]
    Changed password for user [remote_monitoring_user]
    Changed password for user [elastic]
```
##### 登录，账号密码对应的 第三步 第一个设置的密码
```php
127.0.0.1:5601
账号 elastic
密码 123456
```

### go
```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"log"
	"strings"

	elasticsearch "github.com/elastic/go-elasticsearch/v8"
	"github.com/elastic/go-elasticsearch/v8/esapi"
)

func main() {
	cfg := elasticsearch.Config{
		Addresses: []string{
			"http://127.0.0.1:9200",
		},
		Username: "elastic",
		Password: "123456",
	}
	es, err := elasticsearch.NewClient(cfg)
	if err != nil {
		log.Fatalf("Error creating the client: %s", err)
	}
	res, err := es.Info()
	if err != nil {
		log.Fatalf("Error getting response: %s", err)
	}
	defer res.Body.Close()
	fmt.Println("connect to es success")
	// 新增
	add := add(es)
	fmt.Println("新增：", add)
	// 查询
	query := query(es)
	fmt.Println("query:", query.String())
	//删除
	deltete := delete(es)
	fmt.Println("query:", deltete)
}

// 新增
func add(es *elasticsearch.Client) *esapi.Response {
	add, err := es.Index(
		"test",
		strings.NewReader(`{"title" : "hello word"}`),
		es.Index.WithRefresh("true"),
		es.Index.WithPretty(),
		es.Index.WithFilterPath("result", "_id"),
	)
	if err != nil {
		panic(err)
	}
	return add
}

// 查询
func query(es *elasticsearch.Client) *esapi.Response {

	var buf bytes.Buffer
	where := map[string]interface{}{
		"query": map[string]interface{}{
			"match": map[string]interface{}{
				"title": "hello word",
			},
		},
	}
	if err := json.NewEncoder(&buf).Encode(where); err != nil {
		log.Fatalf("Error encoding query: %s", err)
	}
	query, err := es.Search(
		es.Search.WithContext(context.Background()),
		es.Search.WithIndex("test"),
		es.Search.WithBody(&buf),
		es.Search.WithTrackTotalHits(true),
		es.Search.WithPretty(),
	)
	if err != nil {
		panic(err)
	}
	return query
}

// 删除
func delete(es *elasticsearch.Client) *esapi.Response {
	// 删除test索引下id=dae5NoYBvw08KezVlFPX
	delete, err := es.Delete("test", "dae5NoYBvw08KezVlFPX")
	if err != nil {
		panic(err)
	}
	return delete
}

```
