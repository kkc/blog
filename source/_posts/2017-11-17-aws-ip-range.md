title: retrieving AWS IP range
date: 2017-11-17 03:39:05
tags: aws
---

# Usage 
http://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html

AWS provides the ip range file, you can download from [ip-ranges.json](https://ip-ranges.amazonaws.com/ip-ranges.json)

```
jq  '.prefixes[] | select(.region=="us-east-1")' < ipranges.json

{
  "ip_prefix": "23.20.0.0/14",
  "region": "us-east-1",
  "service": "AMAZON"
},
{
  "ip_prefix": "50.16.0.0/15",
  "region": "us-east-1",
  "service": "AMAZON"
},
{
  "ip_prefix": "50.19.0.0/16",
  "region": "us-east-1",
  "service": "AMAZON"
},
...
```

can use it to get specific IP range

# Purpose

- Implementing Egress Control
    allow outbound traffic to the CIDR block in the Amazon list

- Implementing Ingress Control
    while using different cloud or different platform, we can use it to define legal traffic

# AWS IP Address Ranges Notifications

strongly recommened to set this up, really helpful
http://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html#subscribe-notifications
