# EP-05: Hands-on Lab: IAM and Service Account Impersonation

**English** | [عربي](README.ar.md)

EP-03 and EP-04 were mostly ideas: roles and policies, inheritance, service accounts, keys and impersonation. This part puts them to work on a real project. We'll give a teammate access and watch inheritance happen, create a service account key and see how far it can travel, then delete the key and do the same work with an attached service account and with impersonation.

It's a long lab, but every step is small. Do it yourself as you read. You'll remember it far better than if you only read it.

## The plan

Three people take part:

- **Owner**: you. You own the project, so you have the Owner role on it.
- **Developer**: a teammate who joins the project and needs to work with its storage.
- **Stranger**: someone who never signs in to Google Cloud at all. All they'll ever have is a file that leaked.

The lab has three phases:

1. **Roles for the developer.** Give them a role on the project and another on a bucket, see how the two combine, then take the project one away.
2. **A service account key.** Create one, use it in a small app, and watch it work for everyone who has a copy.
3. **No keys.** Delete the key, attach the service account to a VM, and let the developer impersonate it instead.

![On the left, who's who: the owner, who has the Owner role on the project; the developer, a teammate who joins the project; and a stranger, who never signs in to Google Cloud. On the right, the Google Cloud project: a bucket with two files (Phase 1), a VM called lab-vm with no service account at first (Phase 2), and a service account called storage-admin-sa with the Storage Admin role (Phase 2). Along the bottom, the three phases: roles for the developer and how they're inherited; a service account key and everyone it works for; and no keys, using an attached service account and impersonation.](images/lab-setup.drawio.svg)

## What you need

- A project where you're the Owner, like the one from EP-02. This lab assumes the project isn't part of an organization. In an organization, policies may block some of the steps, like creating keys or adding Gmail accounts.
- A second Google account for the developer. Any Gmail account works. Use another browser profile, or a private window, so the two accounts don't mix.
- The gcloud CLI and Python 3 on your machine.

The VM and the bucket cost a few cents, as long as you delete them at the end. The last section shows how.

### One terminal per person

Open three terminals, one for each person. Each one needs its own gcloud sign-in, so give each terminal its own gcloud config folder:

```bash
# in the owner's terminal
export CLOUDSDK_CONFIG=~/gcloud-owner

# in the developer's terminal
export CLOUDSDK_CONFIG=~/gcloud-developer

# in the stranger's terminal
export CLOUDSDK_CONFIG=~/gcloud-stranger
```

gcloud keeps its sign-ins and settings in that folder, and so do the client libraries we'll use later. If you open a new terminal, run its line again. (In PowerShell, it's `$env:CLOUDSDK_CONFIG = "$HOME\gcloud-owner"`.) Three Linux users, or three machines, work just as well. Don't use Cloud Shell for this lab: it signs you in on its own, which would hide part of what we want to see.

In the owner's terminal and in the developer's, sign in and point gcloud at the project. The stranger never signs in.

```bash
gcloud auth login
gcloud config set project PROJECT_ID
```

`gcloud auth login` opens a browser, or gives you a link to open. Make sure you sign in with the account that belongs to that terminal. For the developer, gcloud warns that they don't seem to have access to the project. That's right, they don't yet. Answer `Y`.

Last, make a folder for the lab and install the Python library for Cloud Storage:

```bash
mkdir ~/iam-lab && cd ~/iam-lab
python3 -m venv venv
source venv/bin/activate
pip install google-cloud-storage
```

In the other two terminals, go to the same folder and activate the same environment: `cd ~/iam-lab && source venv/bin/activate`. On Windows, the activate script is `venv\Scripts\activate`.

### Remove an old grant first

As the owner, open **IAM & Admin** > **IAM**. If you turned on the Compute Engine API in EP-02, you'll probably see two principals: you, with the Owner role, and the Compute Engine default service account, `PROJECT_NUMBER-compute@developer.gserviceaccount.com`, with the Editor role. As EP-04 explained, that's far more than it needs. Click the pencil next to it, delete the Editor role and click **Save**.

Now you're the only one with access to the project.

## Phase 1: Roles for the developer

### 1. Create a bucket

Cloud Storage gets its own parts later, from EP-08 to EP-10. For now, a bucket is just something to give access to: a container for files.

1. In the console, go to **Cloud Storage** > **Buckets** and click **Create**.
2. Give it a name. Bucket names are unique across all of Google Cloud, so something like `PROJECT_ID-lab` is a safe choice.
3. For the location type, pick **Region**, and a region near you, for example `europe-west3` (Frankfurt).
4. Leave everything else as it is and click **Create**.
5. A box says that public access will be prevented. Keep **Enforce public access prevention on this bucket** ticked and click **Confirm**. Nobody on the internet should be able to reach these files.
6. Open the bucket and upload two small files. Any files will do.

Or, from the owner's terminal:

```bash
gcloud storage buckets create gs://BUCKET_NAME --location=europe-west3 \
    --uniform-bucket-level-access --public-access-prevention

echo "hello" > file-1.txt
echo "hello again" > file-2.txt
gcloud storage cp file-1.txt file-2.txt gs://BUCKET_NAME
```

### 2. Look as the developer

Open the console as the developer. The project isn't in their project picker, and there's no way to reach it. With no role at all, they can't even see that it exists.

### 3. Give the developer Storage Admin on the project

As the owner:

1. Go to **IAM & Admin** > **IAM** and click **Grant access**.
2. Under **New principals**, type the developer's email.
3. For the role, pick **Cloud Storage** > **Storage Admin**.
4. Click **Save**.

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:DEV_EMAIL" --role="roles/storage.admin"
```

As the developer, refresh the console. The project shows up now, but that doesn't mean they can see everything in it:

- **Compute Engine** says they need more access. Storage Admin only covers Cloud Storage.
- **Cloud Storage** shows the bucket and both files, because the role covers every bucket in the project.

### 4. Add a role on the bucket itself

As the owner, open the bucket and go to its **Permissions** tab. The developer is already listed there with Storage Admin, even though you never touched the bucket. The **Inheritance** column says where it comes from: the project. This is the inheritance from EP-03. A role on the project reaches every resource inside it.

You'll also see a few entries you didn't add, for the project's owners, editors and viewers, with roles called Storage Legacy something. Cloud Storage adds these to every new bucket, so the people who have those basic roles on the project keep access to it. Leave them as they are.

Now give the developer a second role, right on the bucket:

1. Still on the bucket's **Permissions** tab, click **Grant access**.
2. Type the developer's email, pick **Storage Object Viewer** and click **Save**.

```bash
gcloud storage buckets add-iam-policy-binding gs://BUCKET_NAME \
    --member="user:DEV_EMAIL" --role="roles/storage.objectViewer"
```

The developer now has two roles on the bucket: Storage Admin, inherited from the project, and Storage Object Viewer, granted on the bucket. Their effective access is both of them together. Storage Admin already includes everything that Object Viewer does, so nothing changes for them yet.

### 5. Remove the project role

Now take Storage Admin away. Try it from the bucket's **Permissions** tab first: you can't. The role isn't granted on the bucket, it only shows up there because of inheritance. You have to remove it where it was granted, on the project:

1. Go to **IAM & Admin** > **IAM**.
2. Click the pencil next to the developer, delete the Storage Admin role and click **Save**.

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="user:DEV_EMAIL" --role="roles/storage.admin"
```

![Two panels. Both grants: the project's policy gives the developer Storage Admin, which the bucket inherits, and the bucket's own policy gives them Storage Object Viewer. Together, the developer can list the project's buckets, read the files and read the bucket's policy. After removing the project grant: there's nothing on the project to inherit, only Storage Object Viewer on the bucket. Now the developer can read the files, but can't list the project's buckets or read the bucket's policy. An inherited role can only be removed where it was granted: here, on the project.](images/inheritance.drawio.svg)

As the developer, open **Cloud Storage** > **Buckets** again. There's a permission error, and the bucket is gone from the list. Object Viewer lets them read the files in that one bucket, but listing a project's buckets takes a different permission, `storage.buckets.list`, and that one is granted on the project. In the console, they have no way left to click through to the files. In the terminal, it's a different story.

### 6. Check it from the terminal

Run these in both the owner's terminal and the developer's. First, list the project's buckets:

```bash
gcloud storage buckets list
```

The owner gets the bucket and its details. The developer gets a 403 error, saying they don't have `storage.buckets.list` access to the project.

Then list the files in the bucket:

```bash
gcloud storage ls gs://BUCKET_NAME
```

This one works for both. Object Viewer includes `storage.objects.list`, so the developer can list the files. They just need to know the bucket's name.

Last, read the bucket's policy:

```bash
gcloud storage buckets get-iam-policy gs://BUCKET_NAME
```

The owner sees all of it:

```yaml
bindings:
- members:
  - projectEditor:PROJECT_ID
  - projectOwner:PROJECT_ID
  role: roles/storage.legacyBucketOwner
- members:
  - projectViewer:PROJECT_ID
  role: roles/storage.legacyBucketReader
- members:
  - projectEditor:PROJECT_ID
  - projectOwner:PROJECT_ID
  role: roles/storage.legacyObjectOwner
- members:
  - projectViewer:PROJECT_ID
  role: roles/storage.legacyObjectReader
- members:
  - user:DEV_EMAIL
  role: roles/storage.objectViewer
etag: CAI=
```

Those are the legacy entries from step 4, and at the bottom, the developer's Object Viewer role. The developer gets a 403 again, this time for `storage.buckets.getIamPolicy`. Reading a bucket's policy isn't the same as reading its files.

| Command | Owner | Developer |
|---|---|---|
| `gcloud storage buckets list` | works | fails: needs `storage.buckets.list` on the project |
| `gcloud storage ls gs://BUCKET_NAME` | works | works: Object Viewer on the bucket |
| `gcloud storage buckets get-iam-policy gs://BUCKET_NAME` | works | fails: needs `storage.buckets.getIamPolicy` |

That's least privilege in practice: the developer can read the files in one bucket, and nothing more.

## Phase 2: A service account key

### 1. Create a VM with no identity

1. Go to **Compute Engine** > **VM instances** and click **Create instance**.
2. Name it `lab-vm`. Pick the same region as the bucket, and any zone.
3. For the machine type, pick **e2-micro**, a small and cheap one.
4. Under **Security**, find **Identity and API access** and set **Service account** to **No service account**.
5. Click **Create**.

```bash
gcloud compute instances create lab-vm --zone=europe-west3-a \
    --machine-type=e2-micro --no-service-account --no-scopes
```

When it's running, click **SSH** next to it. A terminal opens in the browser, on the VM. Run:

```bash
gcloud auth list
```

It says there are no credentialed accounts. The VM has no identity, so nothing running on it can call a Google Cloud API.

### 2. Create the service account

1. Go to **IAM & Admin** > **Service accounts** and click **Create service account**.
2. Name it `storage-admin-sa` and click **Create and continue**. Its email is `storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com`.
3. Give it the **Storage Admin** role and click **Continue**.
4. Skip the last step for now and click **Done**.

```bash
gcloud iam service-accounts create storage-admin-sa

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
```

### 3. Create a key

1. Click `storage-admin-sa` and open its **Keys** tab.
2. Click **Add key** > **Create new key**, keep **JSON** and click **Create**.

The browser downloads the key file right away. That's the only copy: Google doesn't keep one, and you can't download it again. Move it into the lab folder and name it `sa-key.json`. With gcloud, it's saved straight into the folder you run it from:

```bash
gcloud iam service-accounts keys create sa-key.json \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

If this fails because of an organization policy, your project is in an organization that blocks key creation. Organizations created on or after May 3, 2024 block it by default, and it's a good default. Read the rest of this phase, then carry on from Phase 3.

Open the file and have a look. Shortened, it looks like this:

```json
{
  "type": "service_account",
  "project_id": "PROJECT_ID",
  "private_key_id": "3f9c0d...",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBg...\n-----END PRIVATE KEY-----\n",
  "client_email": "storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com",
  "client_id": "10427...",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

Two fields matter here: `private_key`, the secret itself, and `client_email`, the service account it belongs to. And notice what's missing: nothing in the file says who's allowed to use it. Whoever has the file can.

### 4. A small app that uses the key

Save this as `with-key.py` in the lab folder, with your bucket's name in it:

```python
from google.cloud import storage

KEY_FILE = "sa-key.json"
BUCKET_NAME = "BUCKET_NAME"


def list_files(client):
    print(f"Files in {BUCKET_NAME}:")
    try:
        bucket = client.get_bucket(BUCKET_NAME)
        for blob in bucket.list_blobs():
            print(f"  {blob.name} ({blob.size} bytes)")
    except Exception as e:
        print(f"  error: {e}")


def list_buckets(client):
    print("Buckets in the project:")
    try:
        for bucket in client.list_buckets():
            print(f"  {bucket.name} ({bucket.location})")
    except Exception as e:
        print(f"  error: {e}")


# sign in with the key file
client = storage.Client.from_service_account_json(KEY_FILE)
list_files(client)
list_buckets(client)
```

It does two things: it lists the files in the bucket, and it lists every bucket in the project. Run it in the owner's terminal:

```bash
python3 with-key.py
```

```
Files in BUCKET_NAME:
  file-1.txt (6 bytes)
  file-2.txt (12 bytes)
Buckets in the project:
  BUCKET_NAME (EUROPE-WEST3)
```

It works. And it didn't use your own account at all: everything happened as `storage-admin-sa`, through the key.

### 5. Watch the key travel

**The developer.** The developer asks for the key "just to test something", and you send it. (Here, their terminal already has it, since it's the same folder.) They run the same script, and it works: both lists, everything. Remember that after Phase 1, the developer can only read the files in one bucket. With the key, they can list every bucket in the project, and do anything else that Storage Admin allows, like deleting files, deleting whole buckets or changing who has access to them. The careful roles from Phase 1 don't count anymore.

**The stranger.** Now say the file leaks: someone pastes it in a chat, or commits it to a repository. The stranger runs the script in their terminal. They've never signed in to Google Cloud, and you've never heard of them. It still works.

**The VM.** In the VM's SSH window, click **Upload file** and upload `with-key.py` and `sa-key.json`. Then install the library and run the script:

```bash
sudo apt-get update && sudo apt-get install -y python3-venv
python3 -m venv venv
source venv/bin/activate
pip install google-cloud-storage
python3 with-key.py
```

It works here too, on a VM that has no service account at all.

![A red panel: one key file, copied around. The file sa-key.json is copied to four places: your laptop, the developer who asked for it to test, a stranger who found a leaked copy, and lab-vm, which has no service account. From all four, the code signs in as storage-admin-sa, which has the Storage Admin role on every bucket in the project. Google Cloud only sees storage-admin-sa. It can't tell who is holding the key.](images/key-everywhere.drawio.svg)

Four places, and Google Cloud sees the same identity every time: `storage-admin-sa`. It can't tell a teammate from a stranger. This is what EP-04 meant when it said a key works like a password, only worse.

## Phase 3: The same work, without keys

### 1. Delete the key

1. Open `storage-admin-sa` and go to its **Keys** tab.
2. Click the trash icon next to the key and confirm.

```bash
# find the key's ID
gcloud iam service-accounts keys list --managed-by=user \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com

gcloud iam service-accounts keys delete KEY_ID \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

Every copy of `sa-key.json` is useless now, wherever it is. Run `with-key.py` anywhere, and it fails. (Tokens that were already issued with the key keep working until they expire, which takes an hour at most.)

### 2. The app without a key

Save a copy of the script as `without-key.py`. Delete the `KEY_FILE` line, and change the line that creates the client to this:

```python
client = storage.Client()
```

There's no file name anywhere now. The library finds credentials on its own, using **Application Default Credentials** (ADC). ADC looks in three places, in this order:

1. The `GOOGLE_APPLICATION_CREDENTIALS` environment variable, if it points to a credential file.
2. The file that `gcloud auth application-default login` creates.
3. The service account attached to the resource that the code runs on. The code gets it from the metadata server.

If it finds nothing, it fails with `DefaultCredentialsError`.

![The code says client = storage.Client(), with no key file. ADC looks in order. First, the GOOGLE_APPLICATION_CREDENTIALS environment variable, which points to a file; it isn't used in this lab. If that's not set, the file saved by gcloud auth application-default login, which the owner and the developer use on their laptops. If there's no file, the attached service account, which the code gets from the metadata server; lab-vm uses this once storage-admin-sa is attached. If nothing is attached, it fails with DefaultCredentialsError.](images/adc-order.drawio.svg)

That's why the same code can run on your laptop (place 2) and on a VM (place 3) without any changes.

### 3. Attach the service account to the VM

You can't change a VM's service account while it's running, so stop it first:

1. In **VM instances**, select `lab-vm` and click **Stop**.
2. When it has stopped, open it and click **Edit**.
3. Under **Security**, in **Identity and API access**, set **Service account** to `storage-admin-sa`.
4. Set **Access scopes** to **Allow full access to all Cloud APIs**. As EP-04 explained, the service account's roles are what limit access.
5. Click **Save**, then start the VM again.

```bash
gcloud compute instances stop lab-vm --zone=europe-west3-a

gcloud compute instances set-service-account lab-vm --zone=europe-west3-a \
    --service-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com \
    --scopes=cloud-platform

gcloud compute instances start lab-vm --zone=europe-west3-a
```

To attach a service account, you need the Service Account User role on it. As the project's Owner, you have that already.

SSH into the VM again. `gcloud auth list` now shows `storage-admin-sa` as the active account. Upload `without-key.py`, delete the old key file, and run the script:

```bash
rm sa-key.json
source venv/bin/activate
python3 without-key.py
```

Both lists come back, and there's no key on the VM. Google Cloud gives the VM short-lived tokens for the service account, and replaces them on its own.

### 4. The owner, on their laptop

In the owner's terminal, run the new script:

```bash
python3 without-key.py
```

It fails:

```
google.auth.exceptions.DefaultCredentialsError: Your default credentials were not found.
```

But you signed in to gcloud, didn't you? You did, but `gcloud auth login` only signs in gcloud itself. Client libraries don't use those credentials. They look for ADC, and there isn't any yet. Create it:

```bash
gcloud auth application-default login
```

Sign in again when the browser opens. gcloud saves the credentials in a file called `application_default_credentials.json`, in its config folder. That's `~/.config/gcloud` normally, or the folder in `CLOUDSDK_CONFIG`. Run the script again, and it works, this time as you: your own account, with your own roles. You may see a warning about a quota project. You can ignore it in this lab.

### 5. The developer, as themselves

The developer does the same:

```bash
gcloud auth application-default login
python3 without-key.py
```

This time, both parts fail with 403 errors. The developer doesn't have `storage.buckets.get`, which `get_bucket` needs, or `storage.buckets.list`. That's correct: their only role is Object Viewer on the bucket.

But the developer really does need to test the app with the service account's access. That's what impersonation is for.

### 6. The developer impersonates the service account

```bash
gcloud auth application-default login \
    --impersonate-service-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

After signing in, the ADC file looks different:

```json
{
  "delegates": [],
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com:generateAccessToken",
  "source_credentials": {
    "client_id": "...",
    "client_secret": "...",
    "refresh_token": "...",
    "type": "authorized_user"
  },
  "type": "impersonated_service_account"
}
```

Its type is now `impersonated_service_account`. The `source_credentials` are the developer's own, and the URL is where the library asks for a token for the service account. There's no key in it.

Run the script again:

```bash
python3 without-key.py
```

Another error, but a different one:

```
error: ('Unable to acquire impersonated credentials', ... "code": 403,
  "message": "Permission 'iam.serviceAccounts.getAccessToken' denied on resource (or it may not exist)." ...
```

The developer asked for a token, and IAM said no: they don't have the Token Creator role on `storage-admin-sa`.

### 7. The owner allows it

In the owner's terminal, give the developer Token Creator on the service account:

```bash
gcloud iam service-accounts add-iam-policy-binding \
    storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:DEV_EMAIL" \
    --role="roles/iam.serviceAccountTokenCreator"
```

In the console, it's on the service account's page: open `storage-admin-sa`, go to its **Permissions** tab, click **Grant access**, and give the developer **Service Account Token Creator**.

Notice where the role goes: on the service account, not on the project. On the project, it would let the developer impersonate every service account in it.

Wait a minute or two for the change to apply. Then the developer runs the script again:

```bash
python3 without-key.py
```

Files and buckets, as `storage-admin-sa`, with no key anywhere. If you get an error saying that the IAM Service Account Credentials API is disabled, turn it on with `gcloud services enable iamcredentials.googleapis.com` and try again.

### 8. And the stranger?

There's nothing for them to use. The key is gone. They've never signed in, and even if they signed in with a Google account of their own, they wouldn't have Token Creator on the service account.

![A green panel: no key file anywhere. lab-vm, which has storage-admin-sa attached, gets a token from the metadata server. The developer, who has Token Creator on storage-admin-sa, gets a token that lasts up to an hour. Both act as storage-admin-sa, with the Storage Admin role, on the bucket. The stranger has no key, no role and isn't signed in, so there's nothing to use. Tokens are short-lived, and only principals that IAM allows can get one.](images/no-keys.drawio.svg)

One thing to keep in mind: the developer's ADC file isn't harmless either. It holds their own refresh token, so it should stay private, like any credential. The difference is that it's tied to the developer. They can revoke it, and once you take away their Token Creator role, it can't reach the service account anymore.

## Clean up

As the owner, delete what the lab created:

```bash
gcloud compute instances delete lab-vm --zone=europe-west3-a

# deletes the files, then the bucket
gcloud storage rm --recursive gs://BUCKET_NAME

# remove the role first, then the service account (see EP-04)
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
gcloud iam service-accounts delete storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

The developer's role on the bucket goes away with the bucket, and their Token Creator role goes away with the service account. Then sign the terminals out and remove the lab's folders:

```bash
# in the owner's and the developer's terminals
gcloud auth application-default revoke
gcloud auth revoke

# then, in any terminal
rm -rf ~/iam-lab ~/gcloud-owner ~/gcloud-developer ~/gcloud-stranger
```

## Recap

- A role on the project is inherited by everything inside it. On the resource, it shows up as inherited, and the effective access is all the roles together.
- An inherited role can only be removed where it was granted.
- Roles can be precise. Object Viewer on one bucket lets someone read its files, but not list the project's buckets or read the bucket's policy. Listing buckets is a permission on the project.
- A service account key works for anyone who has it: a teammate with fewer roles, a stranger who never signed in, a VM with no service account. Google Cloud only sees the service account.
- Deleting the key makes every copy useless at once.
- Client libraries find credentials with ADC: the `GOOGLE_APPLICATION_CREDENTIALS` variable first, then the file from `gcloud auth application-default login`, then the attached service account.
- On a VM, attach the service account. You have to stop the VM first. There's no key, just short-lived tokens.
- On a laptop, `gcloud auth application-default login` uses your own account. Add `--impersonate-service-account` to act as a service account; for that, you need Token Creator on it.

## Test yourself

1. You gave someone a role on the project. Why does it show up on the bucket's Permissions tab?
2. The bucket's Permissions tab won't let you remove that role. Why not, and where do you remove it?
3. The developer only has Storage Object Viewer on one bucket. Can they list the project's buckets? List the files in that bucket? Read the bucket's policy?
4. Why does the console's Buckets page give the developer an error, even though they can read the files?
5. The developer runs the app with the service account's key. What can they do now, and why?
6. A VM has no service account, but there's a key file on it. Can code on the VM reach Cloud Storage?
7. What happens to all the copies of a key when you delete the key?
8. Your code calls `storage.Client()` with no key file. Where does it look for credentials, and in what order?
9. You signed in with `gcloud auth login`, but your Python app says it can't find default credentials. Why, and how do you fix it?
10. Impersonation fails with `iam.serviceAccounts.getAccessToken` denied. What's missing, and where should it be granted?
11. What do you have to do before you can change a VM's service account?

<details>
<summary>Answers</summary>

1. Roles are inherited down the hierarchy. A role on the project applies to every resource inside it, so the bucket shows it too, marked as inherited from the project.
2. The role isn't granted on the bucket; the bucket only inherits it. Remove it on the project, in **IAM & Admin** > **IAM**.
3. No, yes and no. Listing buckets needs `storage.buckets.list` on the project, and reading the policy needs `storage.buckets.getIamPolicy`. Object Viewer only lets them list and read the files.
4. The Buckets page lists the project's buckets, which needs `storage.buckets.list` on the project. Without it, the console can't show the bucket to click into. gcloud can still list the files if you give it the bucket's name.
5. Everything the service account can do, which here is Storage Admin on every bucket in the project. With the key, they are the service account, and their own roles don't matter.
6. Yes. The code signs in with the key, so it doesn't need the VM to have a service account.
7. They all stop working, wherever they are. Tokens that were already issued with the key last until they expire, an hour at most.
8. Through ADC: first the `GOOGLE_APPLICATION_CREDENTIALS` variable, then the file from `gcloud auth application-default login`, then the service account attached to the resource, from the metadata server.
9. `gcloud auth login` only signs in gcloud. Client libraries look for ADC, so run `gcloud auth application-default login`.
10. The Service Account Token Creator role, granted to the user on that service account, not on the whole project.
11. Stop it. You can change a VM's service account and access scopes only while it's stopped.

</details>

## Next

EP-06: Compute Engine, Google Cloud's virtual machines.

---

<sub>Useful links: [IAM for Cloud Storage](https://docs.cloud.google.com/storage/docs/access-control/iam) · [IAM roles for Cloud Storage](https://docs.cloud.google.com/storage/docs/access-control/iam-roles) · [Create and delete service account keys](https://docs.cloud.google.com/iam/docs/keys-create-delete) · [How Application Default Credentials works](https://docs.cloud.google.com/docs/authentication/application-default-credentials) · [Use service account impersonation](https://docs.cloud.google.com/docs/authentication/use-service-account-impersonation) · [Change a VM's service account](https://docs.cloud.google.com/compute/docs/instances/change-service-account) · [Secure-by-default organizations](https://docs.cloud.google.com/resource-manager/docs/secure-by-default-organizations)</sub>
