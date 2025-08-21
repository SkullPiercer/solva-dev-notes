Иногда пользователи теряют пароль и оказываются без доступа к личному кабинету. Чтобы восстановить вход, в Django предусмотрен встроенный механизм сброса пароля. Важная часть этой системы — автоматическая отправка электронных писем.

Процесс работает так:

1. Пользователь вводит свой e-mail в специальной форме восстановления.
    
2. Если адрес совпадает с тем, что хранится в базе данных, Django формирует письмо со специальной ссылкой.
    
3. Перейдя по ссылке, владелец почты может назначить новый пароль.

Чтобы всё это работало, в проекте необходимо настроить отправку писем.

---

## Почтовые бэкенды в Django

Под «бэкендом» здесь понимается отдельный механизм, который решает задачу доставки писем. Django предоставляет несколько вариантов:

- **SMTP-бэкенд (по умолчанию)** — реальная отправка писем через внешний почтовый сервер (например, Gmail, Яндекс или корпоративный SMTP).
    
- **Консольный бэкенд** — письма не отправляются, а печатаются прямо в консоль (удобно при разработке).
    
- **Файловый бэкенд** — письма сохраняются как текстовые файлы в указанной директории проекта.
    
- **In-memory бэкенд** — письма складываются в список `django.core.mail.outbox`, который доступен только во время работы приложения.
    
- **Заглушка (Dummy)** — ничего не делает, письма полностью игнорируются. Используется для отключения отправки без изменения кода.

Для разработки реальный SMTP подключать необязательно, поэтому в проекте **Task** мы будем использовать **файловый бэкенд**.

---

## Настройка сохранения писем в файлы

В файле `settings.py` укажем бэкенд и путь, куда сохранять письма:
**settings.py**
```
EMAIL_BACKEND = 'django.core.mail.backends.filebased.EmailBackend'
EMAIL_FILE_PATH = BASE_DIR / 'sent_emails'
```

Теперь каждое «отправленное» письмо будет сохраняться в папке `sent_emails/`.  
Если директории ещё нет — Django создаст её при первой отправке.

---
## Отправка писем из кода

Для примера воспользуемся функцией `send_mail()` из модуля `django.core.mail`:
```
from django.core.mail import send_mail

send_mail(
    subject='Новый комментарий к задаче',
    message='К вашей задаче добавлен комментарий.',
    from_email='noreply@task.local',
    recipient_list=['user@example.com'],
    fail_silently=True,
)
```

Параметры:

- **subject** — тема письма.
    
- **message** — текст сообщения.
    
- **from_email** — отправитель (если не указать, будет использоваться `DEFAULT_FROM_EMAIL`).
    
- **recipient_list** — список получателей.
    
- **fail_silently** — если `True`, то ошибка при отправке не прервёт выполнение программы.

---

## Пример: уведомления в проекте Task

Допустим, в приложении есть форма создания задачи. Если пользователь пытается зарегистрировать задачу с названием `"Test Task"`, мы отправим администратору письмо.
```
# tasks/forms.py
from django import forms
from django.core.mail import send_mail
from .models import Task

class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description']

    def clean_title(self):
        title = self.cleaned_data['title']
        if title == "Test Task":
            send_mail(
                subject='Попытка создать тестовую задачу',
                message=f'Пользователь попробовал завести задачу с именем "{title}"',
                from_email='task_form@task.local',
                recipient_list=['admin@task.local'],
                fail_silently=True,
            )
        return title
```

или прямо во вьюхе
```
def category_detail(request, category_id):
    template_name = 'notes/category_detail.html'
    category = Category.objects.get(id=category_id)
    context = {
        'category': category
    }
    send_mail(
        subject='Вы получили данные о категории',
        message='Вы довольны нашим сервисом?',
        from_email='qwerty@mail.ru',
        recipient_list=['user1@mail.ru', 'user2@mail.ru'],
        fail_silently=True,
    )
    return render(request, template_name, context)
```

После запуска проекта и сохранения такой формы в папке `sent_emails/` появится файл с содержимым письма.

Пример файла:
```
Subject: Попытка создать тестовую задачу
From: task_form@task.local
To: admin@task.local
Date: Thu, 21 Aug 2025 12:35:00 +0000

Пользователь попробовал завести задачу с именем "Test Task".
```

## Альтернатива
```
from django.core.mail import EmailMessage

def send_with_attachment(request):
    email = EmailMessage(
        subject="Отчёт",
        body="Во вложении файл.",
        from_email="your_email@gmail.com",
        to=["receiver@example.com"],
    )
    email.attach_file("report.pdf")
    email.send()
    return HttpResponse("Письмо с файлом отправлено!")
```
Даёт больше гибкости и позволяет прикреплять файлы
