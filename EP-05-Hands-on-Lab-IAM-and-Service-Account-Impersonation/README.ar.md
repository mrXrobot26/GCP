<div dir="rtl" lang="ar">

# الـ Lab العملي للـ IAM والـ Service Account Impersonation، الحلقة (EP-05)

**عربي** | [English](README.md)

الـ EP-03 والـ EP-04 كانوا أفكار في الأغلب: الـ roles والـ policies، والـ inheritance، والـ service accounts، والـ keys والـ impersonation. الجزء ده بيطبق الكلام ده على project حقيقي. هندي زميل access ونشوف الـ inheritance وهي بتحصل قدامنا، وهنعمل key لـ service account ونشوف ممكن يوصل لفين، وبعدين هنمسح الـ key ونعمل نفس الشغل بـ service account مربوط بـ VM، وبالـ impersonation.

الـ lab طويل، بس كل خطوة فيه صغيرة. اعمله بنفسك وانت بتقرا. هتفتكره أحسن بكتير من لو قريته بس.

## الخطة

فيه تلات أشخاص في الـ lab ده:

- الـ **Owner**: ده انت. انت صاحب الـ project، فعندك الـ role بتاعة Owner عليه.
- الـ **Developer**: زميل بينضم للـ project، ومحتاج يشتغل على الـ storage بتاعه.
- الـ **Stranger**: واحد غريب عمره ما بيعمل sign in على Google Cloud خالص. وكل اللي هيبقى معاه هو ملف اتسرب.

والـ lab ليه تلات مراحل:

1. **الـ roles بتاعة الـ developer.** اديله role على الـ project و role تانية على bucket، وشوف الاتنين بيتجمعوا إزاي، وبعدين شيل بتاعة الـ project.
2. **الـ service account key.** اعمل واحد، واستخدمه في تطبيق صغير، وشوفه وهو بيشتغل مع أي حد معاه نسخة منه.
3. **من غير keys.** امسح الـ key، واربط الـ service account بـ VM، وخلي الـ developer يعمله impersonate بداله.

![على الشمال، مين مين: الـ owner، وعنده الـ role بتاعة Owner على الـ project، والـ developer، زميل بينضم للـ project، والـ stranger، اللي عمره ما بيعمل sign in على Google Cloud. وعلى اليمين، الـ project على Google Cloud: bucket فيه ملفين (المرحلة الأولى)، و VM اسمها lab-vm مفيش عليها service account في الأول (المرحلة التانية)، و service account اسمه storage-admin-sa وعليه الـ role بتاعة Storage Admin (المرحلة التانية). وتحت، التلات مراحل: الـ roles بتاعة الـ developer وإزاي بتتورث، والـ service account key وكل اللي يقدر يستخدمه، ومن غير keys، بـ service account مربوط وبالـ impersonation.](images/lab-setup.drawio.svg)

## محتاج إيه

- أي project انت الـ Owner بتاعه، زي اللي عملناه في EP-02. والـ lab ده معمول على إن الـ project مش جوه organization. لو جوه organization، ممكن الـ policies تمنع شوية خطوات، زي إنك تعمل keys أو تضيف Gmail accounts.
- إيميل Google تاني للـ developer. أي Gmail account ينفع. استخدم browser profile تاني، أو private window، عشان الاتنين مايتلخبطوش في بعض.
- الـ gcloud CLI و Python 3 على جهازك.

الـ VM والـ bucket هيكلفوك كام سنت، طالما هتمسحهم في الآخر. وآخر جزء بيوريك إزاي.

### كل شخص وليه terminal

افتح تلات terminals، واحد لكل شخص. كل واحد فيهم محتاج sign in لوحده بالـ gcloud، فادي كل terminal فولدر خاص بيه لإعدادات الـ gcloud. في الـ terminal بتاع الـ owner:

<div dir="ltr">

```bash
export CLOUDSDK_CONFIG=~/gcloud-owner
```

</div>

وفي بتاع الـ developer:

<div dir="ltr">

```bash
export CLOUDSDK_CONFIG=~/gcloud-developer
```

</div>

وفي بتاع الـ stranger:

<div dir="ltr">

```bash
export CLOUDSDK_CONFIG=~/gcloud-stranger
```

</div>

الـ gcloud بيحفظ الـ sign-ins والإعدادات بتاعته في الفولدر ده، والـ client libraries اللي هنستخدمها بعدين برضه. ولو فتحت terminal جديد، شغّل السطر بتاعه تاني. (وفي PowerShell بيبقى `$env:CLOUDSDK_CONFIG = "$HOME\gcloud-owner"`.) وتلات Linux users، أو تلات أجهزة، هيشتغلوا بنفس الطريقة. وماتستخدمش Cloud Shell في الـ lab ده: لأنه بيعملك sign in لوحده، وده هيخبي جزء من اللي عاوزين نشوفه.

في الـ terminal بتاع الـ owner وبتاع الـ developer، اعمل sign in وخلي الـ gcloud يشاور على الـ project. والـ stranger عمره ما بيعمل sign in.

<div dir="ltr">

```bash
gcloud auth login
gcloud config set project PROJECT_ID
```

</div>

الـ `gcloud auth login` بيفتح browser، أو بيديك link تفتحه. اتأكد إنك بتعمل sign in بالـ account اللي تبع الـ terminal ده. وفي حالة الـ developer، الـ gcloud هيقولك إن شكله مالوش access على الـ project. وده صح، لسه مالوش. جاوب بـ `Y`.

وفي الآخر، اعمل فولدر للـ lab ونزّل الـ Python library بتاعة Cloud Storage:

<div dir="ltr">

```bash
mkdir ~/iam-lab && cd ~/iam-lab
python3 -m venv venv
source venv/bin/activate
pip install google-cloud-storage
```

</div>

وفي الـ terminals التانيين، روح على نفس الفولدر وشغّل نفس الـ environment بالـ command ده: `cd ~/iam-lab && source venv/bin/activate`. وعلى Windows، الـ activate script بيبقى `venv\Scripts\activate`.

### شيل grant قديم الأول

كـ owner، افتح **IAM & Admin** > **IAM**. لو كنت شغّلت الـ Compute Engine API في EP-02، غالبًا هتلاقي اتنين principals: انت، ومعاك الـ role بتاعة Owner، والـ default service account بتاع Compute Engine، يعني `PROJECT_NUMBER-compute@developer.gserviceaccount.com`، ومعاه الـ role بتاعة Editor. وزي ما شرحنا في EP-04، ده أكتر بكتير من اللي محتاجه. دوس على القلم اللي جنبه، وامسح الـ role بتاعة Editor ودوس **Save**.

دلوقتي انت الوحيد اللي ليه access على الـ project.

## المرحلة الأولى (Phase 1): الـ roles بتاعة الـ developer

### 1. اعمل bucket

الـ Cloud Storage ليه أجزاء خاصة بيه بعدين، من EP-08 لـ EP-10. ودلوقتي، الـ bucket مجرد حاجة هنديها access: مكان بنحط فيه ملفات.

1. في الـ console، روح على **Cloud Storage** > **Buckets** ودوس **Create**.
2. اديله اسم. أسامي الـ buckets لازم تبقى unique على Google Cloud كلها، فاسم زي `PROJECT_ID-lab` اختيار مضمون.
3. في الـ location type، اختار **Region**، واختار region قريبة منك، مثلًا `europe-west3` (فرانكفورت).
4. سيب كل حاجة تانية زي ما هي ودوس **Create**.
5. هيطلعلك box بيقول إن الـ public access هيتمنع. سيب **Enforce public access prevention on this bucket** متعلّم عليه ودوس **Confirm**. مفيش أي حد على الإنترنت المفروض يوصل للملفات دي.
6. افتح الـ bucket وارفع عليه ملفين صغيرين. أي ملفات تنفع.

أو من الـ terminal بتاع الـ owner:

<div dir="ltr">

```bash
gcloud storage buckets create gs://BUCKET_NAME --location=europe-west3 \
    --uniform-bucket-level-access --public-access-prevention

echo "hello" > file-1.txt
echo "hello again" > file-2.txt
gcloud storage cp file-1.txt file-2.txt gs://BUCKET_NAME
```

</div>

### 2. بص بعين الـ developer

افتح الـ console كـ developer. الـ project مش موجود في الـ project picker بتاعه، ومفيش أي طريقة يوصله بيها. ولأنه مالوش أي role، مش قادر حتى يشوف إن الـ project موجود أصلًا.

### 3. ادي الـ developer الـ role بتاعة Storage Admin على الـ project

كـ owner:

1. روح على **IAM & Admin** > **IAM** ودوس **Grant access**.
2. في **New principals**، اكتب إيميل الـ developer.
3. في الـ role، اختار **Cloud Storage** > **Storage Admin**.
4. دوس **Save**.

<div dir="ltr">

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:DEV_EMAIL" --role="roles/storage.admin"
```

</div>

كـ developer، اعمل refresh للـ console. الـ project ظهر دلوقتي، بس ده مش معناه إنه شايف كل حاجة جواه:

- الـ **Compute Engine** بيقوله إنه محتاج access أكتر. الـ Storage Admin بتغطي Cloud Storage بس.
- والـ **Cloud Storage** بيوريه الـ bucket والملفين، عشان الـ role دي بتغطي كل الـ buckets اللي في الـ project.

### 4. ضيف role على الـ bucket نفسه

كـ owner، افتح الـ bucket وروح على تاب **Permissions** بتاعه. هتلاقي الـ developer موجود فيه بالفعل بالـ Storage Admin، مع إنك عمرك ما لمست الـ bucket. وعمود **Inheritance** بيقولك هي جاية منين: من الـ project. ودي الـ inheritance اللي شفناها في EP-03: الـ role اللي على الـ project بتوصل لكل resource جواه.

وهتلاقي كمان كام entry انت ماضفتهمش، للـ owners والـ editors والـ viewers بتوع الـ project، وعليهم roles اسمها Storage Legacy حاجة. الـ Cloud Storage بيضيف دول لأي bucket جديد، عشان اللي عندهم الـ basic roles دي على الـ project يفضل ليهم access عليه. سيبهم زي ما هما.

دلوقتي ادي الـ developer role تانية، على الـ bucket نفسه:

1. وانت لسه في تاب **Permissions** بتاع الـ bucket، دوس **Grant access**.
2. اكتب إيميل الـ developer، واختار **Storage Object Viewer** ودوس **Save**.

<div dir="ltr">

```bash
gcloud storage buckets add-iam-policy-binding gs://BUCKET_NAME \
    --member="user:DEV_EMAIL" --role="roles/storage.objectViewer"
```

</div>

دلوقتي الـ developer عنده اتنين roles على الـ bucket: الـ Storage Admin، متورثة من الـ project، والـ Storage Object Viewer، متدية على الـ bucket. والـ access الفعلي بتاعه هو الاتنين مع بعض. والـ Storage Admin فيها أصلًا كل اللي في الـ Object Viewer، فمفيش حاجة هتتغير بالنسباله لحد دلوقتي.

### 5. شيل الـ role اللي على الـ project

دلوقتي خد منه الـ Storage Admin. جرب الأول من تاب **Permissions** بتاع الـ bucket: مش هينفع. الـ role مش متدية على الـ bucket، هي بس بتظهر هناك بسبب الـ inheritance. لازم تشيلها من المكان اللي اتدت فيه، يعني من على الـ project:

1. روح على **IAM & Admin** > **IAM**.
2. دوس على القلم اللي جنب الـ developer، وامسح الـ role بتاعة Storage Admin ودوس **Save**.

<div dir="ltr">

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="user:DEV_EMAIL" --role="roles/storage.admin"
```

</div>

![لوحتين. اللي على الشمال فيها الـ grants الاتنين: الـ policy بتاعة الـ project بتدي الـ developer الـ Storage Admin، والـ bucket بيورثها، والـ policy بتاعة الـ bucket نفسه بتديله Storage Object Viewer. ومع بعض، الـ developer يقدر يعمل list للـ buckets اللي في الـ project، ويقرا الملفات، ويقرا الـ policy بتاعة الـ bucket. واللي على اليمين بعد ما شلنا الـ grant اللي على الـ project: مفيش حاجة على الـ project تتورث، وفاضل Storage Object Viewer على الـ bucket بس. ودلوقتي الـ developer يقدر يقرا الملفات، بس مايقدرش يعمل list للـ buckets ولا يقرا الـ policy بتاعة الـ bucket. والـ role المتورثة بتتشال بس من المكان اللي اتدت فيه: هنا، على الـ project.](images/inheritance.drawio.svg)

كـ developer، افتح **Cloud Storage** > **Buckets** تاني. هتلاقي permission error، والـ bucket اختفى من اللستة. الـ Object Viewer بتخليه يقرا الملفات اللي في الـ bucket ده بس، لكن إنك تعمل list للـ buckets اللي في project محتاج permission تانية، هي `storage.buckets.list`، ودي بتتدي على الـ project. فمن الـ console، مبقاش عنده أي طريقة يوصل بيها للملفات. لكن من الـ terminal، الحكاية مختلفة.

### 6. اتأكد من الـ terminal

شغّل دول في الـ terminal بتاع الـ owner وبتاع الـ developer. الأول، اعمل list للـ buckets اللي في الـ project:

<div dir="ltr">

```bash
gcloud storage buckets list
```

</div>

الـ owner هيلاقي الـ bucket والتفاصيل بتاعته. والـ developer هياخد 403 error، بيقول إنه مالوش access بـ `storage.buckets.list` على الـ project.

بعدين اعمل list للملفات اللي في الـ bucket:

<div dir="ltr">

```bash
gcloud storage ls gs://BUCKET_NAME
```

</div>

دي هتشتغل مع الاتنين. الـ Object Viewer فيها `storage.objects.list`، فالـ developer يقدر يعمل list للملفات. هو بس محتاج يعرف اسم الـ bucket.

وفي الآخر، اقرا الـ policy بتاعة الـ bucket:

<div dir="ltr">

```bash
gcloud storage buckets get-iam-policy gs://BUCKET_NAME
```

</div>

الـ owner هيشوفها كلها:

<div dir="ltr">

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

</div>

دي الـ legacy entries اللي اتكلمنا عنها في الخطوة 4، وفي الآخر خالص، الـ role بتاعة Object Viewer اللي اديناها للـ developer. والـ developer هياخد 403 تاني، المرة دي عشان `storage.buckets.getIamPolicy`. إنك تقرا الـ policy بتاعة bucket مش زي إنك تقرا الملفات اللي فيه.

<table dir="rtl">
<thead>
<tr><th align="right" style="text-align:right">الـ Command</th><th align="right" style="text-align:right">الـ Owner</th><th align="right" style="text-align:right">الـ Developer</th></tr>
</thead>
<tbody>
<tr><td><code>gcloud storage buckets list</code></td><td>بيشتغل</td><td>بيفشل: محتاج <code>storage.buckets.list</code> على الـ project</td></tr>
<tr><td><code>gcloud storage ls gs://BUCKET_NAME</code></td><td>بيشتغل</td><td>بيشتغل: عنده Object Viewer على الـ bucket</td></tr>
<tr><td><code>gcloud storage buckets get-iam-policy gs://BUCKET_NAME</code></td><td>بيشتغل</td><td>بيفشل: محتاج <code>storage.buckets.getIamPolicy</code></td></tr>
</tbody>
</table>

ودي الـ least privilege على أرض الواقع: الـ developer يقدر يقرا الملفات اللي في bucket واحد، ومفيش أي حاجة تانية.

## المرحلة التانية (Phase 2): الـ service account key

### 1. اعمل VM مالهاش identity

1. روح على **Compute Engine** > **VM instances** ودوس **Create instance**.
2. سميها `lab-vm`. واختار نفس الـ region بتاعة الـ bucket، وأي zone.
3. في الـ machine type، اختار **e2-micro**، وهي صغيرة ورخيصة.
4. في **Security**، دوّر على **Identity and API access** وخلي الـ **Service account** على **No service account**.
5. دوس **Create**.

<div dir="ltr">

```bash
gcloud compute instances create lab-vm --zone=europe-west3-a \
    --machine-type=e2-micro --no-service-account --no-scopes
```

</div>

لما تشتغل، دوس **SSH** اللي جنبها. هيفتحلك terminal في الـ browser، على الـ VM. شغّل:

<div dir="ltr">

```bash
gcloud auth list
```

</div>

هيقولك إن مفيش credentialed accounts. الـ VM مالهاش identity، فأي حاجة شغالة عليها مش هتقدر تكلم أي Google Cloud API.

### 2. اعمل الـ service account

1. روح على **IAM & Admin** > **Service accounts** ودوس **Create service account**.
2. سميه `storage-admin-sa` ودوس **Create and continue**. والإيميل بتاعه هيبقى `storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com`.
3. اديله الـ role بتاعة **Storage Admin** ودوس **Continue**.
4. اعمل skip لآخر خطوة دلوقتي ودوس **Done**.

<div dir="ltr">

```bash
gcloud iam service-accounts create storage-admin-sa

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
```

</div>

### 3. اعمل key

1. دوس على `storage-admin-sa` وافتح تاب **Keys** بتاعه.
2. دوس **Add key** > **Create new key**، وسيب **JSON** زي ما هو ودوس **Create**.

الـ browser هينزّل ملف الـ key على طول. ودي النسخة الوحيدة: Google مش بتحتفظ بنسخة، ومش هتقدر تنزّله تاني. انقله لفولدر الـ lab وسميه `sa-key.json`. ولو عملته بالـ gcloud، بيتحفظ على طول في الفولدر اللي انت شغّال منه:

<div dir="ltr">

```bash
gcloud iam service-accounts keys create sa-key.json \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

</div>

لو ده فشل بسبب organization policy، يبقى الـ project بتاعك جوه organization بتمنع إنك تعمل keys. والـ organizations اللي اتعملت في 3 مايو 2024 أو بعده بتمنعها by default، ودي default كويسة. اقرا باقي المرحلة دي، وبعدين كمّل من المرحلة التالتة.

افتح الملف وبص عليه. بعد ما نختصره، شكله كده:

<div dir="ltr">

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

</div>

فيه حقلين مهمين هنا: الـ `private_key`، وده الـ secret نفسه، والـ `client_email`، وده الـ service account اللي الـ key تبعه. وخد بالك من الحاجة اللي مش موجودة: مفيش أي حاجة في الملف بتقول مين مسموح له يستخدمه. أي حد معاه الملف يقدر.

### 4. تطبيق صغير بيستخدم الـ key

احفظ ده باسم `with-key.py` في فولدر الـ lab، وحط فيه اسم الـ bucket بتاعك:

<div dir="ltr">

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


client = storage.Client.from_service_account_json(KEY_FILE)
list_files(client)
list_buckets(client)
```

</div>

هو بيعمل حاجتين: بيعمل list للملفات اللي في الـ bucket، وبيعمل list لكل الـ buckets اللي في الـ project. والسطر اللي فيه `from_service_account_json` هو اللي بيعمل sign in بملف الـ key. شغّله في الـ terminal بتاع الـ owner:

<div dir="ltr">

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

</div>

اشتغل. وماستخدمش الـ account بتاعك خالص: كل حاجة حصلت كأنها `storage-admin-sa`، عن طريق الـ key.

### 5. الـ key بيتنقل من إيد لإيد

**الـ developer.** الـ developer بيطلب الـ key "عشان يجرب حاجة بس"، وانت بتبعتهوله. (هنا الـ terminal بتاعه معاه الملف أصلًا، عشان هو نفس الفولدر.) وبيشغّل نفس الـ script، وبيشتغل: الـ list الاتنين، وكل حاجة. افتكر إن بعد المرحلة الأولى، الـ developer يقدر يقرا الملفات اللي في bucket واحد بس. لكن بالـ key، يقدر يعمل list لكل الـ buckets اللي في الـ project، ويعمل أي حاجة تانية الـ Storage Admin بتسمح بيها، زي إنه يمسح ملفات، أو يمسح buckets بحالها، أو يغير مين ليه access عليها. والـ roles المظبوطة اللي عملناها في المرحلة الأولى مبقتش فارقة.

**الـ stranger.** دلوقتي افترض إن الملف اتسرب: حد عمله paste في شات، أو عمله commit في repository. الـ stranger بيشغّل الـ script في الـ terminal بتاعه. هو عمره ما عمل sign in على Google Cloud، وانت عمرك ما سمعت عنه. ولسه شغال.

**الـ VM.** في شباك الـ SSH بتاع الـ VM، دوس **Upload file** وارفع `with-key.py` و `sa-key.json`. وبعدين نزّل الـ library وشغّل الـ script:

<div dir="ltr">

```bash
sudo apt-get update && sudo apt-get install -y python3-venv
python3 -m venv venv
source venv/bin/activate
pip install google-cloud-storage
python3 with-key.py
```

</div>

وهنا كمان بيشتغل، على VM مالهاش أي service account.

![لوحة حمرا: ملف key واحد، بيتنقل من مكان لمكان. الملف sa-key.json اتعمله copy في أربع أماكن: اللابتوب بتاعك، والـ developer اللي طلبه عشان يجرب، والـ stranger اللي لقى نسخة متسربة، والـ lab-vm اللي مالهاش service account. ومن الأربع أماكن، الكود بيعمل sign in كأنه storage-admin-sa، اللي عنده الـ role بتاعة Storage Admin على كل الـ buckets اللي في الـ project. والـ Google Cloud مش شايفة غير storage-admin-sa، ومش قادرة تعرف مين اللي ماسك الـ key.](images/key-everywhere.drawio.svg)

أربع أماكن، والـ Google Cloud بتشوف نفس الـ identity كل مرة: `storage-admin-sa`. مش قادرة تفرق بين زميلك وواحد غريب. وده اللي كنا نقصده في EP-04 لما قلنا إن الـ key بيشتغل زي الـ password، بس أسوأ.

## المرحلة التالتة (Phase 3): نفس الشغل، من غير keys

### 1. امسح الـ key

1. افتح `storage-admin-sa` وروح على تاب **Keys** بتاعه.
2. دوس على أيقونة الزبالة اللي جنب الـ key وأكّد.

ومن الـ gcloud، الأول هات الـ ID بتاع الـ key:

<div dir="ltr">

```bash
gcloud iam service-accounts keys list --managed-by=user \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

</div>

وبعدين امسحه:

<div dir="ltr">

```bash
gcloud iam service-accounts keys delete KEY_ID \
    --iam-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

</div>

دلوقتي أي نسخة من `sa-key.json` بقت ملهاش لازمة، في أي مكان. شغّل `with-key.py` في أي حتة، وهيفشل. (الـ tokens اللي اتطلعت بالـ key قبل كده بتفضل شغالة لحد ما تخلص، وده بياخد ساعة بالكتير.)

### 2. التطبيق من غير key

احفظ نسخة من الـ script باسم `without-key.py`. امسح سطر الـ `KEY_FILE`، وغيّر السطر اللي بيعمل الـ client للسطر ده:

<div dir="ltr">

```python
client = storage.Client()
```

</div>

مفيش اسم ملف في أي حتة دلوقتي. الـ library بتلاقي الـ credentials لوحدها، عن طريق حاجة اسمها **Application Default Credentials**، أو الـ ADC. والـ ADC بيدور في تلات أماكن، بالترتيب ده:

1. الـ environment variable اللي اسمه `GOOGLE_APPLICATION_CREDENTIALS`، لو بيشاور على ملف credentials.
2. الملف اللي `gcloud auth application-default login` بيعمله.
3. الـ service account المربوط بالـ resource اللي الكود شغال عليه. والكود بياخده من الـ metadata server.

ولو مالقاش حاجة، بيفشل بـ `DefaultCredentialsError`.

![الكود فيه client = storage.Client()، من غير ملف key. والـ ADC بيدور بالترتيب. الأول، الـ environment variable اللي اسمه GOOGLE_APPLICATION_CREDENTIALS، وده بيشاور على ملف، ومش مستخدم في الـ lab ده. لو مش موجود، الملف اللي gcloud auth application-default login بيحفظه، وده اللي الـ owner والـ developer بيستخدموه على اللابتوبات بتاعتهم. ولو مفيش ملف، الـ service account المربوط، والكود بياخده من الـ metadata server، وده اللي lab-vm بتستخدمه بعد ما storage-admin-sa يتربط بيها. ولو مفيش حاجة مربوطة، بيفشل بـ DefaultCredentialsError.](images/adc-order.drawio.svg)

وعشان كده نفس الكود يقدر يشتغل على اللابتوب بتاعك (المكان التاني) وعلى VM (المكان التالت) من غير أي تغيير.

### 3. اربط الـ service account بالـ VM

مينفعش تغير الـ service account بتاع VM وهي شغالة، فاعملها stop الأول:

1. في **VM instances**، اختار `lab-vm` ودوس **Stop**.
2. لما تقف، افتحها ودوس **Edit**.
3. في **Security**، جوه **Identity and API access**، خلي الـ **Service account** هو `storage-admin-sa`.
4. خلي الـ **Access scopes** على **Allow full access to all Cloud APIs**. وزي ما شرحنا في EP-04، الـ roles بتاعة الـ service account هي اللي بتحدد الـ access.
5. دوس **Save**، وبعدين شغّل الـ VM تاني.

وبالـ gcloud، الأول اعمل stop للـ VM:

<div dir="ltr">

```bash
gcloud compute instances stop lab-vm --zone=europe-west3-a
```

</div>

وبعدين غيّر الـ service account والـ scopes:

<div dir="ltr">

```bash
gcloud compute instances set-service-account lab-vm --zone=europe-west3-a \
    --service-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com \
    --scopes=cloud-platform
```

</div>

وشغّلها تاني:

<div dir="ltr">

```bash
gcloud compute instances start lab-vm --zone=europe-west3-a
```

</div>

عشان تربط service account، لازم يبقى عندك الـ role بتاعة Service Account User عليه. ولأنك الـ Owner بتاع الـ project، فهي عندك أصلًا.

اعمل SSH على الـ VM تاني. الـ `gcloud auth list` هيوريك `storage-admin-sa` كـ active account. ارفع `without-key.py`، وامسح ملف الـ key القديم، وشغّل الـ script:

<div dir="ltr">

```bash
rm sa-key.json
source venv/bin/activate
python3 without-key.py
```

</div>

الـ list الاتنين رجعوا، ومفيش أي key على الـ VM. والـ Google Cloud بتدي الـ VM tokens قصيرة العمر للـ service account، وبتغيرها لوحدها.

### 4. الـ owner، على اللابتوب بتاعه

في الـ terminal بتاع الـ owner، شغّل الـ script الجديد:

<div dir="ltr">

```bash
python3 without-key.py
```

</div>

هيفشل:

<div dir="ltr">

```
google.auth.exceptions.DefaultCredentialsError: Your default credentials were not found.
```

</div>

بس انت عملت sign in على الـ gcloud، صح؟ أيوه، بس الـ `gcloud auth login` بيعمل sign in للـ gcloud نفسه بس. الـ client libraries مش بتستخدم الـ credentials دي. هي بتدور على الـ ADC، ولسه مفيش. اعمله:

<div dir="ltr">

```bash
gcloud auth application-default login
```

</div>

اعمل sign in في الـ browser تاني. الـ gcloud هيحفظ الـ credentials في ملف اسمه `application_default_credentials.json`، في فولدر الإعدادات بتاعه. وده بيبقى `~/.config/gcloud` في العادي، أو الفولدر اللي في `CLOUDSDK_CONFIG`. شغّل الـ script تاني، وهيشتغل، المرة دي بيك انت: بالـ account بتاعك، وبالـ roles بتاعتك. وممكن تشوف warning عن حاجة اسمها quota project. تقدر تتجاهله في الـ lab ده.

### 5. الـ developer، بالـ account بتاعه

الـ developer بيعمل نفس الكلام:

<div dir="ltr">

```bash
gcloud auth application-default login
python3 without-key.py
```

</div>

المرة دي، الجزئين بيفشلوا بـ 403 errors. الـ developer مالوش `storage.buckets.get`، اللي الـ `get_bucket` محتاجاها، ولا `storage.buckets.list`. وده صح: الـ role الوحيدة اللي عنده هي Object Viewer على الـ bucket.

بس الـ developer محتاج فعلًا يجرب التطبيق بالـ access بتاع الـ service account. وده بالظبط اللي الـ impersonation معمولة عشانه.

### 6. الـ developer بيعمل impersonate للـ service account

<div dir="ltr">

```bash
gcloud auth application-default login \
    --impersonate-service-account=storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

</div>

بعد ما يعمل sign in، ملف الـ ADC هيبقى شكله مختلف:

<div dir="ltr">

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

</div>

الـ type بتاعه بقى `impersonated_service_account`. والـ `source_credentials` دي الـ credentials بتاعة الـ developer نفسه، والـ URL هو المكان اللي الـ library بتطلب منه token للـ service account. ومفيش أي key فيه.

شغّل الـ script تاني:

<div dir="ltr">

```bash
python3 without-key.py
```

</div>

فيه error تاني، بس مختلف:

<div dir="ltr">

```
error: ('Unable to acquire impersonated credentials', ... "code": 403,
  "message": "Permission 'iam.serviceAccounts.getAccessToken' denied on resource (or it may not exist)." ...
```

</div>

الـ developer طلب token، والـ IAM قاله لأ: مالوش الـ role بتاعة Token Creator على `storage-admin-sa`.

### 7. الـ owner بيسمح بيها

في الـ terminal بتاع الـ owner، ادي الـ developer الـ Token Creator على الـ service account:

<div dir="ltr">

```bash
gcloud iam service-accounts add-iam-policy-binding \
    storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:DEV_EMAIL" \
    --role="roles/iam.serviceAccountTokenCreator"
```

</div>

ومن الـ console، ده بيبقى في صفحة الـ service account: افتح `storage-admin-sa`، وروح على تاب **Permissions** بتاعه، ودوس **Grant access**، وادي الـ developer الـ role بتاعة **Service Account Token Creator**.

وخد بالك الـ role بتتحط فين: على الـ service account، مش على الـ project. لو اتحطت على الـ project، هتخلي الـ developer يقدر يعمل impersonate لكل الـ service accounts اللي فيه.

استنى دقيقة أو اتنين لحد ما التغيير يتطبق. وبعدين الـ developer يشغّل الـ script تاني:

<div dir="ltr">

```bash
python3 without-key.py
```

</div>

الملفات والـ buckets، كأنه `storage-admin-sa`، ومن غير أي key. ولو جالك error بيقول إن الـ IAM Service Account Credentials API مقفول، شغّله بـ `gcloud services enable iamcredentials.googleapis.com` وجرب تاني.

### 8. وإيه أخبار الـ stranger؟

مفيش حاجة يستخدمها. الـ key اتمسح. وهو عمره ما عمل sign in، وحتى لو عمل sign in بـ Google account بتاعه، مش هيبقى عنده Token Creator على الـ service account.

![لوحة خضرا: مفيش ملف key في أي حتة. الـ lab-vm، اللي storage-admin-sa مربوط بيها، بتاخد token من الـ metadata server. والـ developer، اللي عنده Token Creator على storage-admin-sa، بياخد token بيعيش لحد ساعة. والاتنين بيتصرفوا كأنهم storage-admin-sa، بالـ role بتاعة Storage Admin، على الـ bucket. والـ stranger مالوش key ولا role ومش عامل sign in، فمفيش حاجة يستخدمها. والـ tokens عمرها قصير، ومحدش ياخدها غير الـ principals اللي الـ IAM سامح لهم.](images/no-keys.drawio.svg)

حاجة لازم تاخد بالك منها: ملف الـ ADC بتاع الـ developer مش بريء هو كمان. فيه الـ refresh token بتاعه هو، فلازم يفضل private زي أي credential. الفرق إنه مربوط بالـ developer. هو يقدر يعمله revoke، ولما تشيل منه الـ role بتاعة Token Creator، الملف مش هيقدر يوصل للـ service account تاني.

## نضّف وراك

كـ owner، امسح الحاجات اللي الـ lab عملها. الأول الـ VM:

<div dir="ltr">

```bash
gcloud compute instances delete lab-vm --zone=europe-west3-a
```

</div>

وبعدين الـ bucket. الـ command ده بيمسح الملفات، وبعدين الـ bucket نفسه:

<div dir="ltr">

```bash
gcloud storage rm --recursive gs://BUCKET_NAME
```

</div>

وبعدين الـ service account. شيل الـ role بتاعته الأول، وبعدين امسحه، زي ما قلنا في EP-04:

<div dir="ltr">

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
gcloud iam service-accounts delete storage-admin-sa@PROJECT_ID.iam.gserviceaccount.com
```

</div>

الـ role بتاعة الـ developer على الـ bucket بتروح مع الـ bucket، والـ Token Creator بتاعته بتروح مع الـ service account. وبعدين اعمل sign out من الـ terminals. في الـ terminal بتاع الـ owner وبتاع الـ developer:

<div dir="ltr">

```bash
gcloud auth application-default revoke
gcloud auth revoke
```

</div>

وفي الآخر، من أي terminal، امسح فولدرات الـ lab:

<div dir="ltr">

```bash
rm -rf ~/iam-lab ~/gcloud-owner ~/gcloud-developer ~/gcloud-stranger
```

</div>

## الخلاصة

- الـ role اللي على الـ project بتتورث لكل حاجة جواه. وعلى الـ resource بتظهر إنها inherited، والـ access الفعلي هو كل الـ roles مع بعض.
- الـ role المتورثة بتتشال بس من المكان اللي اتدت فيه.
- الـ roles ممكن تبقى دقيقة جدًا. الـ Object Viewer على bucket واحد بتخلي الشخص يقرا الملفات اللي فيه، بس مايعملش list للـ buckets اللي في الـ project، ولا يقرا الـ policy بتاعة الـ bucket. والـ list بتاعة الـ buckets دي permission على الـ project.
- الـ service account key بيشتغل مع أي حد معاه: زميل عنده roles أقل، أو واحد غريب عمره ما عمل sign in، أو VM مالهاش service account. والـ Google Cloud مش بتشوف غير الـ service account.
- لما تمسح الـ key، كل النسخ بتبقى ملهاش لازمة مرة واحدة.
- الـ client libraries بتلاقي الـ credentials عن طريق الـ ADC: الـ variable اللي اسمه `GOOGLE_APPLICATION_CREDENTIALS` الأول، وبعدين الملف بتاع `gcloud auth application-default login`، وبعدين الـ service account المربوط.
- على الـ VM، اربط الـ service account. ولازم توقف الـ VM الأول. ومفيش key، فيه tokens قصيرة العمر بس.
- على اللابتوب، الـ `gcloud auth application-default login` بيستخدم الـ account بتاعك انت. وزوّد `--impersonate-service-account` عشان تتصرف كأنك service account، ولده لازم يبقى عندك Token Creator عليه.

## اختبر نفسك

1. اديت حد role على الـ project. ليه بتظهر في تاب الـ Permissions بتاع الـ bucket؟
2. تاب الـ Permissions بتاع الـ bucket مش راضي يخليك تشيل الـ role دي. ليه، وتشيلها منين؟
3. الـ developer عنده Storage Object Viewer على bucket واحد بس. يقدر يعمل list للـ buckets اللي في الـ project؟ ويعمل list للملفات اللي في الـ bucket ده؟ ويقرا الـ policy بتاعة الـ bucket؟
4. ليه صفحة الـ Buckets في الـ console بتدي الـ developer error، مع إنه يقدر يقرا الملفات؟
5. الـ developer شغّل التطبيق بالـ key بتاع الـ service account. يقدر يعمل إيه دلوقتي، وليه؟
6. عندك VM مالهاش service account، بس عليها ملف key. الكود اللي على الـ VM يقدر يوصل لـ Cloud Storage؟
7. إيه اللي بيحصل لكل نسخ الـ key لما تمسح الـ key؟
8. الكود بتاعك بيعمل `storage.Client()` من غير ملف key. بيدور على الـ credentials فين، وبأنهي ترتيب؟
9. عملت sign in بـ `gcloud auth login`، بس تطبيق الـ Python بتاعك بيقول إنه مش لاقي default credentials. ليه، وتحلها إزاي؟
10. الـ impersonation بتفشل وبتقول إن `iam.serviceAccounts.getAccessToken` اترفضت. إيه الناقص، والمفروض يتدي فين؟
11. لازم تعمل إيه قبل ما تقدر تغير الـ service account بتاع VM؟

<details>
<summary>الإجابات</summary>

1. الـ roles بتتورث من فوق لتحت في الـ hierarchy. الـ role اللي على الـ project بتتطبق على كل resource جواه، فالـ bucket بيظهرها هو كمان، ومكتوب جنبها إنها inherited من الـ project.
2. الـ role مش متدية على الـ bucket، الـ bucket بيورثها بس. شيلها من على الـ project، من **IAM & Admin** > **IAM**.
3. لأ، أيوه، لأ. الـ list بتاعة الـ buckets محتاجة `storage.buckets.list` على الـ project، وقراية الـ policy محتاجة `storage.buckets.getIamPolicy`. والـ Object Viewer بتخليه يعمل list للملفات ويقراها بس.
4. صفحة الـ Buckets بتعمل list للـ buckets اللي في الـ project، ودي محتاجة `storage.buckets.list` على الـ project. ومن غيرها، الـ console مش هيقدر يوريه الـ bucket عشان يدخل عليه. لكن الـ gcloud لسه يقدر يعمل list للملفات لو اديته اسم الـ bucket.
5. أي حاجة الـ service account يقدر يعملها، واللي هنا هي Storage Admin على كل الـ buckets اللي في الـ project. بالـ key، هو بقى الـ service account، والـ roles بتاعته هو مش فارقة.
6. أيوه. الكود بيعمل sign in بالـ key، فمش محتاج الـ VM يبقى عليها service account.
7. كلها بتبطل تشتغل، في أي مكان. والـ tokens اللي اتطلعت بالـ key قبل كده بتعيش لحد ما تخلص، ساعة بالكتير.
8. عن طريق الـ ADC: الأول الـ variable اللي اسمه `GOOGLE_APPLICATION_CREDENTIALS`، وبعدين الملف بتاع `gcloud auth application-default login`، وبعدين الـ service account المربوط بالـ resource، من الـ metadata server.
9. الـ `gcloud auth login` بيعمل sign in للـ gcloud بس. والـ client libraries بتدور على الـ ADC، فشغّل `gcloud auth application-default login`.
10. الـ role بتاعة Service Account Token Creator، تتدي للـ user على الـ service account ده، مش على الـ project كله.
11. توقفها. مينفعش تغير الـ service account والـ access scopes بتوع VM غير وهي واقفة.

</details>

## الجاي

الجزء الجاي (EP-06) عن Compute Engine، يعني الـ virtual machines بتاعة Google Cloud.

---

<sub>روابط مفيدة: [الـ IAM في Cloud Storage](https://docs.cloud.google.com/storage/docs/access-control/iam) · [الـ IAM roles بتاعة Cloud Storage](https://docs.cloud.google.com/storage/docs/access-control/iam-roles) · [إزاي تعمل وتمسح service account keys](https://docs.cloud.google.com/iam/docs/keys-create-delete) · [الـ Application Default Credentials بيشتغل إزاي](https://docs.cloud.google.com/docs/authentication/application-default-credentials) · [استخدام الـ service account impersonation](https://docs.cloud.google.com/docs/authentication/use-service-account-impersonation) · [تغيير الـ service account بتاع VM](https://docs.cloud.google.com/compute/docs/instances/change-service-account) · [الـ Secure-by-default organizations](https://docs.cloud.google.com/resource-manager/docs/secure-by-default-organizations)</sub>

</div>
