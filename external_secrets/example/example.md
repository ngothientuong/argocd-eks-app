# How to Retrieve AWS Secret via external secrets

## Store secret name "friendslist" in AWS Secret Manager
Store the following in PLAIN TEXT when you create the secret name friendslist

{
  "name": {"first": "Tom", "last": "Anderson"},
  "friends": [
    {"first": "Dale", "last": "Murphy"},
    {"first": "Roger", "last": "Craig"},
    {"first": "Jane", "last": "Murphy"}
  ]
}

## Retrieving secret
- Be sure to be in the namespace of external-secrets

- If you want to store the secret in a different namespace , you'll have to set up the sa in that namespace with irsa role as annotation similarly as the user 'external-secrets' in namespace 'external-secrets'

- Run Kubectl Apply of the following:

    - 1-aws-secret-store-with-sa.yaml

    - 2- retrieve-aws-secret-example.yaml

