# Memcached (MemCache Daemon)
> Официальная документация `https://memcached.org/about`

> Система кеширования данных в операционной памяти. Это позволяет снизить нагрузку на базу данных или файлы, так как обращение к операционной памяти намного быстрее. Эта система запускается отдельным сервером

> В API memcached есть только базовые функции: выбор сервера, коннект и дисконект, добавление, удаление, обновление и получение объекта, инкримент и дикримент. Для каждого объекта устанвливается время жизни, от 1 секунды до бесконечности. При переполнении памяти более старые объекты автоматически удаляются.

# Как примерно работает memcached. 
> Обращаемся к memcached-серверу за получением данных, возвращаем пользователю, если они есть. Если их нет, то уже обращаемся в базу данных, отдаем пользователю и добавляем эти данные в memcached-сервер, чтобы при следующем запросе не обращаться снова в базу данных.

> В memcached-сервере имеет смысл хранить только часто запрашиваемые данные. Если какой то объект запрашивается у вас раз в неделю, то скорее всего лучший использовать файловый кеш.

# Небольшой пример использования Memcached с Django
> Устанавливаем memcached-сервер
```
sudo apt-get install memcached
```

> В `requirements.txt` добавляем 
```
pymemcache
```

> После этого в `settings.py` добавляем настройки кеширования
```py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```

> После этого можете подключить кеширование на нужную вам вьюшку, используя декоратор `cache_page`
```py
from django.views.decorators.cache import cache_page

@cache_page(60*15) # указываете время, которое данные будут храниться в memcached. В данном случае 15 минут
@api_view(['GET'])
def listing(request):
    queryset = Model.objects.all()
    serializer = Serializer(queryset, many=True)
    return Response(serializer.data, status=200)
```

> Если вы хотите задекорировать view, который написан на классах, то используйте еще один декоратор `method_decorator`
```py
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

class ProductListAPIView(APIView):

    @method_decorator(cache_page(60*15))
    def list(self, request):
        queryset = Model.objects.all()
        serializer = Serializer(queryset, many=True)
        return Response(serializer.data, status=200)
```

> Но учтите, что если ваши данные поменяются, то в течение этих 15 минут, которые вы указали в декораторе данные не обновятся, так как данные изменятся в базе данных. А данные которые возвращает кеширование, хранится в операционной памяти. Это относится к любому виду кеширования.

# ошибки с memcached-сервером
> если при запросе, где используется кеширование, вам выходит ошибка, что сервер memcached не отвечает, проверьте запустился ли сервер.

```py
sudo systemctl status memcached
```
> если зелененьким не горит, значит не запустился

> попробуйте запустить вручную

```py
sudo systemctl start memcached
```
