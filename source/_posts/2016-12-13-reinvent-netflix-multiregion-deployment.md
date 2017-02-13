title: Note for AWS re:Invent 2016 Multi-Region Delivery Netflix Style (DEV311)
date: 2016-12-13 09:45:32
tags: AWS
---

# Note for AWS re:Invent 2016: Multi-Region Delivery Netflix Style (DEV311)

<iframe width="560" height="315" src="https://www.youtube.com/embed/1HiilTXQo4w" frameborder="0" allowfullscreen></iframe>

- Use spinnaker to deploy
- Chaos monkeys to make sure robotness of system
- automatically canary analysis
- continually delivery is ultimately about making deployments boring
- 4000 deployments per day
- have a blameless post-mortem culture
- don’t deploy multiple region at the same time
- blue/green deployments is your friends
- opposed to rolling deployment
- the traffic is cyclic -> use deployment windows -> affect smaller amount of ppl
   - during the evening hours SPS going to spike
   - SPS => streaming starts per second
- automatically canary analysis
   - compare 2 different version of running software taking the production traffic 
   - use metrics to see if deployment could be applied.
- automation is a no-brainer
- using slack or SMS inform ppl instead of email (not a good real-time communication mechanism)
- manual judgement, human can decide to keep running pipeline or stop according to each stage status.
   - human have a gut
