# I want to keep images with tags `prod` or `dev` in an ECR repository, and delete all others in 60 days after pushing. How can I configure a lifecycle policy?
Images with tags `prod` or `dev` are used for my application and I do not want to delete them but all other images should expire in 60 days.

### Short description:
To accomplish this we need to use the declarative nature of policy rules and their priorities.

### Resolution:
Since ECR lifecycle policies have only one available action - `expire`, you can achieve the goal by creating three rules. The first two rules with a high priority 
will expire images with tags `prod` or `dev` after a long period of time (for example, in 30 years). The last rule with a low priority will expire all other images in 60 days.

```
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Images with prod tag will expire in 30 years",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["prod"],
                "countNumber": 10950,
                "countUnit": "days"
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 2,
            "description": "Images with dev tag will expire in 30 years",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["dev"],
                "countNumber": 10950,
                "countUnit": "days"
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 10,
            "description": "All other images will expire in 60 days",
            "selection": {
                "tagStatus": "any",
                "countNumber": 60,
                "countUnit": "days"
            },
            "action": {
                "type": "expire"
            }
        }
    ]
}
```
The first two rules with a high priority set expiration period for images with tags `prod` or `dev` to 30 years.
The last rule with a low priority will not be able to delete these images because it would violate the first two rules.

