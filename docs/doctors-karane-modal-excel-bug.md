# باگ: کلیک × مدال «پزشکان محاسبه نشده» → دانلود اکسل

## علت

دکمه بستن مدال بدون `type="button"` تعریف شده:

```html
<button onclick="document.getElementById('notCalcModal').style.display='none';">&times;</button>
```

در HTML، `<button>` پیش‌فرض `type="submit"` است. اگر این دکمه (مستقیم یا به‌خاطر HTML نامعتبر) داخل فرمی باشد که `action="showExcel.php"` دارد، کلیک روی × همان فرم را submit می‌کند و فایل اکسل دانلود می‌شود.

فرم مشکل‌ساز:

```php
echo '<form action="showExcel.php" method="post" name="resultform">';
```

## راه‌حل

### ۱. حتماً `type="button"` بگذارید

```html
<button type="button"
        onclick="closeNotCalcModal();"
        style="background:none;border:none;color:#fff;font-size:20px;cursor:pointer;">
    &times;
</button>
```

```javascript
function closeNotCalcModal(e){
    if(e){
        e.preventDefault();
        e.stopPropagation();
    }
    document.getElementById('notCalcModal').style.display = 'none';
}
```

### ۲. مدال را بیرون از همه `<form>`ها بگذارید

ترتیب درست در انتهای `<body>`:

```html
</form> <!-- resultform بسته شده -->
</div>  <!-- page-wrap -->

<div id="notCalcModal">...</div>
</body>
```

مدال نباید بین `<form name="resultform">` و `</form>` باشد.

### ۳. ساختار div/form را درست کنید

در بلوک PHP انتهای گزارش، `</form>` باید **بعد از** بستن divهای layout و **قبل از** اسکریپت/مدال باشد:

```php
    echo '</table>';
    echo '<input type="hidden" id="checked_centers" value="'.$checked_centers.'">';
    echo '<textarea name="sql_s" id="sql_s" style="display:none;">'.$sql.'</textarea>';
}
else {
    echo '<div class="alert alert-danger">موردی ثبت نشده است</div>';
}

echo '</div></div></div></div>'; // بستن layout
echo '<input type="hidden" value="" name="excelValue" />';
echo '</form>'; // بستن resultform
} // پایان if mode_search
?>
<script>...</script>

<!-- مدال اینجا، بعد از بسته شدن همه formها -->
<div id="notCalcModal">...</div>
```

اگر `</form>` دیر بسته شود، مرورگر ممکن است اسکریپت و مدال را داخل فرم نگه دارد.

## خلاصه

| مشکل | راه‌حل |
|------|--------|
| دکمه × بدون type | `type="button"` |
| مدال داخل resultform | جابجایی بعد از `</form>` |
| submit ناخواسته | `preventDefault()` در onclick |
