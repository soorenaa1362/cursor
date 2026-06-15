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

## ۲. بخش save — ذخیره کد در samen_personell

**فایل:** `personell_register.php`

**قبل از** `$personell_obj->setDataArray($_POST);` (هم در update و هم insert):

```php
// کد تخصص را در nd_subtype ذخیره کن، نه نام نمایشی
if (!empty($_POST['nd_subtype_code'])) {
    $_POST['nd_subtype'] = $_POST['nd_subtype_code'];
} elseif ($update || $personell_nr) {
    // اگر کاربر تخصص را عوض نکرد، مقدار قبلی DB را نگه دار
    $old_nd = $personell_obj->personell_data['nd_subtype'] ?? '';
    if (!empty($old_nd)) {
        $_POST['nd_subtype'] = $old_nd;
    }
}
unset($_POST['nd_subtype_code']); // فیلد DB نیست
```

برای update، `$personell_obj->loadPersonellData` قبلاً اجرا شده؛ اگر نه، قبل از save یک بار load کنید یا مقدار را از query بگیرید:

```php
$old_nd_subtype = '';
if ($personell_nr) {
    $sql_old = "SELECT nd_subtype FROM samen_personell WHERE nr=" . (int)$personell_nr . " LIMIT 1";
    $rs_old = $db->Execute($sql_old);
    if ($rs_old && $rs_old->RecordCount()) {
        $old_nd_subtype = $rs_old->fields['nd_subtype'];
    }
}
// ...
} elseif (!empty($old_nd_subtype)) {
    $_POST['nd_subtype'] = $old_nd_subtype;
}
```

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

| مرحله | منبع | فیلد نمایش | فیلد hidden |
|--------|------|------------|-------------|
| بارگذاری | personell.nd_subtype | نام از specialty_data | کد |
| بارگذاری (خالی) | senddocters | subtype | subtype_nr |
| انتخاب از لیست | JS | value | code |
| ذخیره | POST | — | nd_subtype = code |
