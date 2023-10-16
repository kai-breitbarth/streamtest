# Streaming (e.g. OpenAI) with Django

This example shows only the streaming part. It works with the Daphne Webserver. I didn’t get in running with runserver and I didn’t try with Gunicorn.

## **Step 1: Setup Redis, Django and Channels**

1. Install Redis (on Mac)

```bash
$ brew install redis
To start redis now and restart at login:
  brew services start redis
Or, if you don't want/need a background service you can just run:
  /opt/homebrew/opt/redis/bin/redis-server /opt/homebrew/etc/redis.conf
==> Summary
🍺  /opt/homebrew/Cellar/redis/7.2.1: 14 files, 2.4MB
==> Running `brew cleanup redis`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).

$ brew services start redis
```

2. Create a django app

```bash
$ mkdir streamtest
$ cd streamtest
$ pyenv shell 3.11.5
$ poetry init
# some questions
$ poetry add django
$ poetry shell
$ django-admin startproject django_project
$ python manage.py startapp chat
$ code .
```

3. Now to the streaming app

```bash
$ poetry add daphne 
$ poetry add channels
$ poetry add channels-redis
```

4. Update your Django settings:

```python
INSTALLED_APPS = [
    ...
    'channels',
    'chat',
]

# Use channels layer as the default backend for `asgi`
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],
        },
    },
}

# Use channels for routing
ASGI_APPLICATION = 'django_project.asgi.application'
```

5. Change the **`asgi.py`** in your django_project folder:

```bash
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from chat import routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'django_project.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": URLRouter(
        routing.websocket_urlpatterns
    ),
})
```

## **Step 2: WebSocket Consumers**

1. In your app (chat), create **`consumers.py`**:

```python
import json
import asyncio
from channels.generic.websocket import AsyncWebsocketConsumer

class OpenAIConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()

    async def disconnect(self, close_code):
        pass

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        if message == 'start':
            # Simulate streaming from OpenAI (you'd replace with the OpenAI API call)
            for i in range(10):
                await self.send(text_data=json.dumps({
                    'message': f'Number {i}'
                }))
                await asyncio.sleep(0.5) # needs to be asyncio.sleep
```

2. In your app, create **`routing.py`**:

```python
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/openai/$', consumers.OpenAIConsumer.as_asgi()),
]
```

### **Step 3: Frontend Code**

1. Create **`/chat/templates/index.html`**:

```html
<textarea id="openaiOutput" rows="15"></textarea>
<button onclick="startStreaming()">Start</button>

<script>
    let socket = new WebSocket('ws://' + window.location.host + '/ws/openai/');

    socket.onmessage = function(e) {
        let data = JSON.parse(e.data);
        let message = data['message'];
        document.getElementById('openaiOutput').value += message + '\n';
    };

    function startStreaming() {
        socket.send(JSON.stringify({
            'message': 'start'
        }));
    }
</script>
```

2. Create **`/chat/urls.py`**:

```bash
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

3. Add chat.urls to the **`/django_project/urls.py`**:

```bash
...
urlpatterns = [
    ...
    path('', include('chat.urls')), 
]
```

### **Step 4: Run your Django Server with Channels**

Run the server:

```bash
$ python manage.py migrate
$ daphne django_project.asgi:application
```
