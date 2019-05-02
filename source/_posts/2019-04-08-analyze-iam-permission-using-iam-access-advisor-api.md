---
title: 透過 IAM access advisor API 來幫 IAM permission 做大掃除
catalog: true
date: 2019-04-08 17:37:08
subtitle: 終於學會怎麼系統性的清 IAM 
header-img: header.jpg
tags:
  - AWS
  - IAM
---

## Preface

隨著組織慢慢變大，在 AWS 上面常常會遇到一個問題就是，我的 IAM entity 的 permission 是不是開的太大了，這個問題常常發生在 developer 想要快速驗證自己的 application 能不能 work，而作為 admin 的我們有時會給予太大的權限，等到該專案開展到一定程度的時候，其實需要使用到的權限應該是穩定下來了，但又難以找每個專案負責人慢慢 review 權限，這樣一來，其實違反了 least privilege 的原則，也就是只給於需要的權限就好。

## IAM access advisor API

AWS 其實有推出一組用來分析 IAM 權限管理的 API，而 AWS 官方的 [blog](https://aws.amazon.com/blogs/security/automate-analyzing-permissions-using-iam-access-advisor/) 也有幾篇介紹，完全可以符合我們的需求，把一些用不到的權限限縮。

- `generate-service-last-accessed-details` 針對 IAM ser, role, group, or policy 產生最後存取 (last accessed data) 的資訊，呼叫這個 API 後會拿到一組 `JobId`，接著要等待一陣子，才能透過 `get-service-last-accessed-details` 得到資料。
- `get-service-last-accessed-details` 透過這個 API 輸入 JobId 去得到 last accessed 的資料
- `get-service-last-accessed-details-with-entities` 其實跟上面的 API 很類似，只是可以指定 --service-namespaces 去看特定的 service
- `list-policies-granting-service-access` 可以看到這個權限（針對 service) 是從哪個 policy 來的

有了以上這幾組 API 我們就可以實作一個簡單的 script 去掃出是否有權限太大的 IAM entity。 

## Simple Example

這個範例很大一部分是參考 trek10inc 的 [config-excess-access-exorcism](https://github.com/trek10inc/config-excess-access-exorcism/blob/734fecde2f02dd448e0439f366d5400d4413a6d0/IAM_ALLOWS_UNUSED_SERVICES/iam_rule_helpers.py) 來的，不過有做一些簡單的修改，有了這個程式可以幫我們快速定位，那個 IAM role 開的權限太大，而這個 repo 其實想做到的事情更潮，是將其設定為 AWS config 的 rule，由此一來就可以讓 AWS 幫我們定期去掃 IAM entities。

先透過下面這個 function 拿到該 IAM entity 所有的 service 權限，這邊要注意的是要把 paginate 的資料也拿回來，因為有些權限太多需要好幾個 API call 才拿得齊。
```
def get_iam_last_access_details(iam, arn):
    job = iam.generate_service_last_accessed_details(Arn=arn)
    job_id = job['JobId']
    service_results = []
    while True:
        result = iam.get_service_last_accessed_details(JobId=job_id)
        if result['JobStatus'] == 'IN_PROGRESS':
            print("Awaiting job")
            continue
        elif result['JobStatus'] == 'FAILED':
            raise Exception(f"Could not get access information for {arn}")
        else:
            service_results.extend(paginate_access_details(job_id, result))
            break
        time.sleep(5)
    return service_results

def paginate_access_details(job_id, result):
    more_data, marker = result['IsTruncated'], result.get('Marker')
    if not more_data:
        return result['ServicesLastAccessed']

    all_service_info = result['ServicesLastAccessed'][:]
    while more_data:
        page = iam.get_service_last_accessed_details(JobId=job_id, Marker=marker)
        more_data, marker = page['IsTruncated'], page['Marker']
        all_service_info.extend(page['ServicesLastAccessed'])
    return all_service_info
```

來個簡單的測試
```
detail = get_iam_last_access_details(iam, "arn:aws:iam::AWS_ACCOUNT:role/service-role/AmazonEC2RunCommandRoleForManagedInstances")
pprint(detail)
```

Output 會長得像這樣:
```
[   {   'ServiceName': 'Amazon CloudWatch',
        'ServiceNamespace': 'cloudwatch',
        'TotalAuthenticatedEntities': 0},
    {   'ServiceName': 'AWS Directory Service',
        'ServiceNamespace': 'ds',
        'TotalAuthenticatedEntities': 0},
    {   'ServiceName': 'Amazon EC2',
        'ServiceNamespace': 'ec2',
        'TotalAuthenticatedEntities': 0},
    {   'LastAuthenticated': datetime.datetime(2019, 4, 8, 9, 41, tzinfo=tzutc()),
        'LastAuthenticatedEntity': 'arn:aws:iam::774915305292:role/service-role/AmazonEC2RunCommandRoleForManagedInstances',
        'ServiceName': 'Amazon Message Delivery Service',
        'ServiceNamespace': 'ec2messages',
        'TotalAuthenticatedEntities': 1},
        ...
]
```

有了這個 output 我們就可以來開心的來分析啦，主要就是看 `LastAuthenticated` 這個欄位，如果沒有這個欄位就代表根本沒使用過，這個權限就該被剷除，另外也可以檢查是否這個使用的日期是不是在 180 天前，太久沒用也代表可能不需要了。

```
def never_accessed_services_check(iam, arn):
    service_results = get_iam_last_access_details(iam, arn)
    never_accessed = [
        x for x in service_results if 'LastAuthenticated' not in x
    ]
    if len(never_accessed) > 0:
        return (
            'NON_COMPLIANT',
            "Services " + ', '.join(f"'{x['ServiceNamespace']}'" for x in never_accessed) + " have never been accessed",
        )

    return 'COMPLIANT', 'IAM entity has accessed all allowed services'
```

```
def no_access_in_180_days_check(iam, arn):
    import pytz

    service_results = get_iam_last_access_details(iam, arn)

    pp = pprint.PrettyPrinter(indent=4)
    pp.pprint(service_results)

    utc_now = datetime.datetime.utcnow().replace(tzinfo=pytz.UTC)

    older_than_180_days = [
        x for x in service_results
        if 'LastAuthenticated' in x and (utc_now - x['LastAuthenticated']) > datetime.timedelta(days=180)
    ]
    if len(older_than_180_days) > 0:
        return (
            'NON_COMPLIANT',
            "Services " + ', '.join(f"'{x['ServiceNamespace']}'" for x in older_than_180_days) + " have not been accessed in the last 180 days",
        )

    return 'COMPLIANT', 'IAM entity has accessed all allowed services in the last 180 days'
```

在知道是哪個 service 有問題後，還可以用 `aws iam list-policies-granting-service-access --arn arn:aws:iam::AWS_ACCOUNT:role/service-role/AmazonEC2RunCommandRoleForManagedInstances --service-namespaces s3` 去看這個 service 的權限是從哪個 policy 來的。

```
{
    "PoliciesGrantingServiceAccess": [
        {
            "ServiceNamespace": "s3",
            "Policies": [
                {
                    "PolicyName": "AmazonEC2RoleforSSM",
                    "PolicyType": "MANAGED",
                    "PolicyArn": "arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM"
                }
            ]
        }
    ],
    "IsTruncated": false
}
```

sample code 可以用下列的程式碼

```
def get_policies(iam, arn, service_namespace_list):
    policies = []
    result = iam.list_policies_granting_service_access(Arn=arn, ServiceNamespaces=service_namespace_list)
    policies.extend(paginate_policies(arn, service_namespace_list, result))
    return policies

def paginate_policies(arn, service_namespace_list, result):
    more_data, marker = result['IsTruncated'], result.get('Marker')
    if not more_data:
        return result['PoliciesGrantingServiceAccess']

    all_service_info = result['PoliciesGrantingServiceAccess'][:]
    while more_data:
        page = iam.list_policies_granting_service_access(Arn=arn, ServiceNamespaces=service_namespace_list, Marker=marker)
        more_data, marker = page['IsTruncated'], page['Marker']
        all_service_info.extend(page['PoliciesGrantingServiceAccess'])
    return all_service_info
```

就可以找出需要修正的 policy 像是這樣

```
[  
   {  
      'ServiceNamespace':'s3',
      'Policies':[  
         {  
            'PolicyName':'AmazonEC2RoleforSSM',
            'PolicyType':'MANAGED',
            'PolicyArn':'arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM'
         }
      ]
   }
]
```

## 心得

管理 IAM 其實需要相當的心力，透過一些 AWS 的 cli 加上 python boto lib，可以讓我們事倍功半，很推薦大家多試試看這些 API 掃掃看，我也有蠻多意外的發現XD

## Reference

1. https://www.trek10.com/blog/excess-access-exorcism-with-aws-config/
2. [remove unnecessary permissions in your iam policies by using service last accessed data](https://aws.amazon.com/blogs/security/remove-unnecessary-permissions-in-your-iam-policies-by-using-service-last-accessed-data/)
3. [automate-analyzing-permissions-using-iam-access-advisor](https://aws.amazon.com/blogs/security/automate-analyzing-permissions-using-iam-access-advisor/)

4. header image credit<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@jbriscoe?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Jason Briscoe"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Jason Briscoe</span></a>