В классических Django-проектах для обработки запросов мы часто используем **view-функции**. Они принимают запрос, извлекают данные из базы, подготавливают контекст и отдают HTML-страницу. Но если вдуматься, эти действия очень похожи из проекта в проект:

- вывести список объектов,
    
- показать конкретную запись,
    
- отобразить форму для создания или изменения,
    
- удалить элемент.

Например, в приложении **task** у нас есть сущность "Задача". Пользователь может открыть список всех задач, просмотреть подробности конкретной, создать новую задачу, изменить существующую или удалить ненужную. Такие CRUD-операции встречаются в любом веб-приложении.

Чтобы не переписывать однотипные view-функции снова и снова, Django предоставляет инструмент — **Class-Based Views** (CBV), или представления на основе классов. Это готовые классы, которые умеют выполнять стандартные действия и экономят нам время.

---
### Принцип работы CBV

CBV делают то же самое, что и обычные view-функции: принимают запрос и возвращают ответ. Разница в том, что вместо написания кода вручную мы наследуемся от встроенного класса Django и настраиваем лишь необходимые параметры.

Хранятся такие классы в привычном файле `views.py`.  
Примеры встроенных CBV:

- `ListView` — список объектов,
    
- `DetailView` — одна запись,
    
- `CreateView` — создание новой записи,
    
- `UpdateView` — редактирование,
    
- `DeleteView` — удаление.

Чтобы подключить CBV в маршрутах (`urls.py`), используют метод `.as_view()`:
```
from django.urls import path
from . import views

urlpatterns = [
    # Для функции:
    path("list/", views.task_list, name="list"),

    # Для класса:
    path("list/", views.TaskListView.as_view(), name="list"),
]

```

---
### ListView — выводим список задач

Допустим, мы хотим показать пользователю все задачи. Раньше для этого писали функцию, делали выборку из базы и подключали пагинацию. С `ListView` всё короче:
```
# views.py
from django.views.generic import ListView
from .models import Task

class TaskListView(ListView):
    model = Task
    ordering = "id"
    paginate_by = 10
```

Django сам поймёт, какой шаблон подключить: по умолчанию это будет  
`task/task_list.html`. В контекст автоматически передаётся объект `page_obj`, с которым удобно работать в шаблоне.

---
### CreateView — создание новой задачи

Чтобы добавить задачу, мы можем использовать встроенный `CreateView`. Он сам сгенерирует форму на основе модели:
```
from django.views.generic import CreateView
from django.urls import reverse_lazy
from .models import Task

class TaskCreateView(CreateView):
    model = Task
    fields = "__all__"
    template_name = "task/task_form.html"
    success_url = reverse_lazy("task:list")
```

После успешного сохранения пользователя перенаправит на список задач.  
При желании можно подключить собственный `ModelForm` вместо автогенерации:
```
class TaskCreateView(CreateView):
    model = Task
    form_class = TaskForm
    success_url = reverse_lazy("task:list")
```

---
### UpdateView — редактирование

Редактирование задачи строится аналогично:
```
from django.views.generic import UpdateView

class TaskUpdateView(UpdateView):
    model = Task
    form_class = TaskForm
    template_name = "task/task_form.html"
    success_url = reverse_lazy("task:list")
```

---
### DeleteView — удаление

Для удаления у Django тоже есть готовый класс:
```
from django.views.generic import DeleteView

class TaskDeleteView(DeleteView):
    model = Task
    success_url = reverse_lazy("task:list")
```

Если шаблон назвать `task/task_confirm_delete.html`, то указывать `template_name` даже не обязательно. В нём достаточно вывести информацию об удаляемой задаче и кнопку подтверждения.

---
### Миксины — избавляемся от дублирования

Когда в нескольких классах повторяется один и тот же код (например, `model`, `form_class` или `template_name`), это можно вынести в отдельный миксин:
```
class TaskMixin:
    model = Task
    success_url = reverse_lazy("task:list")

class TaskFormMixin:
    form_class = TaskForm
    template_name = "task/task_form.html"

class TaskCreateView(TaskMixin, TaskFormMixin, CreateView):
    pass

class TaskUpdateView(TaskMixin, TaskFormMixin, UpdateView):
    pass

class TaskDeleteView(TaskMixin, DeleteView):
    pass
```

Теперь код максимально компактный: всё общее вынесено, а классы получились чистыми и лаконичными.

---
### Итоги

Используя CBV, мы получаем:

- меньше кода (а значит — меньше ошибок),
    
- встроенные инструменты для работы со списками, формами и удалением,
    
- гибкую настройку через наследование и миксины,
    
- единообразный стиль кода.

Для приложения **task** это означает, что весь CRUD по задачам можно описать буквально несколькими классами.
