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

1. Add roles
Fresh service account doesn’t have any IAM roles so it doesn’t have permission to do anything. To allow OS Login we have to add these 4 roles to the Ansible service account:

-Compute Instance Admin (beta)
-Compute Instance Admin (v1)
-Compute OS Admin Login
-Service Account User

Here is how to do it via gcloud:

```shell
for role in \
    'roles/compute.instanceAdmin' \
    'roles/compute.instanceAdmin.v1' \
    'roles/compute.osAdminLogin' \
    'roles/iam.serviceAccountUser'
do \
    gcloud projects add-iam-policy-binding \
        my-gcp-project-241123 \
        --member='serviceAccount:ansible-svc@my-gcp-project-12345.iam.gserviceaccount.com' \
        --role="${role}"
done
```

2. Create key for service account and save it

A Service account is useless without key, create one with gcloud:

```shell
$ gcloud iam service-accounts keys create \
    .gcp/gcp-key-ansible-sa.json \
    --iam-account=ansible-svc@my-gcp-project.iam.gserviceaccount.com
```

This will create GCP key, not the SSH key. This key is used for interacting with Google Cloud API – tools like gcloud, gsutil and others are using it. We will need this key for gcloud to add SSH key to the service account.

3. Create SSH key for service account
(This is the easiest part)

```shell
$ ssh-keygen -f ssh-key-ansible-sa
```

4. Add SSH key for OS login to service account

Now, to allow service account to access instances via SSH it has to have SSH key added to it. To do this, first, we have to activate service account in gcloud:

```java
$ gcloud auth activate-service-account \
    --key-file=.gcp/gcp-key-ansible-sa.json
```

This command uses GCP key we’ve created on step 2.

Now we add SSH key to the service account:
```java
$ gcloud compute os-login ssh-keys add \
    --key-file=ssh-key-ansible-sa.pub
```

5. Switch back from service account

```shell
$ gcloud config set account your@hhchealth.org
```

Connecting to the instance with OS login
Now, we have everything configured on the GCP side, we can check that it’s working.

Note, that you don’t need to add SSH key to compute metadata, authentication works via OS login. But this means that you need to know a special user name for the service account.

Find out the service account id:

```shell
$ gcloud iam service-accounts describe \
    ansible-sa@my-gcp-project.iam.gserviceaccount.com \
    --format='value(uniqueId)'
106627723496398399336
```

This id is used to form user name in OS login – it’s `sa_<unique_id>.`

Here is how to use it to check SSH access is working:

```shell
$ ssh -i .ssh/ssh-key-ansible-sa sa_106627723496398399336@10.0.0.44
...
sa_106627723496398399336@instance-1:~$ # Yay!
```

Configuring Ansible
And for the final part – make Ansible work with it.

There is a special variable ansible_user that sets user name for SSH when Ansible connects to the host.

In my case, I have a group gcp where all GCP instances are added, and so I can set ansible_user in group_vars like this:

```shell
# File inventory/dev/group_vars/gcp
ansible_user: sa_106627723496398399336
```

And check it:

```shell
$ ansible -i inventory/dev gcp -m ping
10.0.0.44 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
10.0.0.43 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

And now we have Ansible configured to access GCP instances via OS Login. There is no magic here – just a bit of gluing together a bunch of stuff after reading lots of docs. That’s it for now, till the next time!
