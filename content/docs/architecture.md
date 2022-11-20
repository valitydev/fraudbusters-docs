---
title: architecture

metatitle: architecture

metadescription: architecture

category: architecture
---

# Архитектура

<img alt="svg" src="../img/full-scheme.svg" /></p>

При разработке используется микросервисный подход. Зоны ответственности разделены по функциональным блокам. 
Основной задачей при построении системы было обеспечить максимальный уровень доступности и возможность 
горизонтального масштабирования. Для этого критически важные сервисы, 
были реализованы на основе паттерна **event-sourcing**. В качестве **event-store** был выбран [**kafka-cluster**](https://kafka.apache.org/). 
А в качестве БД выбран [**basho-riak**](https://riak.com/) и [**clickhouse**](https://clickhouse.com/),
которые также предоставляют высокий уровень доступности при хорошей производительности.

#### Интерфейсы взаимодействия

Для начального погружения необходимо рассмотреть интерфейсы взаимодействия с системой. Это сервисы 
[fraudbusters-ui](https://github.com/valitydev/fraudbusters-ui) и [fraudbusters-api](https://github.com/valitydev/fraudbusters-api).

**fraudbusters-ui** - это пользовательский интерфейс, который позволяет создавать и управлять правилами, 
черными и белыми списками, группировать и привязывать правила к разным магазинам. 
Также присутствует все необходимое для аналитики и тестирования гипотиз. Для подробного изучение возможностей 
можно воспользоваться [пробной версией](./free-version.md) или обратиться к [vality.dev](./vality-support.md) за поддержкой.

**fraudbusters-api** - это сервис реализующий протокол взаимодействия с системой из вашего процессинга. 
Подробно о спецификации можно узнать в разделе [Протоколы](./api.md). 
Сервис без состояния легко масштабируется увеличением количества инстансов.

На следующе этапе необходимо ознакомиться со способом авторизации в системе. 
За нее отвечает свободно распространяемый сервис [keycloak](https://www.keycloak.org/), 
который позволяет настроить SSO, AD и LDAP.
На базе него реализована следующая ролевая [модель](./roles.md).

#### Критически важные сервисы

**fraudbusters** - это [ядро системы](https://github.com/valitydev/fraudbusters), занимается управлением оценки транзакции.
В рамках данного сервиса реализуются интерфейсы сгенерированные после сборки [**Fraudo**](https://github.com/valitydev/fraudo).
Взаимодействует с **clickhouse** для сборки агрегатов, с **wb-list-manager** для определения нахождения в списках,
с **columbus** для определения местоположения по IP адресу и **trusted-token-manager** для определения доверия к покупателю.
Сервис разработан так, чтобы максимально быстро и просто расширять систему новыми функциями. 
Для этого необходимо расширить язык и реализовать сервис для новой функциональности.

Далее необходимо рассмотреть уже упомянутые сервисы, реализующие конкретные функции оценки транзакции.

**cloumbus** - [сервис](https://github.com/valitydev/columbus), определяющий местоположение по IP - адресу.

**wblist-manager** - [сервис](https://github.com/valitydev/wb-list-manager), 
предоставляющий возможность создавать записи в черных, белых и серых списках. Сервис слушает команды из 
**fraudbusters-management** и создает записи в списках в базе **riak**, по результату отправляет подтверждение в
**fraudbusters-management** о результате создания, что позволяет поддерживать **eventually-consistent** для клиента.

**trusted-token-manager** - [сервис](https://github.com/valitydev/trusted-tokens-manager), 
собирающий статистику операций из **event-store** и предоставляющий гибкий [интерфейс](https://github.com/valitydev/trusted-tokens-proto)
для определения доверенного плательщика.

#### Вспомогательные сервисы

Также реализован ряд сервисов для вспомогательных операций и аналитики.


**fraudbusters-management** - backend [сервис](https://github.com/valitydev/fraudbusters-management) управления всей системой.
Через него проходят все запросы от UI, а также запросы на управление сущностями системы, такими как шаблоны, группы и связи.
Подробное описание методов можно посмотреть в [спецификации](https://valitydev.github.io/swag-fraudbusters-management/).

Создание необходимых для функционирования сущностей (шаблонов, групп, связей и записей в черных/белых списках)
производится методом отправки соответствующей [команды](https://github.com/valitydev/fraudbusters-proto/blob/master/proto/fraudbusters.thrift#L40)
через event-store (kafka) в ядро системы **fraudbusters** или в **wb-list-manager**.

**fraudbusters-notificator** - [сервис](https://github.com/valitydev/fraudbusters-notificator), управляющий 
запланированными операциями по сбору и отправке оповещений. На данный момент реализован канал отправки по почте, 
но сама реализация подразумевает быстрое расширение под другие каналы.

**fraudbusters-warehouse** - [сервис](https://github.com/valitydev/fraudbusters-warehouse), 
отвечающий за аналитические запросы в **clickhouse**. Реализует следующий [протокол](https://github.com/valitydev/fraudbusters-warehouse-proto).