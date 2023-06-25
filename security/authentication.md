# Authentication

In kubernetes every call to API Server is needs to be authenticated. There are mulitple ways to implement authentication.

## Static Password File
In this type of authentication we store the file in a csv file with a format

```
user-details.csv

password123,user1,u0001
password123,user2,u0001
```

We can then pass the file name when running the kube-apiserver with a flag `--basic-auth-file=user-details.csv`

## Static Token File
This is similar as static password file, but instead of passowrd it has Tokens

```
user-token-details-csv

ksaufa03f1012,user10,u0001,group1
```
We can then pass the file name when running the kube-apiserver with a flag `--token-auth-file=user-token-details.csv`