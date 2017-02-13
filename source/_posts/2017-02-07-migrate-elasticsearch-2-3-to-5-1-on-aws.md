title: migrate elasticsearch 2.3 to 5.1 on AWS
date: 2017-02-07 10:20:28
tags: elasticsearch
---

前陣子看到 Elasticsearch on AWS 終於升級到 5.1，就一直想著要把原本有的 2.3 cluster migrate 到 5.1，而查了一下發現步驟有點小複雜，不過實作過一次就上手了，特地來紀錄一下。

主要是參照下面兩篇文章

1. [http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-version-migration.html](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-version-migration.html)
2. [https://jtran21.gitbooks.io/elasticsearch/content/manual_snapshots.html](https://jtran21.gitbooks.io/elasticsearch/content/manual_snapshots.html)

主要步驟其實就是

1. Manually create snapshot to S3.
2. Create 5.1 cluster
3. Restore Snapshot from S3 to cluster

而建立 snapshot 這步有點小麻煩，先參照這篇 [Working with Manual Index Snapshots](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains.html#es-managedomains-snapshots) 完成幾個必要步驟

1. 建立 S3 bucket for snapshot，例如 `es-index-backups`
2. 建立一個 IAM role 讓你的 elasticsearch cluster 能夠使用這個 role 把備份打上 S3

    在 trust relationship 這邊加上
    ```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "",
          "Effect": "Allow",
          "Principal": {
            "Service": "es.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```

3. 設定 IAM policy 讓這個 role 擁有操作 S3 的權限

    ```
    {
        "Version":"2012-10-17",
        "Statement":[
            {
                "Action":[
                    "s3:ListBucket"
                ],
                "Effect":"Allow",
                "Resource":[
                    "arn:aws:s3:::es-index-backups"
                ]
            },
            {
                "Action":[
                    "s3:GetObject",
                    "s3:PutObject",
                    "s3:DeleteObject",
                    "iam:PassRole"
                ],
                "Effect":"Allow",
                "Resource":[
                    "arn:aws:s3:::es-index-backups/*"
                ]
            }
        ]
    }
    ```

4. 使用提供的一次性 script 建立 elasticsearch snapshot 的點

    ```
    from boto.connection import AWSAuthConnection
    
    class ESConnection(AWSAuthConnection):
    
        def __init__(self, region, **kwargs):
            super(ESConnection, self).__init__(**kwargs)
            self._set_auth_region_name(region)
            self._set_auth_service_name("es")
    
        def _required_auth_capability(self):
            return ['hmac-v4']
    
    if __name__ == "__main__":
    
        client = ESConnection(
                region='us-east-1',
                host='search-weblogs-etrt4mbbu254nsfupy6oiytuz4.us-east-1.es.a9.com',
                aws_access_key_id='my-access-key-id',
                aws_secret_access_key='my-access-key', is_secure=False)
    
        print 'Registering Snapshot Repository'
        resp = client.make_request(method='POST',
                path='/_snapshot/weblogs-index-backups',
                data='{"type": "s3","settings": { "bucket": "es-index-backups","region": "us-east-1","role_arn": "arn:aws:iam::123456789012:role/MyElasticsearchRole"}}')
        body = resp.read()
        print body
    ```

以上步驟完成後就可以用 curl 去產生 snapshot 到 s3 了

### Take manual snapshot

```
curl -s -XPUT "your_es_cluster_host/_snapshot/weblogs-index-backups/snapshot_name" 
```

得到建立資訊
```
curl -s -XGET "your_es_cluster_host/_snapshot/weblogs-index-backups/snapshot_name"
```

```
curl -s -XGET "your_es_cluster_host/_snapshot/weblogs-index-backups/_all"
```

### Restore from snapshot

```
curl -XPOST "your_es_cluster_host/_snapshot/weblogs-index-backups/snapshot_name/_restore"
```
