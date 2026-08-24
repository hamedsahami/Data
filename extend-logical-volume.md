# افزایش فضای `/` در Ubuntu با استفاده از دیسک جدید و LVM

## هدف

در این سناریو یک دیسک جدید با ظرفیت **100GB** به ماشین مجازی Ubuntu اضافه شده بود و هدف این بود که فضای آن به پارتیشن ریشه (`/`) اضافه شود، بدون اینکه `/var` و `swap` تغییر کنند.

ساختار اولیه سیستم:

```text
sda = 40G
├─ sda1 = 1G      /boot/efi
├─ sda2 = 2G      /boot
└─ sda3 = 36.9G  LVM
   ├─ root = 18.5G  /
   ├─ swap = 4G
   └─ var  = 14.5G /var

sdb = 100G
```

در پایان:

```text
sda = 40G
├─ sda1 = 1G      /boot/efi
├─ sda2 = 2G      /boot
└─ sda3 = 36.9G  LVM

sdb = 100G
└─ اضافه‌شده به LVM

ubuntu-vg
├─ root = 118.5G  /
├─ swap = 4G
└─ var  = 14.5G  /var
```

---

# 1. بررسی دیسک‌ها

اول باید ببینیم Ubuntu چه دیسک‌ها و پارتیشن‌هایی دارد.

```bash
lsblk
```

خروجی اولیه ما:

```text
NAME                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                   8:0    0    40G  0 disk
├─sda1                8:1    0     1G  0 part /boot/efi
├─sda2                8:2    0     2G  0 part /boot
└─sda3                8:3    0  36.9G  0 part
  ├─ubuntu--vg-root 252:0    0  18.5G  0 lvm  /
  ├─ubuntu--vg-swap 252:1    0     4G  0 lvm  [SWAP]
  └─ubuntu--vg-var  252:2    0  14.5G  0 lvm  /var

sdb                   8:16   0   100G  0 disk
```

### نکته مهم

در این مرحله:

```text
sdb = 100G
```

دیسک جدید است ولی هنوز در LVM استفاده نشده است.

همچنین مشخص شد که سیستم از **LVM** استفاده می‌کند.

---

# 2. بررسی Physical Volume

برای دیدن Physical Volumeهای موجود:

```bash
sudo pvs
```

خروجی:

```text
PV         VG        Fmt  Attr PSize   PFree
/dev/sda3  ubuntu-vg lvm2 a--  <36.95g    0
```

یعنی:

```text
/dev/sda3
     ↓
ubuntu-vg
```

و نکته مهم این بود که:

```text
PFree = 0
```

یعنی Volume Group فضای آزاد نداشت.

---

# 3. بررسی Volume Group

دستور:

```bash
sudo vgs
```

خروجی:

```text
VG        #PV #LV #SN Attr   VSize   VFree
ubuntu-vg   1   3   0 wz--n- <36.95g    0
```

معنی:

```text
#PV = 1
```

یعنی فقط یک Physical Volume داریم:

```text
/dev/sda3
```

و:

```text
#LV = 3
```

یعنی سه Logical Volume داریم:

```text
root
swap
var
```

و:

```text
VFree = 0
```

یعنی فضای آزاد در `ubuntu-vg` وجود نداشت.

---

# 4. بررسی Logical Volumeها

دستور:

```bash
sudo lvs
```

خروجی:

```text
LV   VG        Attr       LSize
root ubuntu-vg -wi-ao---- 18.47g
swap ubuntu-vg -wi-ao----  4.00g
var  ubuntu-vg -wi-ao---- 14.47g
```

ساختار:

```text
ubuntu-vg
│
├── root  18.47G → /
├── swap   4.00G → SWAP
└── var   14.47G → /var
```

هدف ما این بود که فقط:

```text
root
```

را بزرگ کنیم.

بنابراین:

* `swap` دست‌نخورده باقی می‌ماند.
* `/var` دست‌نخورده باقی می‌ماند.
* فضای جدید به `root` اضافه می‌شود.

---

# 5. ساخت Physical Volume روی دیسک جدید

دیسک جدید:

```text
/dev/sdb
```

را به یک LVM Physical Volume تبدیل کردیم:

```bash
sudo pvcreate /dev/sdb
```

این دستور می‌گوید:

> `/dev/sdb` قرار است به‌عنوان Physical Volume در LVM استفاده شود.

بعد می‌توان با دستور زیر بررسی کرد:

```bash
sudo pvs
```

ساختار جدید تقریباً به این شکل می‌شود:

```text
/dev/sda3 ──┐
            ├── ubuntu-vg
/dev/sdb  ──┘
```

---

# 6. اضافه کردن دیسک جدید به Volume Group

Volume Group ما:

```text
ubuntu-vg
```

است.

بنابراین `/dev/sdb` را به آن اضافه کردیم:

```bash
sudo vgextend ubuntu-vg /dev/sdb
```

معنی دستور:

```text
/dev/sdb
    ↓
Physical Volume
    ↓
ubuntu-vg
```

بعد از این مرحله Volume Group شامل فضای قبلی و فضای جدید می‌شود.

قبل:

```text
ubuntu-vg
└── 36.9G
```

بعد:

```text
ubuntu-vg
├── /dev/sda3 ≈ 36.9G
└── /dev/sdb  = 100G
```

در نتیجه تقریباً:

```text
ubuntu-vg ≈ 136.9G
```

فضا دارد.

---

# 7. افزایش Logical Volume مربوط به `/`

Logical Volume مربوط به root:

```text
/dev/ubuntu-vg/root
```

است.

برای استفاده از تمام فضای آزاد Volume Group:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/root
```

قسمت مهم:

```text
+100%FREE
```

یعنی:

> تمام فضای آزاد موجود در Volume Group را به این Logical Volume اضافه کن.

بنابراین:

```text
root
18.5G
  +
100G
  ↓
≈118.5G
```

بعد از این مرحله `lsblk` نشان داد:

```text
ubuntu--vg-root  118.5G  /
```

و همچنین:

```text
sdb
└─ubuntu--vg-root 118.5G /
```

این یعنی فضای `sdb` اکنون بخشی از Logical Volume مربوط به `/` است.

---

# 8. تفاوت LV و Filesystem

یک نکته بسیار مهم در LVM:

وقتی این دستور را اجرا می‌کنیم:

```bash
sudo lvextend ...
```

در واقع **Logical Volume** بزرگ می‌شود.

اما این الزاماً به معنی بزرگ شدن Filesystem نیست.

مثلاً ممکن است:

```text
LV
118.5G
```

باشد ولی:

```text
Filesystem
18.5G
```

باقی مانده باشد.

بنابراین باید Filesystem را نیز بررسی کنیم.

---

# 9. بررسی Filesystem

برای دیدن نوع Filesystem و اندازه واقعی `/`:

```bash
df -Th /
```

مثلاً ممکن است خروجی چیزی شبیه این باشد:

```text
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-root ext4  116G   10G  100G  10% /
```

قسمت مهم:

```text
Type
```

است.

اگر:

```text
ext4
```

باشد، از `resize2fs` استفاده می‌کنیم.

اگر:

```text
xfs
```

باشد، از `xfs_growfs` استفاده می‌کنیم.

---

# 10. افزایش Filesystem در ext4

اگر Filesystem از نوع `ext4` باشد:

```bash
sudo resize2fs /dev/ubuntu-vg/root
```

این دستور Filesystem را تا اندازه Logical Volume بزرگ می‌کند.

در نتیجه:

```text
LV:
18.5G → 118.5G

Filesystem:
18.5G → حدود 118.5G
```

---

# 11. افزایش Filesystem در XFS

اگر Filesystem از نوع XFS باشد، به‌جای `resize2fs` باید از:

```bash
sudo xfs_growfs /
```

استفاده کرد.

پس:

| Filesystem | دستور                                |
| ---------- | ------------------------------------ |
| ext4       | `sudo resize2fs /dev/ubuntu-vg/root` |
| xfs        | `sudo xfs_growfs /`                  |

---

# 12. بررسی نهایی

بعد از تمام مراحل:

```bash
lsblk
```

و:

```bash
df -h /
```

را اجرا می‌کنیم.

همچنین برای بررسی LVM:

```bash
sudo pvs
sudo vgs
sudo lvs
```

---

# نتیجه نهایی ما

در ابتدا:

```text
/dev/sda = 40G
/dev/sdb = 100G

root = 18.5G
```

در پایان:

```text
/dev/sda = 40G
/dev/sdb = 100G

ubuntu-vg
│
├── root ≈ 118.5G → /
├── var   ≈ 14.5G → /var
└── swap   = 4G
```

یعنی **100GB دیسک جدید بدون تغییر `/var` و `swap` به Logical Volume مربوط به `/` اضافه شد.**

---

# خلاصه دستورات

اگر همین سناریو را روی یک VM مشابه انجام بدهیم:

```bash
# 1. مشاهده دیسک‌ها
lsblk

# 2. بررسی LVM
sudo pvs
sudo vgs
sudo lvs

# 3. تبدیل دیسک جدید به Physical Volume
sudo pvcreate /dev/sdb

# 4. اضافه کردن به Volume Group
sudo vgextend ubuntu-vg /dev/sdb

# 5. بررسی فضای آزاد
sudo vgs

# 6. اضافه کردن تمام فضای آزاد به root
sudo lvextend -l +100%FREE /dev/ubuntu-vg/root

# 7. بررسی نوع filesystem
df -Th /

# 8. اگر ext4 بود
sudo resize2fs /dev/ubuntu-vg/root

# 9. بررسی نهایی
df -h /
lsblk
```

> ⚠️ **نکته ایمنی مهم:** قبل از اجرای `pvcreate` روی هر دیسکی، حتماً با `lsblk` مطمئن شو که دیسک موردنظر واقعاً دیسک جدید و خالی است. اجرای `pvcreate` روی دیسک اشتباه می‌تواند اطلاعات آن را از بین ببرد.

### مدل ذهنی LVM

برای اینکه ساختار LVM را راحت‌تر به خاطر بسپاری:

```text
Physical Disk
     │
     ▼
Partition / Disk
     │
     ▼
Physical Volume (PV)
     │
     ▼
Volume Group (VG)
     │
     ├──────────────┐
     ▼              ▼
Logical Volume   Logical Volume
     │              │
     ▼              ▼
Filesystem       Filesystem
     │              │
     ▼              ▼
     /             /var
```

در پروژه خودمان:

```text
/dev/sda3 ───────┐
                 │
                 ▼
             ubuntu-vg
                 ▲
                 │
/dev/sdb ────────┘
                 │
                 ▼
        /dev/ubuntu-vg/root
                 │
                 ▼
                 /
```

**نکته کلیدی:** در LVM معمولاً زنجیره‌ی کار این است:

**Disk → PV → VG → LV → Filesystem → Mount Point**

یعنی اگر فضای دیسک جدید را اضافه کردی، باید بدانی در کدام لایه هستی؛ صرفاً بزرگ شدن Disk به معنی بزرگ شدن `/` نیست.
