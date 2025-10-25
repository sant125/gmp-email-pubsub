# IAM Policies

## amp-sns-policy.json

Policy que permite o AlertManager publicar no SNS.

Substitua SUBSTITUA_PELO_SEU_TOPIC_ARN pelo ARN do teu tópico SNS.

Uso:
```bash
aws iam create-policy \
  --policy-name AMPAlertManagerSNSPolicy \
  --policy-document file://amp-sns-policy.json
```
