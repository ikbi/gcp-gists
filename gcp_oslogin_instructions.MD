## GCP OS Login Instructions

You 

There are 2 ways that this could be accomplished:

1) Add SSH key to the project metadata 
2) Use OS Login feature. As you can guess I’m using OS Login. You can read about OS Login and its benefits in docs. 

So, how to make Ansible work via OS Login?? In the end, we'll have a service account for Ansible that will be able to SSH connect to instances via OS login.

### Service account

In short, OS Login allows SSH access for IAM users - there is no need to provision Linux users on an instance. So Ansible should have access to the instances via IAM user. This is accomplished via IAM service account.

We can create service account via Console (web UI), via Terraform template or (as in my case) via gcloud:

```shell
$ gcloud iam service-accounts create ansible-sa \
     --display-name "Service account for Ansible"
```

### Configure OS Login
Now, the trickiest part – configuring OS Login for service account. Before you do anything else make sure to enable it for your project:

```shell
$ gcloud compute project-info add-metadata \
    --metadata enable-oslogin=TRUE
```
