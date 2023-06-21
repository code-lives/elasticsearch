## 注意 (增加了分词)

⚠️ kibana 连接 es 的账号密码 对应的 kibana.yml 以及第三步里面 kibana 的密码。

## 前置条件

logstash 文件下的内容 是提前从容器复制出来的一份，修改了 logstash.yml 的连接 es 内容

logstash 连接器 用于连接 msyql https://mvnrepository.com/artifact/mysql/mysql-connector-java

### conf.d 文件 写了 mysql.conf 连接配置等。

如果自己写的脚本运行报错 会参数一个.lock 的文件 记得删除【config】文件下

### logstash/config 文件下的 pipelines.yml 配置了 自动执行 mysql.conf

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

## logstash

启动后默认连接数据库 test 表 读取了 att_faq 表数据，id,content 两个字段存储在 es 的 test 索引中（不需要的话 修改 pipelines.yml 配置）

### 配置文件【logstash】，是预先从 config 里面复制出来的

### mysql-connector-java-5.1.49.jar 上面的提到连接器（可能映射后有权限问题,可以提前自定义镜像，或者加命令行）

```php
ls -l /usr/share/logstash/config/mysql-connector-java-5.1.49.jar
chmod +r /usr/share/logstash/config/mysql-connector-java-5.1.49.jar

```

### Laravel

使用了分词

```php
<?php

namespace App\Http\Controllers;

use Elastic\Elasticsearch\ClientBuilder;
use Illuminate\Support\Facades\DB;

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
    // 创建分词索引
    public function esCreateIk()
    {
        $params = [
            'index' => 'ik',
            'body' => [
                'mappings' => [
                    'properties' => [
                        'content' => [
                            'type' => 'text',
                            'analyzer' => 'ik_max_word',
                        ],
                    ],
                ],
            ],
        ];

        $ik = $this->client->indices()->create($params);
        dd($ik->asArray());
    }
    //判断索引是否存在
    public function isIndex()
    {
        $index = $this->client->indices()->exists(
            ['index' => 'ik']
        )->asBool();
        dd($index);
    }
    //查看索引的的信息
    public function indexInfo()
    {
        $index = $this->client->indices()->getMapping(
            ['index' => 'ik']
        );
        dd($index->asArray());
    }
    //删除索引及数据
    public function indexDelete()
    {
        $index = $this->client->indices()->delete(
            ['index' => 'ik']
        );
        dd($index->asArray());
    }
    //删除索引下面id=1的数据
    public function esDelete()
    {
        $params = [
            'index' => 'ik',
            'id' => 1,
        ];
        $response = $this->client->delete($params);
        dd($response->asArray());
    }
    // 数据插入
    public function esCreateIkData()
    {
        $array = [
            'index' => 'ik',
            'type' => 'doc',
            'id' => 1,
            'body' => [
                'content' => '测试数据',
            ],
        ];
        $result = $this->client->index($array);
        dd($result);
    }
    // 批量插入数据
    public function eaCreateIkDataBulk()
    {

        set_time_limit(0);
        $data = DB::table('faq')->get();
        // 一条一条插入
        foreach ($data as $key => $value) {
            $array['body'][] = ['index' => ['_index' => 'ik', '_id' => $value->id]];
            $array['body'][] = ['content' => $value->content];
        }
        $result = $this->client->bulk($array);
        dd($result);
    }
    // 查询当前索引下有多少数据
    public function esCountData()
    {
        $params = [
            'index' => 'ik',
        ];
        echo $this->client->count($params);
    }
    /**
     * 查询 ik 下面所有数据
     * 默认返回最多10数据
     */
    public function esIkSearch()
    {
        $query = [
            'index' => 'ik',
            // 'id' => 1, // 查询id 的话就加这个字段
        ];
        $result = $this->client->search($query);
        dd($result->asArray());
    }
    /**
     * 查询 ik 下面数据 加各种条件
     *
     */
    public function esIkSearchWhere()
    {
        $query = [
            'index' => 'ik',
            'body' => [
                'query' => [
                    'match' => [
                        'content' => '被骗了几千块钱，有微信怎么要回来'
                    ]
                ]
            ],
            '_source' => ['content'], //目前只有content 如果字段多了 想要那个返回哪个。可以不设置。默认返回所有数据
            'size' => 5, //设置一次返回5条数据、可以不设置
            'from' => 2, //从第几条开始 类似于limit 5,2 可以不设置
        ];
        $result = $this->client->search($query);
        dd($result->asArray());
    }
    /**
     * 修改数据
     * 把 ik 下面 id=1 的content 修改
     */
    public function esIkedit()
    {
        $query = [
            'index' => 'ik',
            'id' => 1,
            'body' => [
                'doc' => [
                    'content' => '修改数据'
                ],
            ],
        ];
        $result = $this->client->update($query);
        dd($result);
    }
}
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
