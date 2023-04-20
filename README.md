## 注意 (增加了分词)

⚠️ kibana 连接 es 的账号密码 对应的 kibana.yml 以及第三步里面 kibana 的密码。

```php
elasticsearch.username: "kibana"
elasticsearch.password: "123456"
```

##### 启动命令

    docker-compose up -d

###### 第一步 进入 es 容器

```php
docker exec -it elasticsearch sh
```

###### 第二步 进入 es 容器

```php
bin/elasticsearch-setup-passwords interactive
```

###### 第三步 输入 y 下一步，进行密码设置

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

### Laravel php

#### 普通

```
<?php

namespace App\Http\Controllers;

use Elastic\Elasticsearch\ClientBuilder;

class ElasticSearch extends Controller
{
    public $client = null;
    public function __construct()
    {
        $this->client = ClientBuilder::create()
            ->setHosts(["http://elasticsearch:9200"])
            ->setBasicAuthentication('elastic', "123456")
            ->build();
    }
    public function infos()
    {
        $response = $this->client->info();
        echo "<pre>";
        print_R($response);
    }
    // 创建
    public function esCreate()
    {
        $doc = [
            'index' => 'my_index',
            'type' => 'doc',
            'id' => 2,
            'body' => [
                'title' => "一二三四五六七八九十",
                'content' => "额。数字啊",
            ],
            'form' => 0,
            'size' => 10,
        ];

        $response = $this->client->index($doc);
        dd($response->asArray());
    }
    public function esQuery()
    {
        $params = [
            'index' => 'my_index',
            'body' => [
                'query' => [
                    'match' => [
                        'title' => '一二'
                    ]
                ]
            ]
        ];
        $response = $this->client->search($params);
        dd($response->asArray());
    }
    public function esDelete()
    {
        $params = [
            'index' => 'my_index',
            'id' => 1,
        ];
        $response = $this->client->delete($params);
        dd($response->asArray());
    }

    public function esIkSearch()
    {
        $params = [
            'index' => 'my_index',
            'body' => [
                'settings' => [
                    'analysis' => [
                        'analyzer' => [
                            'ik_max_word' => [
                                'tokenizer' => 'ik_max_word',
                            ],
                        ],
                    ],
                ],
                'mappings' => [
                    'properties' => [
                        'title' => [
                            'type' => 'text',
                            'analyzer' => 'ik_max_word',
                        ],
                        'content' => [
                            'type' => 'text',
                            'analyzer' => 'ik_max_word',
                        ],
                    ],
                ],
            ],
        ];
    }
}
```

#### 分词

```
use Elasticsearch\ClientBuilder;

$client = ClientBuilder::create()->build();

// 创建索引
$params = [
    'index' => 'my_index',
    'body' => [
        'settings' => [
            'analysis' => [
                'analyzer' => [
                    'ik_max_word' => [
                        'tokenizer' => 'ik_max_word',
                    ],
                ],
            ],
        ],
        'mappings' => [
            'properties' => [
                'title' => [
                    'type' => 'text',
                    'analyzer' => 'ik_max_word',
                ],
                'content' => [
                    'type' => 'text',
                    'analyzer' => 'ik_max_word',
                ],
            ],
        ],
    ],
];

$client->indices()->create($params);

// 添加文档
$params = [
    'index' => 'my_index',
    'id' => '1',
    'body' => [
        'title' => '这是一个测试标题',
        'content' => '这是一个测试内容，包含一些关键词：测试、标题、内容',
    ],
];

$client->index($params);

// 更新文档
$params = [
    'index' => 'my_index',
    'id' => '1',
    'body' => [
        'doc' => [
            'title' => '这是一个更新后的标题',
            'content' => '这是一个更新后的内容，包含一些关键词：更新、标题、内容',
        ],
    ],
];

$client->update($params);

// 删除文档
$params = [
    'index' => 'my_index',
    'id' => '1',
];

$client->delete($params);

// 搜索文档
$params = [
    'index' => 'my_index',
    'body' => [
        'query' => [
            'multi_match' => [
                'query' => '测试标题',
                'fields' => ['title', 'content'],
                'analyzer' => 'ik_max_word',
            ],
        ],
    ],
];

$response = $client->search($params);

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
