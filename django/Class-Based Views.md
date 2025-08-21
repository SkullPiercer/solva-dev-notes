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
**views.py**
```
from django.views.generic import ListView
from .models import Task

class TaskListView(ListView):
    model = Task
    ordering = "id"
    paginate_by = 10
```

Django сам поймёт, какой шаблон подключить: по умолчанию это будет  
`task/task_list.html`. В контекст автоматически передаётся объект `page_obj`, с которым удобно работать в шаблоне.

```
    # Метод для добавления элементов в контекст
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['category_count'] = Category.objects.count()
        return context
        
	# Метод для обработки параметров url-запроса
    def get_queryset(self):
        queryset = super().get_queryset()
        query = self.request.GET.get('q')
        if query:
            queryset = queryset.filter(name__startswith=query)
        return queryset
```

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
# Детальный просмотр задачи (DetailView)

После перехода на CBV наш код стал компактнее, но вместе с этим мы потеряли одну полезную функцию: теперь нигде не отображается **подробная информация о задаче**, например, оставшееся время до её дедлайна.  
Исправим ситуацию и сделаем страницу, где будет показана **отдельная задача** и вычисленный остаток времени до её завершения.

---

## Настройка маршрутов

Чтобы организовать работу с отдельными объектами, Django предлагает специальный класс-представление — **DetailView**. Мы создадим на его основе собственный класс, который будет открывать задачу по адресу вида `/task/<pk>/`.

Пример конфигурации маршрутов:
**task/urls.py**
```
from django.urls import path
from . import views

app_name = 'task'

urlpatterns = [
    path('', views.TaskCreateView.as_view(), name='create'),
    path('list/', views.TaskListView.as_view(), name='list'),
    path('<int:pk>/', views.TaskDetailView.as_view(), name='detail'),
    path('<int:pk>/edit/', views.TaskUpdateView.as_view(), name='edit'),
    path('<int:pk>/delete/', views.TaskDeleteView.as_view(), name='delete'),
]
```
## Класс-представление
Начнём с самого простого варианта:
**task/views.py**
```
from django.views.generic import (
    CreateView, DeleteView, DetailView, ListView, UpdateView
)
from .models import Task

class TaskDetailView(DetailView):
    model = Task
```

## Шаблон для DetailView

По умолчанию Django ищет шаблон с окончанием `_detail`.  
Значит, для модели `Task` ожидается файл `task/task_detail.html`.

Простейший шаблон:
**task/task_detail.html**
```
{% extends "base.html" %}

{% block content %}
  <h2>Задача №{{ object.id }} — {{ object.title }}</h2>
  <p>{{ object.description }}</p>
  {% if deadline_left == 0 %}
    <p>Дедлайн сегодня!</p>
  {% else %}
    <p>До дедлайна осталось: {{ deadline_left }} дней</p>
  {% endif %}
{% endblock content %}
```

## Добавляем вычисления в контекст
Чтобы посчитать, сколько дней осталось до завершения задачи, используем метод `get_context_data()`.
**task/views.py**
```
from .utils import calculate_deadline

class TaskDetailView(DetailView):
    model = Task

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['deadline_left'] = calculate_deadline(self.object.deadline)
        return context
```

Теперь на странице задачи будет отображаться динамически рассчитанное время до дедлайна.

---
## Ссылка на детальную страницу

Чтобы пользователь мог попасть на подробный просмотр задачи, добавим ссылку в список:
 **task/task_list.html**
```
<div>
  {{ task.title }} ({{ task.deadline }})
  <a href="{% url 'task:detail' task.id %}">Подробнее</a>
</div>
```

## Упрощаем структуру

Чтобы проект был чище, стоит привести шаблоны к тем именам, которые Django ожидает по умолчанию. Например:

- `task_form.html` — для создания и редактирования задач,
    
- `task_detail.html` — для просмотра,
    
- `task_list.html` — для списка.

При таком подходе можно убрать часть лишних атрибутов (`template_name`) во вьюхах.

Пример оптимизированного кода:
**task/views.py**
```
from django.views.generic import (
    CreateView, DeleteView, DetailView, ListView, UpdateView
)
from django.urls import reverse_lazy
from .forms import TaskForm
from .models import Task
from .utils import calculate_deadline

class TaskListView(ListView):
    model = Task
    ordering = 'id'
    paginate_by = 10

class TaskCreateView(CreateView):
    model = Task
    form_class = TaskForm

class TaskUpdateView(UpdateView):
    model = Task
    form_class = TaskForm

class TaskDeleteView(DeleteView):
    model = Task
    success_url = reverse_lazy('task:list')

class TaskDetailView(DetailView):
    model = Task

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['deadline_left'] = calculate_deadline(self.object.deadline)
        return context
```

## Редирект на страницу задачи

После создания или редактирования задачи логично перенаправлять пользователя **на страницу этой задачи**, а не в общий список.  
Для этого в модели можно определить метод `get_absolute_url()`.
**task/models.py**
```
from django.urls import reverse
from django.db import models

class Task(models.Model):
    title = models.CharField(max_length=200)
    description = models.TextField()
    deadline = models.DateField()

    def get_absolute_url(self):
        return reverse('task:detail', kwargs={'pk': self.pk})
```
Теперь `CreateView` и `UpdateView` будут автоматически перенаправлять пользователя на страницу конкретной задачи.

---
## Статичные страницы и TemplateView

До этого момента мы делали только динамические страницы: их содержимое напрямую зависело от данных в базе. Например:

- если таблицы пусты — страницы тоже пусты;
    
- как только пользователь добавляет новые записи, сайт сразу отображает их;
    
- любое изменение объекта в БД отражается в выводе.

Однако не все страницы сайта обязаны быть «живыми». Иногда нужны простые неизменяемые разделы:

- контакты,
    
- справка,
    
- условия использования,
    
- правила безопасности и т. д.

Для таких случаев Django предоставляет удобный класс — **TemplateView**. Его задача проста: обработать запрос и вернуть страницу, собранную на основе готового шаблона.

Особенности `TemplateView`:

- работает в основном с **GET-запросами**;
    
- требует указания шаблона через атрибут `template_name`;
    
- может возвращать данные в шаблон через контекст, но это не обязательно.
    

---

### Пример: главная страница приложения задач

В нашем проекте `task` у нас есть простая стартовая страница. Раньше она описывалась обычной view-функцией, которая напрямую рендерила шаблон `pages/index.html`. Контекст там не использовался, так как всё содержимое было зашито прямо в HTML.

Теперь перепишем её на основе `TemplateView`.

**pages/views.py**
```
from django.views.generic import TemplateView


class HomePage(TemplateView):
    # Указываем шаблон, который будет отрисован
    template_name = 'pages/index.html'
```

### Подключаем маршрут

Чтобы Django понимал, что при обращении к главной странице нужно использовать новый класс, изменим `urls.py`:

**pages/urls.py**
```
from django.urls import path
from . import views

app_name = 'pages'

urlpatterns = [
    path('', views.HomePage.as_view(), name='homepage'),
]
```

Теперь при заходе на `/` откроется шаблон `index.html`, но обрабатываться он будет через `TemplateView`.

---

### Добавление данных в контекст

Хотя `TemplateView` изначально рассчитан на статичный вывод, он также умеет передавать переменные в шаблон. Делается это через переопределение метода **get_context_data()**.

Например, давайте выведем на главную страницу количество задач, созданных пользователями.

**pages/views.py**
```
from django.views.generic import TemplateView
from task.models import Task


class HomePage(TemplateView):
    template_name = 'pages/index.html'

    def get_context_data(self, **kwargs):
        # Получаем контекст от родительского метода
        context = super().get_context_data(**kwargs)
        # Добавляем туда количество объектов модели Task
        context['task_count'] = Task.objects.count()
        return context
```

### Шаблон

Теперь в `index.html` можно использовать переменную `task_count`:

**pages/index.html**
```
{% extends "base.html" %}

{% block content %}
  <h1>Добро пожаловать в Task Manager</h1>
  <p>Здесь вы можете создавать и управлять своими задачами.</p>

  <div class="row">
    <div class="col-3">
      Всего задач в системе:
    </div>
    <div class="col-9">
      <div class="display-1">{{ task_count }}</div>
    </div>
  </div>
{% endblock %}
```

---
## Преимущества CBV

- меньше кода — всё уже встроено,
    
- быстрее разработка,
    
- меньше ошибок: стандартные классы протестированы,
    
- удобная работа с GET и POST через разные методы,
    
- использование принципов ООП (наследование, переиспользование).

Django предоставляет обширный список CBV, а также отдельный сайт со справочником по ним. Освоив DetailView, CreateView и UpdateView, дальше будет значительно проще использовать остальные.
