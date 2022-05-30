# Инициализация

Когда плагин включается он опрашивает сервер на предмет запущенных агентов:

```
comm.send("get", "agent", {}).then(function(list: AgentDesc[]) { ... });
```

где AgentDesc — { id: number, name: string }. Пример ответа:

```
[
    { id: 11, name: "power-switch" },
    { id: 12, name: "vehicle-monitor" }
]
```

В перспективе эту штуку будет делать AgentMultiplexer.


# Запрос списка PowerSwitch

Для запроса нужно выяснить к какому агенту обращаться: найти его id. В примере
выше у агента PowerSwitch id=11.

```
comm.send("upload", "agent", {
    id:   11,
    name: "get-switches",
    args: { timeline: null }
}).then(function(list: SwitchDesc[]) { ... });
```

где SwitchDesc:
```
{
    vehicle: number, // номер (или ID) аппатата
    entity:  number, // ID сущности отвечающей за PowerSwitch
    name:    string  // Название сущности
}
```

Пример ответа:
```
[
    {
        vehicle: 32768,
        entity:  2048,
        name:    "Main Power-Switch"
    },
    {
        vehicle: 32768,
        entity:  2049,
        name:    "Payload Power-Switch"
    },
    {
        vehicle: 16384,
        entity:  256,
        name:    "Power-Switch"
    }
]
```

Плагину нужно запомнить Vehicle ID и Entity ID для оперативных запросов к
агентам: таким, как "get-channels". Также нужно запомнить и пару Vehicle ID и
Entity Name на случай, если аппатат перезагрузится и все Entity ID изменятся.

На будущее:
Если выбранный Vehicle был обнаружен в сети — плагину нужно перезапросить
список каналов и найти такой Power-Switch, название и Vehicle ID которого
совпадают с сохранёнными, а также обновить соответствующий Entity ID для
Power-Switch и запросить у него список каналов повторно.


# Запрос каналов на выбранном PowerSwitch

Опрос выполняется:

```
comm.send("upload", "agent", {
    id: 11,
    name: "get-channels",
    args: {
        vehicle:  32768,
        entity:   2049, // ID свича
        timeline: null
    }
}).then(function(list: ChannelDesc[]) { ... });
```

где ChannelDesc — { entity: number, name: string, order: number, state: enum }.
Пример:
```
[
    {
        entity:  4096,  // Entity-ID of specific channel
        state:   "off",
        order:   0,
        name:    "Navigation Lights"
    },
    {
        entity:  4097,
        state:   "on",
        order:   10,
        name:    "Left Thruster"
    },
    {
        entity:  4098,
        state:   "unknown",
        order:   11,
        name:    "Right Thruster"
    }
]
```

Здесь Entity ID указаны для каждого канала индивидуально. От этих Entities
будут приходить события об изменении состояния каналов.

Имена каналов можно не запоминать. В случае подключения к аппарату — список
каналов можно опросить повторно.

Поле order используется только для упорядочивания каналов в списке.


# Управление каналами

Для изменения состояния канала ON→OFF или OFF→ON используется команда
"control-channel":

```
comm.send("upload", "agent", {
    id: 11,
    name: "control-channel",
    args: {
        vehicle: 32768,
        entity:  4096, // ID канала
        state:   "on"
    }
});
```

Ответ сервера на эту команду не важен. Но пользователю нужно как-то показать,
что команда отправлена. А когда будет получен статус от канала — этот
временный фидбек можно убрать.


# Получение оповещений от PowerSwitch

Для регистрации на подписку нужно отправить команду "start" менеджеру агентов:
```
comm.send("start", "agent", { id: 11 });
```

Здесь ID агента передаётся внутри объекта, а не просто числом. Это сделано для
возможного дальнейшего расширения интерфейса.

После выполнения этой команды агент начёт посылать по WebSocket апдейты
информации по состоянию каналов, а также телеметрию от них.

Для получения этих сообщений из WebSocket нужно подписаться:
```
comm.OnServerRequest.subscribe(function({
    agent: 11,      // от какого агента
    name:  "state", // название сообщения
    data: {
        vehicle: 32768, // ID аппарата
        entity: 4096,   // ID канала
        state:  "on"    // его состояние
    }
}) { ...  });
```

Это довольно низкоуровневый интерфейс. В перспективе нужно будет подписаться в
AgentMultiplexer и приходить будут события типа PowerChannelState с данными
vehicle, entity и state.

Приходить такие сообщения будут для всех каналов всех свичей на всех
аппаратах. Плагину нужно отфильтровывать неинтересные каналы самостоятельно.
