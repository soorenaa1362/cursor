# تغییرات nd_subtype — راهنمای دقیق

## مشکل فعلی

1. کوئری `samen_senddocters` فقط `subtype` را می‌گیرد، نه `subtype_nr` → کد hidden خالی می‌ماند
2. اولویت از `samen_personell.nd_subtype` چک نمی‌شود
3. در ذخیره، `nd_subtype` نام تخصص ذخیره می‌شود نه کد
4. اعتبارسنجی فقط شماره نظام را چک می‌کند، نه تخصص

---

## ۱. بخش GUI — نمایش تخصص (جایگزین بلوک takhasos)

**فایل:** `gui_bridge/default/gui_personell_register.php` (یا همان فایل GUI)

**پیدا کنید:**
```php
$sp_show = '';
if (!empty($job_nezam_nr)) {
    $sql_sp = "SELECT subtype FROM samen_senddocters WHERE nezam_nr='" . addslashes($job_nezam_nr) . "' LIMIT 1";
```

**جایگزین کنید با:**
```php
$sp_show = '';
$sp_code_val = '';

// اولویت ۱: از samen_personell.nd_subtype (کد ذخیره‌شده)
if (!empty($nd_subtype)) {
    $sp_code_val = $nd_subtype;
    if (isset($specialty_data[$nd_subtype])) {
        $sp_show = $specialty_data[$nd_subtype];
    } else {
        // اگر کد در لیست نبود، خود کد را نشان بده
        $sp_show = $nd_subtype;
    }
}

// اولویت ۲: اگر خالی بود، از senddocters
if ($sp_code_val === '' && !empty($job_nezam_nr)) {
    $sql_sp = "SELECT subtype, subtype_nr FROM samen_senddocters 
               WHERE nezam_nr='" . addslashes($job_nezam_nr) . "' LIMIT 1";
    $rs_sp = $db->Execute($sql_sp);
    if ($rs_sp && $rs_sp->RecordCount()) {
        $sp_show = $rs_sp->fields['subtype'];
        $sp_code_val = $rs_sp->fields['subtype_nr'];
    }
}
```

**نکته:** `$specialty_data` باید **قبل** از این بلوک لود شده باشد (همان `$info_obj->getSpecial(1, true)` که دارید).

---

## ۲. بخش save — ذخیره در **هر دو** جدول

**فایل:** `personell_register.php`

تخصص باید همزمان در `samen_senddocters` و `samen_personell` ذخیره شود. ترتیب مهم است:

1. اول `samen_senddocters` (با **نام** + **کد**)
2. بعد `samen_personell` (فقط **کد** در فیلد `nd_subtype`)

### ۲-الف. samen_senddocters — همان بلوک اول save (دست نزنید، فقط مطمئن شوید هست)

این بلوک **قبل از** validation و **قبل از** `setDataArray` اجرا می‌شود. باید بماند:

```php
if ($mode == 'save') {
    // ...
    $job_nezam_nr    = isset($_POST['job_nezam_nr']) ? trim($_POST['job_nezam_nr']) : '';
    $nd_subtype_code = isset($_POST['nd_subtype_code']) ? $_POST['nd_subtype_code'] : '';
    $nd_subtype_name = isset($_POST['nd_subtype']) ? trim($_POST['nd_subtype']) : '';

    if (!empty($job_nezam_nr)) {
        $sql_chk = "SELECT nr FROM samen_senddocters 
                    WHERE nezam_nr='" . addslashes($job_nezam_nr) . "' LIMIT 1";
        $chk = $db->Execute($sql_chk);
        // ...
        if ($chk->RecordCount() > 0) {
            $sql = "UPDATE samen_senddocters SET
                name='" . addslashes($name_last) . "',
                fname='" . addslashes($name_first) . "',
                subtype='" . addslashes($nd_subtype_name) . "',
                subtype_nr='" . addslashes($nd_subtype_code) . "',
                sepas_code='" . addslashes($nd_subtype_code) . "',
                modify_id='" . $user . "',
                modify_time='" . $now . "'
                WHERE nezam_nr='" . addslashes($job_nezam_nr) . "'";
        } else {
            $sql = "INSERT INTO samen_senddocters
                (nezam_nr, name, fname, subtype, subtype_nr, sepas_code, create_id, create_time)
                VALUES (..., '" . addslashes($nd_subtype_name) . "', '" . addslashes($nd_subtype_code) . "', ...)";
        }
        $db->Execute($sql);
    }
}
```

| جدول | فیلد | مقدار |
|------|------|-------|
| `samen_senddocters` | `subtype` | نام تخصص (متن input) |
| `samen_senddocters` | `subtype_nr` | کد (`nd_subtype_code`) |
| `samen_senddocters` | `sepas_code` | کد (`nd_subtype_code`) |

### ۲-ب. samen_personell — قبل از setDataArray

**بعد از** بلوک senddocters و **قبل از** `$personell_obj->setDataArray($_POST);` (هم update هم insert):

```php
// نام نمایشی را نگه دار (senddocters قبلاً با همین ذخیره شده)
$nd_subtype_name = isset($_POST['nd_subtype']) ? trim($_POST['nd_subtype']) : '';
$nd_subtype_code = isset($_POST['nd_subtype_code']) ? trim($_POST['nd_subtype_code']) : '';

// کد را در samen_personell.nd_subtype ذخیره کن
if ($nd_subtype_code !== '') {
    $_POST['nd_subtype'] = $nd_subtype_code;
} elseif ($personell_nr) {
    // کاربر تخصص را عوض نکرد → مقدار قبلی personell را نگه دار
    $sql_old = "SELECT nd_subtype FROM samen_personell WHERE nr=" . (int)$personell_nr . " LIMIT 1";
    $rs_old = $db->Execute($sql_old);
    if ($rs_old && $rs_old->RecordCount() && $rs_old->fields['nd_subtype'] !== '') {
        $_POST['nd_subtype'] = $rs_old->fields['nd_subtype'];
    }
}

unset($_POST['nd_subtype_code']); // فقط فیلد فرم است، ستون DB نیست
```

| جدول | فیلد | مقدار |
|------|------|-------|
| `samen_personell` | `nd_subtype` | کد تخصص |

**چرا ترتیب مهم است:** بلوک senddocters از `$_POST['nd_subtype']` به‌عنوان **نام** استفاده می‌کند. اگر قبل از آن `nd_subtype` را به کد تبدیل کنید، نام اشتباه در senddocters ذخیره می‌شود.

---

## ۳. JavaScript — اعتبارسنجی تخصص با notyf

**جایگزین بلوک submit فعلی:**

```javascript
aufnahmeForm.addEventListener('submit', function(e){
    var classNumber = document.querySelector('[name="class_number"]');
    var ndSubtype = document.querySelector('#nd_subtype');
    var ndSubtypeCode = document.querySelector('#nd_subtype_code');
    var nezam = document.getElementById('job_nezam_nr_input');
    if (!classNumber) return;

    var selectedOpt = classNumber.options[classNumber.selectedIndex];
    var studyType = selectedOpt ? selectedOpt.getAttribute('data-type') : '';

    if (parseInt(studyType, 10) >= 2) {
        if (!ndSubtypeCode || !ndSubtypeCode.value.trim()) {
            e.preventDefault();
            if (typeof notyf !== 'undefined') {
                notyf.warning('تخصص را وارد کنید');
            } else {
                alert('تخصص را وارد کنید');
            }
            if (ndSubtype) {
                ndSubtype.focus();
                ndSubtype.style.border = '2px solid red';
            }
            return;
        }
    }

    if (nezam && parseInt(studyType, 10) >= 2 && (!nezam.value || nezam.value.trim() === '')) {
        e.preventDefault();
        if (typeof notyf !== 'undefined') {
            notyf.warning('شماره نظام را وارد نمایید');
        } else {
            alert('شماره نظام را وارد نمایید');
        }
        nezam.focus();
        nezam.style.border = '2px solid red';
    }
});

if (ndSubtype) {
    ndSubtype.addEventListener('input', function () {
        this.style.border = '';
    });
}
```

---

## ۴. حالت نمایش (show_file)

در بلوک `if ($show_file || $path == 'mali')` برای تخصص:

```php
if ($show_file || $path == 'mali') {
    echo htmlspecialchars($sp_show);
}
```

همان `$sp_show` و `$sp_code_val` از بلوک اول ساخته می‌شود — نیازی به تغییر جداگانه نیست.

---

## خلاصه جریان

### بارگذاری

| مرحله | منبع | فیلد نمایش | فیلد hidden |
|--------|------|------------|-------------|
| اولویت ۱ | `personell.nd_subtype` (کد) | نام از `specialty_data` | همان کد |
| اولویت ۲ | `senddocters` | `subtype` | `subtype_nr` |
| انتخاب از لیست | JS | `value` | `code` |

### ذخیره (هر دو جدول)

| جدول | فیلد | مقدار |
|------|------|-------|
| `samen_senddocters` | `subtype` | نام (`nd_subtype` از فرم) |
| `samen_senddocters` | `subtype_nr` / `sepas_code` | کد (`nd_subtype_code`) |
| `samen_personell` | `nd_subtype` | کد (`nd_subtype_code`) |

شرط senddocters: `job_nezam_nr` خالی نباشد.
