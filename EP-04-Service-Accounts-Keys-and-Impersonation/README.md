# EP-04: Service Accounts, Keys and Impersonation (IAM, Part 2)

**English** | [عربي](README.ar.md)

In EP-03 we saw that a principal can be a person, a group or a service account, and we left service accounts for later. They deserve a part of their own. You'll meet them in almost every project, and they're behind a lot of security problems, mostly because of the way their keys get handled.

This part covers what a service account is, the different kinds, how your code uses one, why keys are risky, and the safer ways to do the same job, especially impersonation.

## User accounts and service accounts

A person who works on Google Cloud uses a **user account**. It has an email address and a password (and, ideally, MFA), and it gets IAM roles that decide what the person can do.

A **service account** is an account for an app instead of a person. It also has an email address, and it also gets IAM roles, exactly like any other principal. The differences:

- It has no password. It proves who it is with a key, and in most cases Google takes care of that key for you.
- It can't sign in to the console in a browser. Nobody types anything into it; code uses it.
- Its email address looks like `my-app@my-project.iam.gserviceaccount.com`.

Say your app runs on a VM and needs to read files from a bucket and write logs. You don't put your own email and password inside the app. You create a service account for the app, give it the roles it needs, and the app runs as that service account.

![Two cards. The user account has an email like bob@mycompany.com, proves who it is with a password plus MFA, is used by a person and can sign in to the console. The service account has an email like my-app@my-project.iam.gserviceaccount.com, proves who it is with a key, is used by an app or a Google Cloud service and can't sign in to the console. Both get access through IAM roles.](images/user-vs-service-account.drawio.svg)

## Kinds of service accounts

There are two big groups: the ones Google manages and the ones you manage.

### Service agents: managed by Google

Google creates these and uses them to do work in your project on behalf of a service, for example when a Google service needs to reach something of yours to do its job. They're created and managed by Google outside your project, so they don't appear in your list of service accounts, and the IAM page hides their roles unless you tick **Include Google-provided role grants**.

Usually you just leave them alone. Don't remove their roles, or the service they belong to can stop working. Now and then a feature asks you to give a service agent an extra role, and that's about the only time you touch them.

### User-managed service accounts

These are the ones you work with, and they come in two kinds.

**User-created** service accounts are the ones you create yourself. You name them, give them roles, and disable or delete them when they're no longer needed. Each one gets an email address like `NAME@PROJECT_ID.iam.gserviceaccount.com`. Give them clear names, like `orders-api` or `nightly-backup`, so anyone can tell what each one is for. This is the kind you should use.

**Default** service accounts are created automatically by some services when you turn on their API. They're older than user-created ones, but they're still around:

| Service | Default service account |
|---|---|
| Compute Engine | `PROJECT_NUMBER-compute@developer.gserviceaccount.com` |
| App Engine | `PROJECT_ID@appspot.gserviceaccount.com` |

The catch is that, depending on your organization's settings, a default service account may automatically get the **Editor** role on the project. That's far more than any single app needs, and it goes against least privilege. Organizations created after May 3, 2024 have this automatic grant turned off by default. Anywhere else, check: if a default service account has Editor, remove it.

![A tree. Service accounts split into user-managed (you manage them) and Google-managed (service agents). User-managed ones are either user-created, the kind you should use, or default, which are made when you turn on some APIs, may get the Editor role, and are shared by everything that uses them. Service agents are created and run by Google, hidden in the IAM page by default, and you leave their roles alone.](images/service-account-types.drawio.svg)

### Why the default service account is a problem

Say you have three VMs that all use the Compute Engine default service account. They will unless you pick a different one when you create them.

- `vm-1` needs to write files to Cloud Storage and write logs.
- `vm-2` needs to publish messages to Pub/Sub and read data from BigQuery.
- `vm-3` needs to read one secret from Secret Manager.

Because they share one service account, that account needs all of these roles. And now every VM can do all of it: `vm-3` can write to your buckets and read BigQuery, even though it only needs one secret. If any one of the VMs is broken into, the attacker gets everything.

The fix is one service account per VM (or per app), each with only the roles that its VM needs. The default service account still works, and for a quick test it's fine, but for anything real, create your own.

![Left: three VMs share the Compute Engine default service account, which has five roles, so every VM can use all five. Right: each VM has its own service account. vm-1-sa has Storage Object Admin and Logs Writer, vm-2-sa has Pub/Sub Publisher and BigQuery Data Viewer, and vm-3-sa has Secret Manager Secret Accessor, so each VM can use only the roles it needs.](images/default-vs-dedicated.drawio.svg)

When you create a VM you'll also see **access scopes**. They're an older way to limit what a VM can do with its service account: a request only goes through if both the scopes and the IAM roles allow it. Google now recommends a simpler setup: give the VM the full `cloud-platform` scope ("Allow full access to all Cloud APIs"), and control access with the roles of a dedicated service account. That way there's one place to manage access instead of two.

## A service account is a principal and a resource

A service account plays two parts at the same time:

- **As a principal**, it gets roles on other resources, like Storage Object Viewer on a bucket or Cloud SQL Client on a project. That's what lets your app do its work.
- **As a resource**, other principals get roles on it. Somebody has to be allowed to create it, manage its keys, attach it to a VM or impersonate it.

These are the roles you'll see on service accounts:

| Role | What it lets you do with the service account |
|---|---|
| Service Account Admin (`roles/iam.serviceAccountAdmin`) | Create, change, disable and delete it |
| Service Account User (`roles/iam.serviceAccountUser`) | Attach it to a resource, like a VM, so code there runs as it |
| Service Account Token Creator (`roles/iam.serviceAccountTokenCreator`) | Get short-lived tokens for it, which is how impersonation works |
| Service Account Key Admin (`roles/iam.serviceAccountKeyAdmin`) | Create and delete its keys |

Be careful who gets these. Someone who can attach or impersonate a powerful service account can do everything that account can do, even without having those roles themselves.

![The service account sits in the middle. Above it, people have roles on it: Service Account Admin to create and delete it, Service Account User to attach it to a VM, and Token Creator to impersonate it. Below it, the service account has roles on other resources: Storage Object Viewer on Cloud Storage and Cloud SQL Client on Cloud SQL.](images/principal-and-resource.drawio.svg)

## Service account keys

The oldest way to use a service account from your code is with a **key**. You create a key for the service account and download it as a JSON file, and your code or tool loads that file to sign in to Google Cloud as the service account. Keys show up in places like these:

- An app running on a VM, Cloud Run or GKE that reads from Cloud Storage or Cloud SQL.
- CI/CD pipelines and infrastructure-as-code tools like Terraform, which create resources for you.
- Apps running outside Google Cloud: on your own servers, on AWS or on Azure.
- Developers' laptops, so they can test their code against real resources.

### Why keys are risky

A key works like a password, only worse. A user account's password is backed by MFA and Google's sign-in checks, so a stolen password often isn't enough on its own. A service account key has nothing like that: **whoever has the key can act exactly as the service account**, from anywhere. And keys you create don't expire unless you set that up. A key stays valid until someone deletes it.

So if you do use keys, handle them with care.

**Don't:**

- Commit keys to a source code repository. Deleting the file later doesn't help, because it stays in the history. If it happens, delete the key itself in IAM.
- Send keys to other people over email or chat.
- Leave copies lying around in temporary places, like your Downloads folder.
- Keep keys in a public Cloud Storage bucket.
- Count on a secret store like Secret Manager to fix the problem. Google advises against storing keys there: to read the secret, your app already needs an identity that Google Cloud trusts, and that identity could be used directly instead of the key.

**Do:**

- Find and delete keys that nobody uses. Service account insights and the key usage metrics show which ones haven't been used lately.
- Rotate keys regularly: create a new key, switch your app to it, then delete the old one. If an old key leaked, it stops working.
- Set an expiry time, so new keys stop working on their own after a while.
- Turn on the organization policy that disables a key automatically when Google finds it leaked in public, for example pushed to a public GitHub repository.

Google's [best practices for managing service account keys](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys) go through all of this in detail, and it's worth reading slowly.

### Better: don't use keys at all

A key is a long-lived credential, and Google's advice is to use one only when nothing else works. Today there's a safer option for almost every case:

- **Code running on Google Cloud** (a VM, App Engine, Cloud Run): attach the service account to the resource. The Google Cloud client libraries find the credentials on their own (this is called Application Default Credentials), and Google creates and rotates the keys behind the scenes. There's no key file anywhere.
- **Code on GKE**, Google's managed Kubernetes: Workload Identity Federation for GKE lets each workload act as a service account. More on that in the GKE part.
- **Your laptop**: use your own account (`gcloud auth application-default login`), or impersonate a service account, as shown below.
- **CI/CD and Terraform**: impersonation, or Workload Identity Federation if the tool runs outside Google Cloud.
- **Apps outside Google Cloud** (AWS, Azure, your own servers): Workload Identity Federation. Google Cloud trusts the identity the app already has where it runs, and gives it a short-lived token instead of a key. It's often used together with impersonation.

Workload Identity Federation is a big topic of its own, so we won't go deeper into it here.

![Where the code runs, and what to use instead of a key. On Google Cloud (a VM, App Engine or Cloud Run): attach a service account to the resource. On GKE: Workload Identity Federation for GKE. On your laptop: your own user credentials, or impersonation. In CI/CD and Terraform: impersonation, or Workload Identity Federation. Outside Google Cloud: Workload Identity Federation. A key is the last resort.](images/key-alternatives.drawio.svg)

## Service account impersonation

Impersonation means a principal acting as a service account for a short time. Despite the name, there's nothing shady about it. It's the normal, safe way to use a service account's access without a key.

Here's how it works:

1. You create a service account and give it the roles the work needs, say Storage Admin and Cloud SQL Client.
2. You give a user (or another service account) the **Service Account Token Creator** role on that service account.
3. The user signs in as themselves, then asks Google Cloud for an access token for the service account.
4. IAM checks that the user has the Token Creator role on it. If they do, it returns a **short-lived access token**, which lasts one hour at most by default.
5. The user's code, or gcloud, calls Google Cloud with that token, acting as the service account.

When the token expires, the user has to ask again, and IAM checks the role again. Take the role away, and they can't get new tokens. No key is ever created, so there's nothing long-lived to leak. Audit logs also record both identities: the service account, and the person who impersonated it.

![alice@example.com is signed in with gcloud and has the Service Account Token Creator role on storage-sa. Step 1: she asks Google Cloud for a token for storage-sa. Step 2: she gets back a token that lasts up to an hour. Step 3: she calls the Cloud Storage API with that token. storage-sa has the Storage Admin and Cloud SQL Client roles. No key is involved.](images/impersonation.drawio.svg)

### Impersonation with gcloud

An admin gives the user the role on the service account. Here the service account is the resource:

```bash
gcloud iam service-accounts add-iam-policy-binding SA_EMAIL \
    --member="user:USER_EMAIL" \
    --role="roles/iam.serviceAccountTokenCreator"
```

The user signs in as themselves, then adds `--impersonate-service-account` to any gcloud command:

```bash
gcloud auth login
gcloud storage ls --impersonate-service-account=SA_EMAIL
```

gcloud prints a warning that it's using impersonation, then runs the command as the service account. Without the Token Creator role, the command fails with a permission error. Many other tools work the same way; Terraform, for example, can impersonate a service account instead of using a key.

## Try it

EP-05 is a full lab on IAM and impersonation, so this is just a quick first run. You need a project with the Compute Engine API turned on, like the one from EP-02.

Create a service account and give it read-only access to Compute Engine:

```bash
gcloud iam service-accounts create demo-viewer --display-name="Demo viewer"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:demo-viewer@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.viewer"
```

Allow yourself to impersonate it. Even as the project's owner, you need this role:

```bash
gcloud iam service-accounts add-iam-policy-binding \
    demo-viewer@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:YOUR_EMAIL" --role="roles/iam.serviceAccountTokenCreator"
```

IAM changes can take a minute or two to apply. Then run a command as the service account:

```bash
gcloud compute instances list \
    --impersonate-service-account=demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

It works, even if the list is empty. Now try something the service account isn't allowed to do:

```bash
gcloud storage buckets list \
    --impersonate-service-account=demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

This one fails with a permission error. The service account only has Compute Viewer, so while you're acting as it, that's all you get, even though you own the project.

To clean up, remove the role first, then delete the service account. If you delete it first, its old binding stays in the project's policy as a `deleted:serviceAccount:...` entry.

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:demo-viewer@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.viewer"

gcloud iam service-accounts delete demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

## Recap

- A service account is an account for an app or a Google Cloud service. It has an email address and gets IAM roles, but it has no password and can't sign in to the console.
- Service agents are Google's own service accounts, used by its services to work in your project. Leave them alone.
- User-managed service accounts are either user-created (use these) or default ones made by services like Compute Engine. Default ones are shared and may get Editor: remove it, or better, don't use them.
- Use one service account per app or VM, with only the roles it needs.
- A service account is both a principal (it gets roles on resources) and a resource (people get roles on it, like Service Account User or Token Creator).
- A key lets anyone who has it act as the service account, with no MFA, and it doesn't expire by default. Avoid keys. If you must use one, never commit or share it, rotate it, and delete the ones nobody uses.
- Use the alternatives: attach the service account on Google Cloud, Workload Identity Federation on GKE and outside Google Cloud, and your own credentials or impersonation for development and CI/CD.
- Impersonation gives a principal with the Token Creator role a short-lived token for the service account. There's no key, and every use is checked and logged.

## Test yourself

1. How is a service account different from a user account?
2. Who creates service agents, and what should you do with them?
3. Your project's Compute Engine default service account has the Editor role. What should you do?
4. Three VMs share one service account. Why is that a problem?
5. Why is a service account called both a principal and a resource?
6. What can someone do with the Service Account User role on a service account?
7. Why is a leaked service account key more dangerous than a leaked user password?
8. A teammate wants to put a key in the app's Git repository "just for now". What do you tell them?
9. Your app runs on Cloud Run and needs to read from Cloud Storage. How should it sign in?
10. What role does a user need to impersonate a service account, and how long does the token last?

<details>
<summary>Answers</summary>

1. It's for an app, not a person. It has no password: it uses keys or tokens, and it can't sign in to the console. Like a user account, it has an email address and gets IAM roles.
2. Google creates and manages them. Leave them as they are, and don't remove their roles.
3. Remove the Editor role and give it only the roles it needs. Better still, create a service account for each app and use those instead.
4. The account needs every role that any of the VMs needs, so every VM gets all of them. If one VM is broken into, the attacker gets everything.
5. As a principal, it gets roles on other resources. As a resource, other principals get roles on it, like permission to manage it, attach it or impersonate it.
6. Attach it to a resource, like a VM, so code there runs as the service account. That gives them the service account's access indirectly.
7. A user account has MFA and sign-in checks behind it. A key has none, and it doesn't expire by default, so whoever has it can act as the service account from anywhere.
8. Don't. Git keeps it in the history even after the file is deleted. Better not to use a key at all, and if one was committed, delete that key in IAM.
9. Attach a service account to the Cloud Run service, with only the roles it needs. The client libraries pick up the credentials on their own, with no key.
10. Service Account Token Creator on that service account. The token lasts one hour at most by default.

</details>

## Next

EP-05: a hands-on lab on IAM and service account impersonation.

---

<sub>Useful links: [Service accounts overview](https://docs.cloud.google.com/iam/docs/service-account-overview) · [Types of service accounts](https://docs.cloud.google.com/iam/docs/service-account-types) · [Best practices for using service accounts](https://docs.cloud.google.com/iam/docs/best-practices-service-accounts) · [Best practices for managing keys](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys) · [Service account impersonation](https://docs.cloud.google.com/iam/docs/service-account-impersonation) · [Roles for service accounts](https://docs.cloud.google.com/iam/docs/service-account-permissions) · [Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation) · [Application Default Credentials](https://docs.cloud.google.com/docs/authentication/application-default-credentials) · [Service accounts on Compute Engine](https://docs.cloud.google.com/compute/docs/access/service-accounts)</sub>
