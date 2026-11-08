# Django + Cloudflare R2 (Media Storage) গাইড

Django প্রজেক্টে **Cloudflare R2** কে media/file storage হিসেবে ব্যবহার করার গাইড। R2 হলো S3-compatible object storage, তাই `django-storages` ও `boto3` দিয়ে Django এর `FileField`/`ImageField` সরাসরি R2 bucket এ ফাইল আপলোড করে।

## সূচিপত্র

- [প্রয়োজনীয় জিনিস](#প্রয়োজনীয়-জিনিস)
- [ধাপ ১: Cloudflare এ Bucket তৈরি](#ধাপ-১-cloudflare-এ-bucket-তৈরি)
- [ধাপ ২: Public access চালু](#ধাপ-২-public-access-চালু)
- [ধাপ ৩: API token তৈরি](#ধাপ-৩-api-token-তৈরি)
- [ধাপ ৪: Django প্রজেক্ট ও package](#ধাপ-৪-django-প্রজেক্ট-ও-package)
- [ধাপ ৫: `.env` ফাইল](#ধাপ-৫-env-ফাইল)
- [ধাপ ৬: `settings.py`](#ধাপ-৬-settingspy)
- [ধাপ ৭: Connection test](#ধাপ-৭-connection-test)
- [Model এ ব্যবহার](#model-এ-ব্যবহার)
- [Troubleshooting](#troubleshooting)
- [নিরাপত্তা](#নিরাপত্তা)
- [Production এ deploy করার আগে](#production-এ-deploy-করার-আগে)
- [প্রজেক্টের গঠন](#প্রজেক্টের-গঠন)

## প্রয়োজনীয় জিনিস

- Python 3.12
- Cloudflare account (R2 চালু করতে payment method লাগতে পারে; ফ্রি tier এর বর্তমান সীমা Cloudflare এর pricing পেজে দেখে নিন)

---

## ধাপ ১: Cloudflare এ Bucket তৈরি

1. Cloudflare dashboard এ **R2 Object Storage** এ যান।
2. **Create bucket** ক্লিক করে নাম দিন (এই গাইডে `media`)।
3. R2 এর মূল পেজ থেকে **Account ID** নোট করুন (৩২ অক্ষরের মান)।

## ধাপ ২: Public access চালু

R2 bucket ডিফল্টভাবে **private**। ফাইলের URL সবার জন্য খোলা চাইলে:

- **Public Development URL (r2.dev)**: bucket এর **Settings** এ গিয়ে চালু করুন। URL দেখতে `https://pub-xxxxxxxx.r2.dev`। শুধু development ও পরীক্ষার জন্য; Cloudflare এর rate limit থাকতে পারে।
- **Custom domain**: আপনার domain Cloudflare এ থাকলে bucket এর **Settings > Custom Domains** থেকে সংযুক্ত করুন (যেমন `media.yourdomain.com`)। Production এর জন্য এটাই সঠিক।

Private ফাইল রাখলে এই ধাপ বাদ দিন; তখন Django সময়সীমাওয়ালা signed URL দেবে।

## ধাপ ৩: API token তৈরি

1. R2 এর মূল পেজে **Manage API Tokens** এ যান।
2. **Create API token** ক্লিক করুন।
3. Permission: **Object Read & Write** (Read only দিলে আপলোডে `403` আসবে)।
4. **Specify bucket(s)**: **Apply to specific buckets only** বেছে শুধু `media` সিলেক্ট করুন।
5. IP filtering ফাঁকা রাখুন, TTL নিজের পছন্দমতো দিন।
6. **Create** করার পর দেখানো মানগুলো থেকে এগুলো নিন:
   - **Access Key ID**
   - **Secret Access Key**

> **Secret শুধু একবার দেখায়।** পেজ বন্ধ করার আগে `.env` এ বসান।
> **Token value** (`cfat_...`) Django এ লাগে না, `.env` এ রাখবেন না।
> স্ক্রিনশট বা চ্যাটে এই মান পাঠাবেন না। ভুলে প্রকাশ হয়ে গেলে সাথে সাথে token মুছে নতুন বানান।

---

## ধাপ ৪: Django প্রজেক্ট ও package

```
mkdir r2project
cd r2project
py -m venv venv
venv\Scripts\activate
pip install django django-storages boto3 python-dotenv
django-admin startproject config .
```

অথবা `requirements.txt` থেকে:

```
pip install -r requirements.txt
```

## ধাপ ৫: `.env` ফাইল

`manage.py` এর পাশে `.env` নামে ফাইল বানান:

```
R2_ACCOUNT_ID=<ACCOUNT_ID>
R2_ACCESS_KEY_ID=<ACCESS_KEY_ID>
R2_SECRET_ACCESS_KEY=<SECRET_ACCESS_KEY>
R2_BUCKET_NAME=media
R2_PUBLIC_DOMAIN=pub-xxxxxxxx.r2.dev
```

| Variable | কী বসবে |
|---|---|
| `R2_ACCOUNT_ID` | Cloudflare Account ID (৩২ অক্ষর)। S3 API URL (`https://<ACCOUNT_ID>.r2.cloudflarestorage.com`) এর মাঝের অংশ |
| `R2_ACCESS_KEY_ID` | নতুন token এর Access Key ID |
| `R2_SECRET_ACCESS_KEY` | নতুন token এর Secret Access Key |
| `R2_BUCKET_NAME` | Bucket এর নাম (হুবহু, বড়-ছোট হাতের অক্ষরসহ) |
| `R2_PUBLIC_DOMAIN` | `https://` ছাড়া শুধু হোস্টনাম। **Private bucket হলে এই লাইনটা মুছে দিন** |

নিয়ম:

- `=` এর দুই পাশে স্পেস বা কোটেশন নেই।
- ফাইলের নাম ঠিক `.env` হতে হবে, `.env.txt` নয় (`dir /a .env*` দিয়ে দেখুন)।
- পুরো S3 API URL লাগে না; `settings.py` Account ID থেকে নিজেই endpoint বানায়।

`.gitignore` এ অবশ্যই থাকতে হবে:

```
.env
venv/
__pycache__/
*.pyc
db.sqlite3
```

## ধাপ ৬: `settings.py`

### ক) ফাইলের উপরে

```python
import os
from pathlib import Path

import dotenv

BASE_DIR = Path(__file__).resolve().parent.parent

# Load variables from the .env file next to manage.py
dotenv.load_dotenv(BASE_DIR / '.env')
```

### খ) `INSTALLED_APPS` এ

```python
INSTALLED_APPS = [
    # ... আগের গুলো
    'storages',
]
```

### গ) `STATIC_URL` এর নিচে

```python
# Cloudflare R2 media storage (S3-compatible)
# Empty R2_PUBLIC_DOMAIN -> private bucket (signed, expiring URLs)

R2_PUBLIC_DOMAIN = os.environ.get('R2_PUBLIC_DOMAIN', '').strip()

_r2_options = {
    'access_key': os.environ['R2_ACCESS_KEY_ID'],
    'secret_key': os.environ['R2_SECRET_ACCESS_KEY'],
    'bucket_name': os.environ['R2_BUCKET_NAME'],
    'endpoint_url': f"https://{os.environ['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
    'region_name': 'auto',
    'signature_version': 's3v4',
    'addressing_style': 'path',
    'default_acl': None,
    'file_overwrite': False,   # একই নামের ফাইল ওভাররাইট হবে না
}

if R2_PUBLIC_DOMAIN:
    _r2_options.update({
        'querystring_auth': False,
        'custom_domain': R2_PUBLIC_DOMAIN,
    })

STORAGES = {
    'default': {
        'BACKEND': 'storages.backends.s3.S3Storage',
        'OPTIONS': _r2_options,
    },
    'staticfiles': {
        'BACKEND': 'django.contrib.staticfiles.storage.StaticFilesStorage',
    },
}
```

### ঘ) Email setting (Django তে `MAILERS` নামে কোনো setting নেই)

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

### মূল বিষয়

- Region সবসময় `auto`।
- Endpoint এ bucket এর নাম বসে না।
- `R2_PUBLIC_DOMAIN` থাকলে public URL, না থাকলে signed URL পাওয়া যায়। তাই public থেকে private এ যেতে শুধু `.env` বদলালেই হয়।

---

## ধাপ ৭: Connection test

### ১. Django ছাড়া (boto3 দিয়ে)

Credentials ঠিক আছে কিনা আলাদা করে দেখতে `test_r2.py` চালান:

```
py test_r2.py
```

সফল হলে আউটপুট:

```
1. Upload OK
2. Read back OK: hello r2
3. Public URL: https://pub-xxxx.r2.dev/test/hello.txt
4. Test file deleted. Connection is working.
```

### ২. Django এর ভেতর থেকে

```
py manage.py shell
```

```python
from django.core.files.base import ContentFile
from django.core.files.storage import default_storage

path = default_storage.save('test/django-hello.txt', ContentFile(b'hello from django'))
print(default_storage.url(path))
```

প্রত্যাশিত আউটপুট:

```
https://pub-xxxx.r2.dev/test/django-hello.txt
```

URL ব্রাউজারে খুলে `hello from django` দেখা গেলে public access ও ঠিক আছে। পরীক্ষা শেষে ফাইলটা মুছে দিন:

```python
default_storage.delete(path)
```

---

## Model এ ব্যবহার

```python
from django.db import models

class Document(models.Model):
    title = models.CharField(max_length=200)
    file = models.FileField(upload_to='uploads/%Y/%m/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

`ImageField` ব্যবহার করলে `pip install Pillow` লাগবে। তারপর:

```
py manage.py makemigrations
py manage.py migrate
```

Admin বা ফর্ম থেকে ফাইল আপলোড করলে সরাসরি R2 তে যাবে। Template এ `{{ obj.file.url }}` দিলে URL পাওয়া যাবে।

রেকর্ড মুছলে R2 থেকেও ফাইল মুছতে চাইলে:

```python
from django.db.models.signals import post_delete
from django.dispatch import receiver

@receiver(post_delete, sender=Document)
def delete_file_from_storage(sender, instance, **kwargs):
    if instance.file:
        instance.file.delete(save=False)
```

ফর্মে ফাইলের আকার সীমিত করতে:

```python
def clean_file(self):
    f = self.cleaned_data['file']
    if f.size > 10 * 1024 * 1024:
        raise forms.ValidationError('File must be 10 MB or smaller.')
    return f
```

---

## Troubleshooting

| Error | কারণ | সমাধান |
|---|---|---|
| `KeyError: 'R2_...'` | `.env` এ variable নেই, বা ফাইলের নাম ভুল (`.env.txt`) | `.env` ও নাম দেখুন (`dir /a .env*`) |
| `NoSuchBucket` | `R2_BUCKET_NAME` ভুল | Bucket এর নাম হুবহু মেলান |
| `AccessDenied` / `403` | Token এ Object Read **& Write** নেই, বা token অন্য bucket এর | নতুন token বানান |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch` | Access Key বা Secret ভুল কপি হয়েছে, বা বাড়তি স্পেস আছে | আবার কপি করুন, দরকারে নতুন token |
| Endpoint বা connection error | `R2_ACCOUNT_ID` ভুল | Cloudflare এর S3 API URL থেকে ID মিলিয়ে দেখুন |
| আপলোড হয় কিন্তু URL খোলে না | Public Development URL চালু নেই, বা `R2_PUBLIC_DOMAIN` ভুল | bucket Settings এ public access দেখুন; `https://` ছাড়া হোস্টনাম দিন |
| `ModuleNotFoundError: storages` | `django-storages` install হয়নি, বা `INSTALLED_APPS` এ `'storages'` নেই | `pip install django-storages boto3` চালান |

`.env` ঠিকমতো পড়া হচ্ছে কিনা দেখতে (আউটপুট শেয়ার করার আগে secret মুছে দিন):

```
py -c "import dotenv; print(dotenv.dotenv_values('.env'))"
```

---

## নিরাপত্তা

- **Secret key, Access Key, Account ID** git, চ্যাট, স্ক্রিনশট বা ফ্রন্টএন্ড কোডে যাবে না। প্রকাশ হয়ে গেলে Cloudflare এর **Manage API Tokens** থেকে token মুছে নতুন বানান।
- Token এর scope শুধু প্রয়োজনীয় bucket এ রাখুন, সব bucket এ নয়।
- **Public bucket এর ফাইল যে কেউ খুলতে পারে।** NID, রিসিপ্ট, ব্যক্তিগত ডকুমেন্ট এখানে রাখবেন না; সেগুলোর জন্য আলাদা **private bucket** বানান (`R2_PUBLIC_DOMAIN` ফাঁকা রেখে signed URL ব্যবহার করুন)।
- r2.dev URL শুধু development এর জন্য। Production এ custom domain ব্যবহার করুন।
- আপলোডে login বাধ্যতামূলক করুন (`@login_required`) এবং আপলোডের ধরন ও আকার সীমিত করুন।
- ব্যবহারকারী সরাসরি ব্রাউজার থেকে R2 তে আপলোড করবে, এমন ক্ষেত্রে এই পদ্ধতি নয়; তখন CORS policy ও presigned upload URL লাগে।

## Production এ deploy করার আগে

- `SECRET_KEY` কে `.env` এ সরান এবং নতুন লম্বা key বানান:

  ```
  python -c "from django.core.management.utils import get_random_secret_key as g; print(g())"
  ```

- `DEBUG = False` করুন এবং `ALLOWED_HOSTS` এ আপনার domain দিন।
- Hosting প্ল্যাটফর্মের environment variables এ `.env` এর সব মান বসান (`.env` ফাইল সার্ভারে আপলোড না করাই ভালো)।
- r2.dev এর বদলে custom domain সংযুক্ত করে `R2_PUBLIC_DOMAIN` বদলান।
- Static file এর জন্য `STATIC_ROOT` ঠিক করে `collectstatic` চালান।

## `.env.example`

এটি git এ commit করা যাবে (আসল মান ছাড়া):

```
R2_ACCOUNT_ID=<ACCOUNT_ID>
R2_ACCESS_KEY_ID=<ACCESS_KEY_ID>
R2_SECRET_ACCESS_KEY=<SECRET_ACCESS_KEY>
R2_BUCKET_NAME=media
# Public bucket: r2.dev হোস্ট বা custom domain, https:// ছাড়া
# Private bucket হলে এই লাইন মুছে দিন
R2_PUBLIC_DOMAIN=pub-xxxxxxxx.r2.dev
```

## প্রজেক্টের গঠন

```
R2/
├── config/            # Django settings, urls, wsgi
├── manage.py
├── test_r2.py         # Django ছাড়া connection test
├── requirements.txt
├── .env               # গোপন, git এ যাবে না
├── .env.example       # নমুনা, git এ যাবে
├── .gitignore
└── README.md
```
