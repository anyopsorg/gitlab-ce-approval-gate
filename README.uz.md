# gitlab-ce-approval-gate

[🇬🇧 English](README.md) | **🇺🇿 O'zbekcha**

**GitLab CE'da Merge Request tasdiqlanishini qattiq majburiylash.** Ushbu
ikkita kichik CI shabloni, agar MR'da kerakli tasdiqlar bo'lmasa,
pipeline'ni qulab tushiradi; bu yerda GitLab'ning "Pipelines must succeed"
branch protection sozlamasi bilan birga ishlatib, Merge tugmasi haqiqatan
ham bloklanadi.

GitLab'ning o'rnatilgan (native) approval rules funksiyasi — bu faqat
[Premium/EE'da mavjud](https://docs.gitlab.com/ee/user/project/merge_requests/approvals/).
GitLab CE'da siz UI orqali approval rules sozlay olasiz, lekin ular
majburiylashtirilmaydi — merge huquqi bor har qanday foydalanuvchi
tasdiqlar holatidan qat'i nazar merge qila oladi. Bu repo sizga EE
litsenziyasi olmasdan CE ustida haqiqiy majburiylash beradi.

## Nima olasiz

- **`team-gate.yml`** — himoyalangan (protected) branch'ga yo'naltirilgan
  MR'larda jamoaviy tasdiqlash qoidalarini majburiylaydi. Sozlanadigan:
  minimal umumiy tasdiqchilar soni, nomli tasdiqchilar ro'yxati (N-of-M),
  va "ro'yxatdan tashqari N tasdiqchi bo'lishi kerak" qoidasi. MR muallifi
  har doim hisobdan chiqariladi.
- **`infra-gate.yml`** — MR infrastruktura fayllariga tegsa (`Dockerfile`,
  `.gitlab-ci.yml`, `entrypoint.sh`, `helm/`), infra-egalari pulidan
  tasdiqlash talab qiladi. Faqat kod o'zgarishlari bo'lgan MR'larga
  ta'sir qilmaydi. Muallifning o'zi infra-egalaridan biri bo'lsa,
  uning mualliflik qilishi tasdiqlash sifatida hisoblanadi.

Ikkalasi birgalikda ishlaydi — bir xil pipeline'ga ikkalasini ham include
qiling, ular alohida job sifatida ishga tushadi va merge uchun ikkalasi
ham yashil bo'lishi kerak.

## Amalda qanday ko'rinadi

`main`'ga yo'naltirilgan MR'larga ikkala gate'ni ishlatgan loyihadan haqiqiy
screenshot'lar. Screenshot'lardagi username'lar — shu loyiha uchun sozlangan
qiymatlar; sizda o'z jamoangiz bo'ladi. Shablonlarning o'zi loyihaga bog'liq
hech narsa o'z ichida saqlamaydi.

### Tasdiqlar yetmasa pipeline blokirovka qilinadi

![Pipeline ro'yxati: bir MR fail, ikkinchisi passed](docs/screenshots/01-pipeline-blocked-vs-passed.png)

Pastki MR kerakli tasdiqlarsiz ochilgan — uning `mr-approval-check` job'i
fail qildi va GitLab Merge tugmasini yoqishdan bosh tortdi. Yetmagan
tasdiqchilar "Approve" bosishi va pipeline qayta ishga tushishi bilan
(yuqori qatordagi MR), ikkala gate ham o'tdi.

### Ikkala gate alohida job sifatida ishlaydi

![Related jobs paneli ikki gate job bilan](docs/screenshots/02-related-jobs-sidebar.png)

`mr-approval-check` (team approval) himoyalangan branch'ga yo'naltirilgan
har bir MR'da ishga tushadi. `devops-approval-check` esa faqat MR diff'i
infra fayllariga tegsa ishga tushadi. Har ikkalasi ham o'tishi kerak;
har biri alohida retry qilinadi.

### Team-approval gate natijasi

![Team gate job log: gate config, muallifning hisobdan chiqarilishi va qoidalar bahosi](docs/screenshots/03-team-gate-job-log.png)

Job sozlangan qoidalarni chop etadi, kim tasdiqlaganini ro'yxatlaydi,
**MR muallifini hisobdan chiqaradi** (agar u o'zini Approve qilgan
bo'lsa ham) va har bir gate uchun holatni (`OK` / `FAIL`) ko'rsatadi.
Pipeline faqat barcha sozlangan qoidalar bajarilganda yashil bo'ladi.

### Infra-approval gate natijasi

![Infra gate job log: muallif DevOps pulidan avtomatik hisoblangan](docs/screenshots/04-infra-gate-job-log.png)

MR infra fayllariga tegsa, ikkinchi job ishga tushadi.
`DevOps author auto-counted: …` qatoriga e'tibor bering — agar MR
muallifining o'zi infra pulida bo'lsa, uning mualliflik qilishi minimal
hisobiga 1 ta bo'lib qo'shiladi. Yolg'iz infra-egasi Dockerfile'ni
tahrirlasa, ikkinchi "Approve" bosilishi kerak emas.

### Ikkala gate o'tgach Merge ochiladi

![Merge qilingan MR: ikkala gate yashil, 3 ta tasdiqchi](docs/screenshots/05-merged-after-approvals.png)

"Pipelines must succeed" yoqilgan bo'lsa (sozlash quyida), Merge tugmasi
faqat barcha tegishli gate'lar yashil bo'lgandagina yoqiladi. Merge
qilinganidan keyin MR to'liq tarixni saqlab qoladi — kim tasdiqladi,
qaysi pipeline tekshirdi, va squash/merge tafsilotlari.

### Kerakli GitLab sozlamalari

Gate'ning haqiqatan merge'ni bloklashi uchun har bir loyihada uchta
sozlama bo'lishi kerak:

**1. Maqsadli branch'ni himoyalang** — Settings → Repository → Protected
branches. Merge faqat MR orqali bo'lsin; hech kim to'g'ridan-to'g'ri
push qila olmasin.

![Protected branches sozlamasi: main himoyalangan, Maintainer merge qila oladi, hech kim push qila olmaydi](docs/screenshots/06-protected-branches-setting.png)

**2. "Pipelines must succeed"** — Settings → Merge requests. Aynan shu
sozlama gate job'ning exit code'ini Merge tugmasi bilan bog'laydi.

![Pipelines must succeed merge checks'da yoqilgan](docs/screenshots/07-pipelines-must-succeed-setting.png)

**3. CI/CD variables** — Settings → CI/CD → Variables. Token va loyihaga
oid qoidalar. **Muhim:** to'rtta variable ham "Protect variable" belgilash
**o'chirilgan** holatda bo'lishi kerak — chunki MR pipeline'lari
(himoyalanmagan) source branch'da ishlaydi va Protected variable'larni
ko'ra olmaydi.

![CI/CD variables: MR_APPROVAL_GITLAB_TOKEN (Masked), MR_APPROVAL_REQUIRED_USERS, MR_APPROVAL_REQUIRED_USERS_MIN=2, MR_APPROVAL_TARGET_BRANCH=main](docs/screenshots/08-cicd-variables.png)

## Tezkor boshlash

### 1. Loyihangizning `.gitlab-ci.yml` fayliga shablonni qo'shing

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/anyopsorg/gitlab-ce-approval-gate/main/team-gate.yml'

stages:
  - validate
```

Agar ushbu reponi o'zingizning GitLab serveringizga mirror qilmoqchi
bo'lsangiz:

```yaml
include:
  - project: 'sizning-group/gitlab-ce-approval-gate'
    ref: main
    file: '/team-gate.yml'
```

### 2. Project access token yarating

Project Settings → Access tokens → Add new token:
- Name: `mr-approval-gate`
- Role: **Reporter** (MR approval'larini o'qish uchun yetadi)
- Scopes: **`read_api`** (faqat o'qish — yetarli)
- Expiration: rotatsiyani eslab qoladigan sana tanlang

### 3. Tokenni CI/CD o'zgaruvchisi sifatida qo'shing

Project Settings → CI/CD → Variables → Add:

| Maydon | Qiymat |
|---|---|
| Key | `MR_APPROVAL_GITLAB_TOKEN` |
| Value | yangi yaratilgan `glpat-…` token |
| **Protect variable** | ☐ **belgilanmagan** (juda muhim — pastda tushuntirilgan) |
| Mask variable | ☑ belgilangan |

> **Nega "Protected" o'chirilgan bo'lishi kerak:** MR pipeline'lari source
> (feature) branch ustida ishlaydi, va u protected branch emas. Protected
> CI/CD variables faqat protected branch'da ishlayotgan pipeline'larga
> ko'rinadi — demak Protected token gate job'iga ko'rinmaydi va u jim-jit
> ravishda muvaffaqiyatsiz bo'ladi.

### 4. Gate'ni sozlang

Sizga kerak bo'lgan variable'larni qo'shing (Project Settings → CI/CD →
Variables, hammasini "Protect variable" o'chirilgan holatda):

```
MR_APPROVAL_TARGET_BRANCH        main
MR_APPROVAL_MIN_TOTAL            2
```

Tamom — endi `main`'ga yo'naltirilgan har bir MR muallifdan tashqari
kamida 2 ta tasdiqchini talab qiladi.

### 5. Branch'ni qulflang

Merge tugmasini haqiqatan bloklash uchun:

- Project Settings → **Repository → Protected branches** → maqsadli
  branch uchun:
  - Allowed to merge: Maintainers (yoki sizning siyosatingizga ko'ra)
  - Allowed to push: No one
- Project Settings → **Merge requests**:
  - ☑ **Pipelines must succeed**
  - ☑ All discussions must be resolved before a merge request can be merged

## Sozlash ma'lumotnomasi

### `team-gate.yml`

| Variable | Standart | Vazifasi |
|---|---|---|
| `MR_APPROVAL_GITLAB_TOKEN` | — | **Majburiy.** `read_api`-scope'li token, Masked, Protected EMAS. |
| `MR_APPROVAL_TARGET_BRANCH` | `main` | Gate himoya qiladigan bitta branch nomi (aniq moslik). `MR_APPROVAL_TARGET_PATTERN` bo'sh bo'lsa ishlatiladi. |
| `MR_APPROVAL_TARGET_PATTERN` | (bo'sh) | Bir nechta branch'ni himoyalash uchun regex (slash'lar bilan), masalan `/^(staging\|main\|master)$/`. O'rnatilsa, `MR_APPROVAL_TARGET_BRANCH`'dan ustun turadi. |
| `MR_APPROVAL_MIN_TOTAL` | (yo'q) | Min alohida tasdiqchilar soni, har qanday rol. Bo'sh = gate o'tkazib yuboriladi. |
| `MR_APPROVAL_REQUIRED_USERS` | (yo'q) | "Talab qilingan" pulda bo'sh joy bilan ajratilgan usernamelar. Bo'sh = gate o'tkazib yuboriladi. |
| `MR_APPROVAL_REQUIRED_USERS_MIN` | pulning hammasi | Puldan nechta tasdiqlashi kerak. |
| `MR_APPROVAL_MIN_OTHERS` | (yo'q) | Required puldan *tashqari* min tasdiqchilar. Bo'sh = gate o'tkazib yuboriladi. |
| `MR_APPROVAL_IMAGE` | `alpine:3.20` | Container image (ichida `curl` + `jq` bo'lishi yoki o'rnatilishi kerak). |
| `MR_APPROVAL_HTTP_PROXY` | (bo'sh) | Tashqi proxy URL — [Korporativ proxy ortida](#korporativ-proxy-ortida) qismiga qarang. |
| `MR_APPROVAL_NO_PROXY` | (bo'sh) | NO_PROXY ro'yxati. |

Har bir gate alohida ixtiyoriy. Hech birini o'rnatmasangiz gate no-op
holatda bo'ladi (avtomatik o'tadi).

### `infra-gate.yml`

| Variable | Standart | Vazifasi |
|---|---|---|
| `MR_APPROVAL_GITLAB_TOKEN` | — | **Majburiy.** `team-gate.yml` bilan bir xil token. |
| `MR_APPROVAL_INFRA_USERS` | (bo'sh) | Infra-egalari usernamelari, bo'sh joy bilan ajratilgan. Bo'sh = gate no-op. |
| `MR_APPROVAL_INFRA_MIN` | `1` | Min tasdiqchilar puldan. |
| `MR_APPROVAL_INFRA_TARGET_PATTERN` | `/^(staging\|main\|master)$/` | Gate ishga tushadigan target branch'lar uchun regex. |
| `MR_APPROVAL_IMAGE` | `alpine:3.20` | Container image. |
| `MR_APPROVAL_HTTP_PROXY` | (bo'sh) | Pastda qarang. |
| `MR_APPROVAL_NO_PROXY` | (bo'sh) | Pastda qarang. |

Infra gate faqat MR'ning diff'i quyidagi fayllardan birortasini o'z
ichiga olsa ishga tushadi:
- `Dockerfile`, `Dockerfile.*` (root va subdir'larda)
- `.gitlab-ci.yml` (faqat root'da)
- `entrypoint.sh` (root va subdir'larda)
- `helm/**` (root'dagi helm papkasi, rekursiv)

Bu fayllar ro'yxati `rules:changes:` ichida qattiq qatlangan, chunki
GitLab bu yerda variable'larni qabul qilmaydi. Boshqa fayllar uchun
gate kerak bo'lsa, `infra-gate.yml`'ni o'z loyihangizga fork qiling.

## Retsept'lar

### Backend xizmati — 2 ta tasdiq, kim bo'lsa ham

```
MR_APPROVAL_MIN_TOTAL=2
```

### Bir xil qoida ham staging'da ham master'da

```
MR_APPROVAL_TARGET_PATTERN=/^(staging|master)$/
MR_APPROVAL_MIN_TOTAL=2
```

Staging o'zining sifat darajasi bo'lganda foydali (masalan backend'lar
`feature → staging` orqali QA'ga, keyin `staging → master` orqali prod'ga
ketadi) — peer review ikkala bosqichda ham talab qilinadi.

### Mobil jamoa — ikkita lead va plus 1 ta boshqa dev

```
MR_APPROVAL_REQUIRED_USERS="lead-1 lead-2"
MR_APPROVAL_REQUIRED_USERS_MIN=2
MR_APPROVAL_MIN_OTHERS=1
```

### Kompliance uchun muhim repo — 3 ta xavfsizlik reviewer'idan 1 ta + 2 ta boshqa

```
MR_APPROVAL_REQUIRED_USERS="sec-alice sec-bob sec-carol"
MR_APPROVAL_REQUIRED_USERS_MIN=1
MR_APPROVAL_MIN_OTHERS=2
```

### Infra-gated repo — faqat Dockerfile/CI/helm o'zgarishlari uchun infra approval

```
MR_APPROVAL_INFRA_USERS="infra-lead-1 infra-lead-2"
MR_APPROVAL_INFRA_MIN=1
```

Faqat kod o'zgarishlari bo'lgan MR oddiy team approval bilan o'tadi.
Kimdir `Dockerfile`, `.gitlab-ci.yml`, `entrypoint.sh` yoki `helm/`
papkasiga tegishi bilanoq ikkinchi job ishga tushadi va infra-egasidan
tasdiq talab qiladi (muallifning o'zi pulda bo'lsa, hisoblanadi).

## Korporativ proxy ortida

Agar sizning GitLab runner'laringiz tashqi internetga to'g'ridan-to'g'ri
chiqa olmasa (enterprise muhitlarda tez-tez uchraydi), `apk add curl jq`
qadami xato beradi. Quyidagini sozlang:

```
MR_APPROVAL_HTTP_PROXY=http://sizning-corp-proxy:3128
MR_APPROVAL_NO_PROXY=.sizning-gitlab-domain.example,localhost,127.0.0.1
```

> **`NO_PROXY` sintaksisi nozik joyi:** curl'ning `no_proxy` *suffix-match*
> ishlatadi va `*` belgisini literal harf sifatida ko'radi (agar u butun
> qiymat bo'lmasa). Ya'ni `*.example.com` `gitlab.example.com`'ga **mos
> kelmaydi** — buning o'rniga `.example.com` (boshida nuqta) ishlating.
> CIDR diapazonlari (`10.0.0.0/8`) ham ishlamaydi — aniq hostname'lardan
> foydalaning.

Agar runner Docker Hub'ga ham chiqa olmasa, `alpine:3.20`'ni o'zingizning
ichki registry'ingizga mirror qiling va override qiling:

```
MR_APPROVAL_IMAGE=registry.internal.example/library/alpine:3.20
```

## Qanday ishlaydi

GitLab CE loyihalari approval rules'ni UI orqali sozlay olishadi (Settings
→ Merge requests → Approvals), lekin bu qoidalar majburiylashtirilmaydi —
merge huquqiga ega har bir kishi tasdiqlar holatidan qat'i nazar merge
qila oladi. Ammo CE'dagi `/approvals` API endpoint kim "Approve" tugmasini
bosgan bo'lsa, ularni hali ham qaytaradi, shuning uchun:

1. Har bir MR pipeline'da gate job `GET /merge_requests/:iid` chaqirib
   muallifni aniqlaydi, keyin `GET /merge_requests/:iid/approvals`
   chaqirib kim tasdiqlaganini bilib oladi.
2. Tasdiqchilar ro'yxatini sozlangan qoidalar bilan solishtiradi va
   qoidalar bajarilmasa nolga teng bo'lmagan kod bilan chiqadi.
3. "Pipelines must succeed" branch protection bilan birga, GitLab gate
   job yashil bo'lmaguncha Merge tugmasini yoqishdan bosh tortadi.

Yangi commit push qilish pipeline'ni qaytadan ishga tushiradi va gate'ni
*joriy* tasdiqlar holati bo'yicha qayta baholaydi, shuning uchun pipeline
o'rtasida berilgan tasdiqlar keyingi ishga tushishda kuchga kiradi.

## Cheklovlar va FAQ

**Savol: Nima uchun `team-gate.yml` muallifni har doim hisobdan chiqaradi?**
Javob: Team gate *peer review* (hamkasblar tomonidan tekshirish)ni
majburiylash uchun. O'z-o'zini tekshirish peer review emas. Biz muallifni
GitLab'ning "Prevent author approval" sozlamasidan **qat'i nazar** olib
tashlaymiz, chunki bu sozlama har bir loyiha uchun alohida bo'lishi
mumkin va o'chirilgan bo'lishi mumkin.

**Savol: Nima uchun `infra-gate.yml` muallif infra-egasi bo'lsa uni hisoblaydi?**
Javob: Boshqa maqsad. Infra-gate "o'zgarishga kamida N ta malakali ko'z"
qoidasini majburiylashadi. Agar muallifning o'zi malakali bo'lsa, uning
mualliflik qilishi tasdiqlash sifatida hisoblanadi — DevOps muhandis bir
qatorlik Dockerfile o'zgarishi uchun boshqa DevOps'dan "Approve" bosishini
talab qilmasligi kerak.

**Savol: Token yashirgan bot foydalanuvchisi tasdiqchi bo'la oladimi?**
Javob: Token foydalanuvchisi hech narsani tasdiqlamaydi — u faqat
tasdiqlar ro'yxatini o'qiydi. Haqiqiy tasdiqchilar — "Approve" tugmasini
bosgan odamlar.

**Savol: Infra gate uchun boshqa fayl ro'yxatini xohlayman.**
Javob: GitLab `rules:changes:paths:` ro'yxatlarida variable'larni qabul
qilmaydi, shuning uchun ro'yxat qattiq qatlangan. Faylni fork qilib
o'zingiz tahrirlang.

**Savol: Gate "barcha discussion'lar yopilgan" ni ham majburiylashtira oladimi?**
Javob: GitLab'ning o'z "All discussions must be resolved before merge"
sozlamasi buni nativ qiladi. Settings → Merge requests'da yoqing.
Gate'da takrorlash shart emas.

**Savol: Token muddati tugasa nima bo'ladi?**
Javob: Gate job `curl 401` xatosi bilan ishlamay qoladi. Settings →
Access tokens'da tokenni rotate qiling, yangi qiymatni CI/CD variable'ga
yopishtiring. Tugashidan bir hafta oldin kalendar eslatmasini qo'yish
foydali.

## Hissa qo'shish

Issue va pull request'lar mamnuniyat bilan kutiladi — ayniqsa:
- Variable sxemasini yaxshilash
- Boshqa CI provayderlar uchun qo'llab-quvvatlash (Bitbucket, Forgejo, Gitea)
- Test'lar
- Yaxshiroq retsept'lar / hujjatlar

## Litsenziya

[MIT](LICENSE) — xohlaganingizni qiling, kafolat yo'q.
