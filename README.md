تمام ✅، خلينا نعملها بالراحة:

---

## 🎯 الهدف

تقفل أي **خروج للإنترنت (outbound)** من الحاويات اللي في رينج:

```
10.0.0.0/8
```

يعني كل IP يبدأ بـ `10.` (زي: 10.0.0.202, 10.0.0.203… إلخ).

---

## 🛠️ القاعدة في iptables

على السيرفر (Ubuntu host):

```bash
iptables -I DOCKER-USER -s 10.0.0.0/8 -o eth0 -j DROP
```

### معنى القاعدة:

* `-s 10.0.0.0/8` = أي حاوية IPها يبدأ بـ 10.*.*.\*
* `-o eth0` = تحاول تخرج على الإنترنت من الإنترفيس الخارجي.
* `-j DROP` = يتمنع.

---

## 🔍 للتحقق

شوف القاعدة انضافت:

```bash
iptables -L DOCKER-USER -n --line-numbers
```

---

## 💾 للحفظ بعد إعادة التشغيل

```bash
netfilter-persistent save
```

أو:

```bash
iptables-save > /etc/iptables/rules.v4
```

---

## 📌 ملاحظات

* كده أي حاوية جوه رينج 10.0.0.0/8 مش هتطلع إنترنت.
* بس لسه يقدروا يتواصلوا داخليًا بين بعض (داخل نفس شبكة Docker).
* لو عايز تعمل استثناء (مثلاً Odoo يطلع على 443) لازم تضيف قاعدة سماح قبل قاعدة الـ DROP.

---

تحب أجهزلك سكربت فيه:

* قفل النت عن 10.0.0.0/8 بالكامل.
* مع إمكانية إضافة استثناءات (بورتات أو حاويات محددة)؟







تمام ✅، خليني أديك ملخص نهائي مرتب تقدر تحفظه كـ **Documentation** عندك:

---








تمام — ده **ملخص مختصر وحاسم** للي عملناه علشان تحفظه عندك 👇

---

# 🎯 الهدف

* قفل **خروج** كل الحاويات للإنترنت (Outbound).
* السماح فقط لشبكة **Mailcow** بالخروج على المنافذ الضرورية.
* إبقاء **الدخول** لخدماتك المنشورة (Portainer على 9505، Odoo على 8069/8072) شغال طبيعي.

---

# 🧩 المشكلة اللي ظهرت

* إضافة قاعدة **DROP عامة** في `DOCKER-USER` كسّرت Portainer (والاتصالات الراجعة).
* كمان Portainer ما فتحش لأن السماح كان في `DOCKER-USER` بدل ما يكون في **INPUT** (الدخول من الإنترنت).

**العلاج:**

* إضافة قاعدة **ESTABLISHED,RELATED** أول `DOCKER-USER` للسماح بالـ replies.
* نقل/إضافة السماح لـ 9505/8069/8072 في **INPUT** (دخول)، مش في `DOCKER-USER`.

---

# ✅ الحالة النهائية (المنطق الفعّال)

**DOCKER-USER (للخروج فقط):**

```bash
# 1) اسمح بردود الجلسات القائمة (مهم جدًا)
iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 2) Mailcow subnet فقط (مثال: 172.22.1.0/24) تخرج على المنافذ الضرورية
iptables -I DOCKER-USER 2 -s 172.22.1.0/24 -o eth0 -p udp --dport 53 -j ACCEPT
iptables -I DOCKER-USER 3 -s 172.22.1.0/24 -o eth0 -p tcp --dport 53 -j ACCEPT
iptables -I DOCKER-USER 4 -s 172.22.1.0/24 -o eth0 -p tcp -m multiport --dports 25,465,587,443 -j ACCEPT

# 3) قفل خروج باقي الحاويات
iptables -A DOCKER-USER -o eth0 -j DROP
```

> ملاحظة: السماح لـ **9505** داخل `DOCKER-USER` **مش لازم**؛ ده نتحكم فيه من INPUT (دخول).

**INPUT (للدخول من الإنترنت للسيرفر):**

```bash
# Portainer
iptables -I INPUT 1 -p tcp --dport 9505 -j ACCEPT
# Odoo
iptables -I INPUT 1 -p tcp --dport 8069 -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 8072 -j ACCEPT
# (اختياري) قصّرها على IP معيّن:
# iptables -I INPUT 1 -p tcp -s <YOUR_IP> --dport 9505 -j ACCEPT
```

---

# 🔎 التحقق السريع

```bash
# راجع القواعد والعدادات
iptables -L DOCKER-USER -n -v --line-numbers
iptables -L INPUT -n -v --line-numbers

# تأكد Portainer شغال محليًا
ss -tlnp | grep 9505
curl -I http://localhost:9505
```

* من حاوية غير مسموح لها جرّب:

  ```bash
  docker exec -it <ctr> curl -I https://www.google.com  # المفروض يفشل
  ```
* من حاوية Mailcow جرّب 443/53/25/465/587 → تعمل.

---

# 💾 حفظ القواعد

```bash
netfilter-persistent save
# أو
iptables-save > /etc/iptables/rules.v4
```

---

# 📝 ملاحظات مهمة

* قاعدة `ESTABLISHED,RELATED` **آمنة ولا تفتح منافذ**؛ بتسمح بالـ replies فقط.
* **تجنّب** `DROP all` بدون `-o eth0` لأنه يكسر الدخول للخدمات المنشورة.
* لو عندك **Cloud Firewall** عند المزوّد، افتح 9505/8069/8072/443 هناك أيضًا.

---

ده الملخص التنفيذي: **Whitelist لخروج Mailcow فقط + DROP للخروج للباقي + INPUT سماح لبورتات Portainer/Odoo + قاعدة ESTABLISHED أول DOCKER-USER**.
