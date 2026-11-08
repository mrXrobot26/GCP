<div dir="rtl" lang="ar">

# الـ Service Accounts والـ Keys والـ Impersonation، الجزء التاني من الـ IAM، الحلقة (EP-04)

**عربي** | [English](README.md)

في EP-03 شفنا إن الـ principal ممكن يبقى شخص، أو group، أو service account، وسيبنا الـ service accounts لبعدين. وهي تستاهل جزء لوحدها. هتقابلها في كل project تقريبًا، وهي ورا مشاكل security كتير، وأغلبها بسبب الطريقة اللي الـ keys بتاعتها بتتعامل بيها.

الجزء ده بيغطي الـ service account يعني إيه، وأنواعه، والكود بتاعك بيستخدمه إزاي، وليه الـ keys خطر، والطرق الأأمن اللي بتعمل نفس الشغل، وأهمها الـ impersonation.

## الـ User accounts والـ service accounts

الشخص اللي بيشتغل على Google Cloud بيستخدم **user account**. وده ليه إيميل و password (والأحسن يبقى معاهم MFA)، وبياخد IAM roles بتحدد الشخص ده يقدر يعمل إيه.

أما الـ **service account** فهو account لتطبيق بدل ما يكون لشخص. وبرضه ليه إيميل، وبرضه بياخد IAM roles، بالظبط زي أي principal تاني. والفروق:

- مفيش ليه password. بيثبت هو مين بـ key، وفي أغلب الحالات Google هي اللي بتهتم بالـ key ده بدالك.
- مايقدرش يعمل sign in على الـ console من الـ browser. محدش بيكتب فيه حاجة، الكود هو اللي بيستخدمه.
- الإيميل بتاعه شكله كده: `my-app@my-project.iam.gserviceaccount.com`.

افترض إن التطبيق بتاعك شغال على VM ومحتاج يقرا ملفات من bucket ويكتب logs. مش هتحط الإيميل والـ password بتوعك جوه التطبيق. انت بتعمل service account للتطبيق، وتديله الـ roles اللي محتاجها، والتطبيق بيشتغل بالـ service account ده.

![كارتين. الـ user account ليه إيميل زي bob@mycompany.com، وبيثبت هو مين بـ password ومعاه MFA، وبيستخدمه شخص، ويقدر يعمل sign in على الـ console. والـ service account ليه إيميل زي my-app@my-project.iam.gserviceaccount.com، وبيثبت هو مين بـ key، وبيستخدمه تطبيق أو خدمة من Google Cloud، ومايقدرش يعمل sign in على الـ console. والاتنين بياخدوا الـ access عن طريق الـ IAM roles.](images/user-vs-service-account.drawio.svg)

## أنواع الـ service accounts

فيه مجموعتين كبار: اللي Google بتديرهم، واللي انت بتديرهم.

### الـ Service agents: بتديرهم Google

دول Google بتعملهم وبتستخدمهم عشان تشتغل في الـ project بتاعك نيابة عن خدمة معينة، مثلًا لما خدمة من خدمات Google تحتاج توصل لحاجة عندك عشان تعمل شغلها. وبيتعملوا ويتداروا من Google بره الـ project بتاعك، فمش بيظهروا في لستة الـ service accounts بتاعتك، وصفحة الـ IAM بتخبي الـ roles بتاعتهم إلا لو علّمت على **Include Google-provided role grants**.

في العادي بتسيبهم في حالهم. ماتشيلش الـ roles بتاعتهم، وإلا الخدمة اللي هما تبعها ممكن تبطل تشتغل. وكل فين وفين، feature معينة بتطلب منك تدي service agent role زيادة، ودي تقريبًا المرة الوحيدة اللي بتلمسهم فيها.

### الـ User-managed service accounts

دول اللي انت بتشتغل معاهم، وليهم نوعين.

الـ **User-created** service accounts هي اللي انت بتعملها بنفسك. بتسميها، وتديها roles، وتعملها disable أو delete لما متبقاش محتاجها. وكل واحدة بتاخد إيميل شكله `NAME@PROJECT_ID.iam.gserviceaccount.com`. اديها أسامي واضحة، زي `orders-api` أو `nightly-backup`، عشان أي حد يعرف كل واحدة لازمتها إيه. وده النوع اللي المفروض تستخدمه.

الـ **Default** service accounts بتتعمل لوحدها من بعض الخدمات لما تشغّل الـ API بتاعها. وهي أقدم من الـ user-created، بس لسه موجودة:

<table dir="rtl">
<thead>
<tr><th align="right" style="text-align:right">الخدمة</th><th align="right" style="text-align:right">الـ Default service account</th></tr>
</thead>
<tbody>
<tr><td>الـ Compute Engine</td><td><code>PROJECT_NUMBER-compute@developer.gserviceaccount.com</code></td></tr>
<tr><td>الـ App Engine</td><td><code>PROJECT_ID@appspot.gserviceaccount.com</code></td></tr>
</tbody>
</table>

المشكلة إن الـ default service account، على حسب إعدادات الـ organization بتاعتك، ممكن ياخد الـ role بتاعة **Editor** على الـ project أوتوماتيك. وده أكتر بكتير من اللي أي تطبيق لوحده محتاجه، وعكس الـ least privilege. والـ organizations اللي اتعملت بعد 3 مايو 2024 الحكاية دي مقفولة فيها by default. وفي أي حالة تانية، اتأكد بنفسك: لو default service account عنده Editor، شيله.

![شجرة. الـ service accounts بتتقسم لـ user-managed، يعني انت اللي بتديرها، و Google-managed، يعني الـ service agents. والـ user-managed يا إما user-created، ودي اللي المفروض تستخدمها، يا إما default، ودي بتتعمل لما تشغّل بعض الـ APIs، وممكن تاخد الـ Editor role، وبيشترك فيها كل اللي بيستخدمها. والـ service agents بتعملها وبتشغلها Google، ومستخبية في صفحة الـ IAM by default، وبتسيب الـ roles بتاعتها زي ما هي.](images/service-account-types.drawio.svg)

### ليه الـ default service account مشكلة

افترض إن عندك تلات VMs كلهم بيستخدموا الـ default service account بتاع Compute Engine. وده اللي هيحصل فعلًا إلا لو اخترت واحد تاني وانت بتعملهم.

- الـ `vm-1` محتاجة تكتب ملفات في Cloud Storage وتكتب logs.
- الـ `vm-2` محتاجة تبعت messages لـ Pub/Sub وتقرا داتا من BigQuery.
- الـ `vm-3` محتاجة تقرا secret واحد من Secret Manager.

ولأنهم شايلين service account واحد، الـ account ده لازم ياخد كل الـ roles دي. ودلوقتي كل VM تقدر تعمل كل ده: الـ `vm-3` تقدر تكتب في الـ buckets بتاعتك وتقرا من BigQuery، مع إنها محتاجة secret واحد بس. ولو أي VM فيهم اتخترقت، اللي هاجمها هياخد كل حاجة.

والحل إن كل VM (أو كل تطبيق) يبقى ليه service account لوحده، وكل واحد معاه الـ roles اللي الـ VM بتاعته محتاجاها بس. الـ default service account لسه شغال، وفي test سريع مفيش مشكلة، بس لأي حاجة بجد، اعمل الـ service accounts بتاعتك انت.

![على الشمال: تلات VMs شايلين الـ default service account بتاع Compute Engine، وعليه خمس roles، فكل VM تقدر تستخدم الخمسة. وعلى اليمين: كل VM ليها service account لوحدها. الـ vm-1-sa عليها Storage Object Admin و Logs Writer، والـ vm-2-sa عليها Pub/Sub Publisher و BigQuery Data Viewer، والـ vm-3-sa عليها Secret Manager Secret Accessor، فكل VM تقدر تستخدم الـ roles اللي محتاجاها بس.](images/default-vs-dedicated.drawio.svg)

ولما تعمل VM هتشوف كمان حاجة اسمها **access scopes**. ودي طريقة أقدم عشان تحدد الـ VM تقدر تعمل إيه بالـ service account بتاعها: الـ request مش بيعدي غير لو الـ scopes والـ IAM roles الاتنين سامحين بيه. و Google دلوقتي بتنصح بطريقة أبسط: ادي الـ VM الـ scope الكامل `cloud-platform` (يعني "Allow full access to all Cloud APIs")، واتحكم في الـ access عن طريق الـ roles بتاعة service account مخصوص ليها. وكده بيبقى عندك مكان واحد تدير منه الـ access بدل اتنين.

## الـ service account بيبقى principal و resource في نفس الوقت

الـ service account بيلعب دورين في نفس الوقت:

- **كـ principal**، بياخد roles على resources تانية، زي Storage Object Viewer على bucket أو Cloud SQL Client على project. وده اللي بيخلي التطبيق بتاعك يعرف يشتغل.
- **كـ resource**، الـ principals التانيين بياخدوا roles عليه. لازم حد يبقى مسموح له يعمله، أو يدير الـ keys بتاعته، أو يربطه بـ VM، أو يعمله impersonate.

ودي الـ roles اللي هتشوفها على الـ service accounts:

<table dir="rtl">
<thead>
<tr><th align="right" style="text-align:right">الـ Role</th><th align="right" style="text-align:right">الـ ID</th><th align="right" style="text-align:right">بتخليك تعمل إيه بالـ service account</th></tr>
</thead>
<tbody>
<tr><td>الـ Service Account Admin</td><td><code>roles/iam.serviceAccountAdmin</code></td><td>تعمله، وتغيره، وتعمله disable، وتمسحه</td></tr>
<tr><td>الـ Service Account User</td><td><code>roles/iam.serviceAccountUser</code></td><td>تربطه بـ resource زي VM، عشان الكود اللي هناك يشتغل بيه</td></tr>
<tr><td>الـ Service Account Token Creator</td><td><code>roles/iam.serviceAccountTokenCreator</code></td><td>تاخد له tokens قصيرة العمر، ودي اللي الـ impersonation شغال بيها</td></tr>
<tr><td>الـ Service Account Key Admin</td><td><code>roles/iam.serviceAccountKeyAdmin</code></td><td>تعمل الـ keys بتاعته وتمسحها</td></tr>
</tbody>
</table>

خلي بالك مين بياخد الـ roles دي. اللي يقدر يربط service account قوي أو يعمله impersonate، يقدر يعمل كل اللي الـ account ده يقدر يعمله، حتى لو هو نفسه ماعندوش الـ roles دي.

![الـ service account في النص. فوقه، ناس عندهم roles عليه: الـ Service Account Admin عشان يعمله ويمسحه، والـ Service Account User عشان يربطه بـ VM، والـ Token Creator عشان يعمله impersonate. وتحته، الـ service account عنده roles على resources تانية: الـ Storage Object Viewer على Cloud Storage، والـ Cloud SQL Client على Cloud SQL.](images/principal-and-resource.drawio.svg)

## الـ Service account keys

أقدم طريقة تستخدم بيها service account من الكود بتاعك هي الـ **key**. بتعمل key للـ service account وتنزّله كملف JSON، والكود أو الأداة بتاعتك بتقرا الملف ده عشان تعمل sign in على Google Cloud بالـ service account. وهتلاقي الـ keys في أماكن زي دي:

- تطبيق شغال على VM أو Cloud Run أو GKE، وبيقرا من Cloud Storage أو Cloud SQL.
- الـ CI/CD pipelines وأدوات الـ infrastructure as code زي Terraform، اللي بتعمل الـ resources بدالك.
- تطبيقات شغالة بره Google Cloud: على السيرفرات بتاعتك، أو على AWS، أو على Azure.
- اللابتوبات بتاعة الـ developers، عشان يجربوا الكود بتاعهم على resources حقيقية.

### ليه الـ keys خطر

الـ key بيشتغل زي الـ password، بس أسوأ. الـ password بتاع الـ user account وراه MFA وفحوصات الـ sign in بتاعة Google، فالـ password المسروق لوحده غالبًا مش كفاية. لكن الـ service account key مفيش وراه أي حاجة من دي: **أي حد معاه الـ key يقدر يتصرف بالظبط كأنه الـ service account**، ومن أي مكان. وكمان الـ keys اللي بتعملها مش بتنتهي (expire) إلا لو ظبطت ده. الـ key بيفضل شغال لحد ما حد يمسحه.

فلو هتستخدم keys، اتعامل معاها بحرص.

**ماتعملش:**

- تعمل commit للـ keys في source code repository. ومسح الملف بعدين مش هيفيد، لأنه بيفضل في الـ history. ولو ده حصل، امسح الـ key نفسه من الـ IAM.
- تبعت الـ keys لناس تانيين على الإيميل أو الشات.
- تسيب نسخ منها في أماكن مؤقتة، زي فولدر الـ Downloads.
- تحط الـ keys في Cloud Storage bucket عام (public).
- تعتمد على secret store زي Secret Manager إنه يحل المشكلة. ونصيحة Google إنك ماتخزنش الـ keys هناك: عشان التطبيق يقرا الـ secret، لازم يبقى معاه أصلًا identity بتثق فيها Google Cloud، والـ identity دي ممكن تتستخدم على طول بدل الـ key.

**اعمل:**

- دوّر على الـ keys اللي محدش بيستخدمها وامسحها. الـ service account insights والـ metrics بتاعة استخدام الـ keys بيوروك أنهي keys ماتستخدمتش من فترة.
- اعمل rotation للـ keys بانتظام: اعمل key جديد، وخلي التطبيق يستخدمه، وبعدين امسح القديم. ولو key قديم كان اتسرب، هيبطل يشتغل.
- حط expiry time، عشان الـ keys الجديدة تبطل تشتغل لوحدها بعد فترة.
- شغّل الـ organization policy اللي بتعمل disable للـ key أوتوماتيك لما Google تلاقيه متسرب في مكان عام، زي مثلًا لو اتعمله push على GitHub repository عام.

والـ [best practices for managing service account keys](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys) بتاعة Google بتشرح كل ده بالتفصيل، ويستاهل تقراها على مهلك.

### الأحسن: ماتستخدمش keys خالص

الـ key عبارة عن credential عمره طويل، ونصيحة Google إنك تستخدمه بس لما مفيش أي حاجة تانية تنفع. ودلوقتي فيه بديل أأمن لكل حالة تقريبًا:

- **الكود اللي شغال على Google Cloud**، زي VM أو App Engine أو Cloud Run: اربط الـ service account بالـ resource. مكتبات Google Cloud (الـ client libraries) بتلاقي الـ credentials لوحدها (ودي اسمها Application Default Credentials)، و Google بتعمل الـ keys وبتغيرها من وراك. ومفيش ملف key في أي حتة.
- **الكود اللي على GKE**، الـ Kubernetes اللي بتديرها Google: الـ Workload Identity Federation for GKE بتخلي كل workload يشتغل كـ service account. وهنتكلم عنها أكتر في جزء الـ GKE.
- **اللابتوب بتاعك**: استخدم الـ account بتاعك انت (`gcloud auth application-default login`)، أو اعمل impersonate لـ service account، زي ما هنشوف تحت.
- **الـ CI/CD والـ Terraform**: الـ impersonation، أو الـ Workload Identity Federation لو الأداة شغالة بره Google Cloud.
- **التطبيقات اللي بره Google Cloud**، زي AWS و Azure والسيرفرات بتاعتك: الـ Workload Identity Federation. وفيها Google Cloud بتثق في الـ identity اللي التطبيق عنده أصلًا في المكان اللي شغال فيه، وبتديله token قصير العمر بدل الـ key. وغالبًا بتتستخدم مع الـ impersonation.

والـ Workload Identity Federation موضوع كبير لوحده، فمش هندخل فيه أكتر من كده هنا.

![الكود شغال فين، وتستخدم إيه بدل الـ key. على Google Cloud، زي VM أو App Engine أو Cloud Run: اربط service account بالـ resource. على GKE: الـ Workload Identity Federation for GKE. على اللابتوب بتاعك: الـ credentials بتاعتك، أو الـ impersonation. في الـ CI/CD والـ Terraform: الـ impersonation، أو الـ Workload Identity Federation. بره Google Cloud: الـ Workload Identity Federation. والـ key هو آخر حل.](images/key-alternatives.drawio.svg)

## الـ Service account impersonation

الـ impersonation معناها إن principal يتصرف كأنه service account لفترة قصيرة. وبالعربي الكلمة معناها انتحال شخصية، بس مفيش فيها أي حاجة مش تمام. دي الطريقة العادية والآمنة إنك تستخدم صلاحيات service account من غير key.

وده بيشتغل كده:

1. بتعمل service account وتديله الـ roles اللي الشغل محتاجها، مثلًا Storage Admin و Cloud SQL Client.
2. بتدي user (أو service account تاني) الـ role بتاعة **Service Account Token Creator** على الـ service account ده.
3. الـ user بيعمل sign in بنفسه، وبعدين بيطلب من Google Cloud access token للـ service account.
4. الـ IAM بيتأكد إن الـ user عنده Token Creator على الـ service account ده. لو عنده، بيرجعله **short-lived access token**، وده بيعيش ساعة واحدة بالكتير by default.
5. الكود بتاع الـ user، أو الـ gcloud، بيكلم Google Cloud بالـ token ده، كأنه الـ service account.

ولما الـ token يخلص، الـ user لازم يطلب تاني، والـ IAM بيتأكد من الـ role تاني. ولو شلت الـ role، مش هيقدر ياخد tokens جديدة. ومفيش key بيتعمل أصلًا، فمفيش حاجة عمرها طويل ممكن تتسرب. وكمان الـ audit logs بتسجل الاتنين: الـ service account، والشخص اللي عمله impersonate.

![الـ alice@example.com عاملة sign in بالـ gcloud، وعندها الـ role بتاعة Service Account Token Creator على storage-sa. الخطوة 1: بتطلب من Google Cloud token لـ storage-sa. الخطوة 2: بيرجعلها token بيعيش لحد ساعة. الخطوة 3: بتكلم الـ Cloud Storage API بالـ token ده. والـ storage-sa عليها Storage Admin و Cloud SQL Client. ومفيش أي key.](images/impersonation.drawio.svg)

### الـ Impersonation بالـ gcloud

الـ admin بيدي الـ user الـ role على الـ service account. والـ service account هنا هو الـ resource:

<div dir="ltr">

```bash
gcloud iam service-accounts add-iam-policy-binding SA_EMAIL \
    --member="user:USER_EMAIL" \
    --role="roles/iam.serviceAccountTokenCreator"
```

</div>

الـ user بيعمل sign in بنفسه، وبعدين بيزود `--impersonate-service-account` على أي gcloud command:

<div dir="ltr">

```bash
gcloud auth login
gcloud storage ls --impersonate-service-account=SA_EMAIL
```

</div>

الـ gcloud بيطبع warning إنه بيستخدم impersonation، وبعدين بيشغّل الـ command كأنه الـ service account. ولو مفيش Token Creator role، الـ command بيفشل بـ permission error. وأدوات تانية كتير بتشتغل بنفس الطريقة، زي الـ Terraform مثلًا، اللي يقدر يعمل impersonate لـ service account بدل ما يستخدم key.

## جرّب بنفسك

الـ EP-05 عبارة عن lab كامل للـ IAM والـ impersonation، فده مجرد تجربة أولى سريعة. محتاج project فيه الـ Compute Engine API شغال، زي اللي عملناه في EP-02.

اعمل service account، واديله access يشوف بيه Compute Engine من غير ما يغير فيه:

<div dir="ltr">

```bash
gcloud iam service-accounts create demo-viewer --display-name="Demo viewer"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:demo-viewer@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.viewer"
```

</div>

اسمح لنفسك تعمله impersonate. حتى لو انت الـ owner بتاع الـ project، محتاج الـ role دي:

<div dir="ltr">

```bash
gcloud iam service-accounts add-iam-policy-binding \
    demo-viewer@PROJECT_ID.iam.gserviceaccount.com \
    --member="user:YOUR_EMAIL" --role="roles/iam.serviceAccountTokenCreator"
```

</div>

تغييرات الـ IAM ممكن تاخد دقيقة أو اتنين عشان تتطبق. وبعدين شغّل command كأنك الـ service account:

<div dir="ltr">

```bash
gcloud compute instances list \
    --impersonate-service-account=demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

</div>

هيشتغل، حتى لو اللستة فاضية. دلوقتي جرّب حاجة الـ service account مش مسموح له يعملها:

<div dir="ltr">

```bash
gcloud storage buckets list \
    --impersonate-service-account=demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

</div>

دي هتفشل بـ permission error. الـ service account معاه Compute Viewer بس، فطول ما انت بتتصرف كأنك هو، ده كل اللي هتاخده، حتى لو انت صاحب الـ project.

وعشان تنضف، شيل الـ role الأول، وبعدين امسح الـ service account. لو مسحته الأول، الـ binding القديم بتاعه هيفضل في الـ policy بتاعة الـ project على شكل entry اسمه `deleted:serviceAccount:...`.

<div dir="ltr">

```bash
gcloud projects remove-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:demo-viewer@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.viewer"

gcloud iam service-accounts delete demo-viewer@PROJECT_ID.iam.gserviceaccount.com
```

</div>

## الخلاصة

- الـ service account هو account لتطبيق أو لخدمة من خدمات Google Cloud. ليه إيميل وبياخد IAM roles، بس مفيش ليه password ومايقدرش يعمل sign in على الـ console.
- الـ service agents هي الـ service accounts بتاعة Google نفسها، وخدماتها بتستخدمها عشان تشتغل في الـ project بتاعك. سيبها في حالها.
- الـ user-managed service accounts يا إما user-created (ودي اللي تستخدمها)، يا إما default بتعملها خدمات زي Compute Engine. والـ default مشتركة بين حاجات كتير وممكن تاخد Editor: شيله، أو الأحسن ماتستخدمهاش.
- استخدم service account لكل تطبيق أو VM، ومعاه الـ roles اللي محتاجها بس.
- الـ service account بيبقى principal (بياخد roles على resources) و resource (الناس بتاخد roles عليه، زي Service Account User أو Token Creator).
- الـ key بيخلي أي حد معاه يتصرف كأنه الـ service account، من غير MFA، ومش بينتهي by default. ابعد عن الـ keys. ولو لازم تستخدم واحد، عمرك ما تعمله commit أو تشاركه، واعمله rotation، وامسح اللي محدش بيستخدمه.
- استخدم البدائل: اربط الـ service account على Google Cloud، والـ Workload Identity Federation على GKE وبره Google Cloud، والـ credentials بتاعتك أو الـ impersonation للـ development والـ CI/CD.
- في الـ impersonation، اللي عنده الـ role بتاعة Token Creator بياخد token قصير العمر للـ service account. ومفيش key، وكل استخدام بيتراجع وبيتسجل.

## اختبر نفسك

1. الـ service account بيختلف عن الـ user account في إيه؟
2. مين بيعمل الـ service agents، والمفروض تعمل معاهم إيه؟
3. الـ default service account بتاع Compute Engine في الـ project بتاعك عنده الـ role بتاعة Editor. تعمل إيه؟
4. عندك تلات VMs شايلين service account واحد. ليه دي مشكلة؟
5. ليه بنقول على الـ service account إنه principal و resource في نفس الوقت؟
6. اللي عنده الـ role بتاعة Service Account User على service account يقدر يعمل إيه؟
7. ليه الـ service account key لما يتسرب بيبقى أخطر من الـ password بتاع user؟
8. زميلك عاوز يحط key في الـ Git repository بتاع التطبيق "مؤقتًا بس". هتقوله إيه؟
9. التطبيق بتاعك شغال على Cloud Run ومحتاج يقرا من Cloud Storage. المفروض يعمل sign in إزاي؟
10. الـ user محتاج أنهي role عشان يعمل impersonate لـ service account، والـ token بيعيش قد إيه؟

<details>
<summary>الإجابات</summary>

1. ده لتطبيق مش لشخص. مفيش ليه password: بيستخدم keys أو tokens، ومايقدرش يعمل sign in على الـ console. وزي الـ user account، ليه إيميل وبياخد IAM roles.
2. دي Google هي اللي بتعملها وبتديرها. سيبها زي ما هي، وماتشيلش الـ roles بتاعتها.
3. شيل الـ role بتاعة Editor، واديله الـ roles اللي محتاجها بس. والأحسن تعمل service account لكل تطبيق وتستخدمهم بداله.
4. الـ account لازم ياخد كل role أي VM فيهم محتاجاها، فكل VM بتاخدهم كلهم. ولو VM واحدة اتخترقت، اللي هاجمها هياخد كل حاجة.
5. كـ principal بياخد roles على resources تانية. وكـ resource، الـ principals التانيين بياخدوا roles عليه، زي إنهم يديروه أو يربطوه أو يعملوله impersonate.
6. يربطه بـ resource زي VM، عشان الكود اللي هناك يشتغل كأنه الـ service account. وده بيديله الـ access بتاع الـ service account بشكل غير مباشر.
7. الـ user account وراه MFA وفحوصات sign in. لكن الـ key مفيش وراه أي حاجة، ومش بينتهي by default، فأي حد معاه يقدر يتصرف كأنه الـ service account من أي مكان.
8. لأ. الـ Git بيحتفظ بيه في الـ history حتى بعد ما الملف يتمسح. والأحسن ماتستخدموش key أصلًا، ولو key اتعمله commit، امسح الـ key ده من الـ IAM.
9. اربط service account بالـ Cloud Run service، ومعاه الـ roles اللي محتاجها بس. والـ client libraries هتلاقي الـ credentials لوحدها، من غير key.
10. محتاج Service Account Token Creator على الـ service account ده. والـ token بيعيش ساعة واحدة بالكتير by default.

</details>

## الجاي

الجزء الجاي (EP-05) عبارة عن lab عملي على الـ IAM والـ service account impersonation.

---

<sub>روابط مفيدة: [نظرة عامة على الـ service accounts](https://docs.cloud.google.com/iam/docs/service-account-overview) · [أنواع الـ service accounts](https://docs.cloud.google.com/iam/docs/service-account-types) · [أحسن طريقة تستخدم بيها الـ service accounts](https://docs.cloud.google.com/iam/docs/best-practices-service-accounts) · [إزاي تدير الـ keys صح](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys) · [الـ Service account impersonation](https://docs.cloud.google.com/iam/docs/service-account-impersonation) · [الـ roles بتاعة الـ service accounts](https://docs.cloud.google.com/iam/docs/service-account-permissions) · [الـ Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation) · [الـ Application Default Credentials](https://docs.cloud.google.com/docs/authentication/application-default-credentials) · [الـ service accounts على Compute Engine](https://docs.cloud.google.com/compute/docs/access/service-accounts)</sub>

</div>
