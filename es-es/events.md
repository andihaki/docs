---
layout: default
language: 'es-es'
version: '4.0'
---

# Events Manager

* * *

## Overview

The purpose of this component is to intercept the execution of most of the other components of the framework by creating 'hook points'. These hook points allow the developer to obtain status information, manipulate data or change the flow of execution during the process of a component.

## Naming Convention

Phalcon events use namespaces to avoid naming collisions. Each component in Phalcon occupies a different event namespace and you are free to create your own as you see fit. Event names are formatted as `component:event`. For example, as [Phalcon\Db](api/Phalcon_Db) occupies the `db` namespace, its `afterQuery` event's full name is `db:afterQuery`.

When attaching event listeners to the events manager, you can use `component` to catch all events from that component (eg. `db` to catch all of the [Phalcon\Db](api/Phalcon_Db) events) or `component:event` to target a specific event (eg. `db:afterQuery`).

## Usage Example

In the following example, we will use the EventsManager to listen for the `afterQuery` event produced in a MySQL connection managed by [Phalcon\Db](api/Phalcon_Db):

```php
<?php

use Phalcon\Events\Event;
use Phalcon\Events\Manager as EventsManager;
use Phalcon\Db\Adapter\Pdo\Mysql as DbAdapter;

$eventsManager = new EventsManager();

$eventsManager->attach(
    'db:afterQuery',
    function (Event $event, $connection) {
        echo $connection->getSQLStatement();
    }
);

$connection = new DbAdapter(
    [
        'host'     => 'localhost',
        'username' => 'root',
        'password' => 'secret',
        'dbname'   => 'invo',
    ]
);

// Asignar el eventsManager a la instancia del adaptador db
$connection->setEventsManager($eventsManager);

// Enviar un comando SQL al servidor de base de datos
$connection->query(
    'SELECT * FROM products p WHERE p.status = 1'
);
```

Now every time a query is executed, the SQL statement will be echoed out. The first parameter passed to the lambda function contains contextual information about the event that is running, the second is the source of the event (in this case the connection itself). A third parameter may also be specified which will contain arbitrary data specific to the event.

> You must explicitly set the Events Manager to a component using the `setEventsManager()` method in order for that component to trigger events. You can create a new Events Manager instance for each component or you can set the same Events Manager to multiple components as the naming convention will avoid conflicts
{: .alert .alert-warning }

Instead of using lambda functions, you can use event listener classes instead. Event listeners also allow you to listen to multiple events. In this example, we will implement the [Phalcon\Db\Profiler](api/Phalcon_Db_Profiler) to detect the SQL statements that are taking longer to execute than expected:

```php
<?php

use Phalcon\Db\Profiler;
use Phalcon\Events\Event;
use Phalcon\Logger;
use Phalcon\Logger\Adapter\File;

class MyDbListener
{
    protected $profiler;

    protected $logger;

    /**
     * Creamos el perfilador e iniciamos el registro
     */
    public function __construct()
    {
        $this->profiler = new Profiler();
        $this->logger   = new Logger('../apps/logs/db.log');
    }

    /**
     * Este es ejecutado si el evento disparado es 'beforeQuery'
     */
    public function beforeQuery(Event $event, $connection)
    {
        $this->profiler->startProfile(
            $connection->getSQLStatement()
        );
    }

    /**
     * Este es ejecutado si el evento disparado es 'afterQuery'
     */
    public function afterQuery(Event $event, $connection)
    {
        $this->logger->log(
            $connection->getSQLStatement(),
            Logger::INFO
        );

        $this->profiler->stopProfile();
    }

    public function getProfiler()
    {
        return $this->profiler;
    }
}
```

Attaching an event listener to the events manager is as simple as:

```php
<?php

// Crear un oyente de base de datos
$dbListener = new MyDbListener();

// Escuchar todos los evento de la base de datos
$eventsManager->attach(
    'db',
    $dbListener
);
```

The resulting profile data can be obtained from the listener:

```php
<?php

// Send a SQL command to the database server
$connection->execute(
    'SELECT * FROM products p WHERE p.status = 1'
);

$profiler = $dbListener->getProfiler();

$profiles = $profiler->getProfiles();

foreach ($profiles as $profile) {
    echo 'SQL Statement: ', $profile->getSQLStatement(), '\n';
    echo 'Start Time: ', $profile->getInitialTime(), '\n';
    echo 'Final Time: ', $profile->getFinalTime(), '\n';
    echo 'Total Elapsed Time: ', $profile->getTotalElapsedSeconds(), '\n';
}
```

## Creating components that trigger Events

You can create components in your application that trigger events to an EventsManager. As a consequence, there may exist listeners that react to these events when generated. In the following example we're creating a component called `MyComponent`. This component is EventsManager aware (it implements [Phalcon\Events\EventsAwareInterface](api/Phalcon_Events_EventsAwareInterface)); when its `someTask()` method is executed it triggers two events to any listener in the EventsManager:

```php
<?php

use Phalcon\Events\EventsAwareInterface;
use Phalcon\Events\ManagerInterface;

class MyComponent implements EventsAwareInterface
{
    protected $eventsManager;

    public function setEventsManager(ManagerInterface $eventsManager)
    {
        $this->eventsManager = $eventsManager;
    }

    public function getEventsManager()
    {
        return $this->eventsManager;
    }

    public function someTask()
    {
        $this->eventsManager->fire('my-component:beforeSomeTask', $this);

        // Hacer alguna tarea
        echo 'Aquí, someTask\n';

        $this->eventsManager->fire('my-component:afterSomeTask', $this);
    }
}
```

Notice that in this example, we're using the `my-component` event namespace. Now we need to create an event listener for this component:

```php
<?php

use Phalcon\Events\Event;

class SomeListener
{
    public function beforeSomeTask(Event $event, $myComponent)
    {
        echo "Aquí, beforeSomeTask\n";
    }

    public function afterSomeTask(Event $event, $myComponent)
    {
        echo "Aquí, afterSomeTask\n";
    }
}
```

Now let's make everything work together:

```php
<?php

use Phalcon\Events\Manager as EventsManager;

// Crear un gestor de eventos
$eventsManager = new EventsManager();

// Crear una instancia de MyComponent
$myComponent = new MyComponent();

// Vincular el eventsManager con la instancia
$myComponent->setEventsManager($eventsManager);

// Adjuntar el oyente al the EventsManager
$eventsManager->attach(
    'my-component',
    new SomeListener()
);

// Ejecutar métodos en el componente
$myComponent->someTask();
```

As `someTask()` is executed, the two methods in the listener will be executed, producing the following output:

```bash
Aquí, beforeSomeTask
Aquí, someTask
Aquí, afterSomeTask
```

Additional data may also be passed when triggering an event using the third parameter of `fire()`:

```php
<?php

$eventsManager->fire('my-component:afterSomeTask', $this, $extraData);
```

In a listener the third parameter also receives this data:

```php
<?php

use Phalcon\Events\Event;

// Recibiendo los datos en el tercer parámetro
$eventsManager->attach(
    'my-component',
    function (Event $event, $component, $data) {
        print_r($data);
    }
);

// Recibiendo los datos desde el contexto del evento
$eventsManager->attach(
    'my-component',
    function (Event $event, $component) {
        print_r($event->getData());
    }
);
```

## Using Services From The DI

By extending [Phalcon\Plugin](api/Phalcon_Plugin), you can access services from the DI, just like you would in a controller:

```php
<?php

use Phalcon\Events\Event;
use Phalcon\Plugin;

class SomeListener extends Plugin
{
    public function beforeSomeTask(Event $event, $myComponent)
    {
        echo 'Here, beforeSomeTask\n';

        $this->logger->debug(
            'beforeSomeTask has been triggered'
        );
    }

    public function afterSomeTask(Event $event, $myComponent)
    {
        echo 'Here, afterSomeTask\n';

        $this->logger->debug(
            'afterSomeTask has been triggered'
        );
    }
}
```

## Event Propagation/Cancellation

Many listeners may be added to the same event manager. This means that for the same type of event, many listeners can be notified. The listeners are notified in the order they were registered in the EventsManager. Some events are cancelable, indicating that these may be stopped preventing other listeners from being notified about the event:

```php
<?php

use Phalcon\Events\Event;

$eventsManager->attach(
    'db',
    function (Event $event, $connection) {
        // Detenemos el evento si es cancelable
        if ($event->isCancelable()) {
            // Detenemos el evento, entonces los demás oyentes no serán notificados sobre esto
            $event->stop();
        }

        // ...
    }
);
```

By default, events are cancelable - even most of the events produced by the framework are cancelables. You can fire a not-cancelable event by passing `false` in the fourth parameter of `fire()`:

```php
<?php

$eventsManager->fire('my-component:afterSomeTask', $this, $extraData, false);
```

## Listener Priorities

When attaching listeners you can set a specific priority. With this feature you can attach listeners indicating the order in which they must be called:

```php
<?php

$eventsManager->enablePriorities(true);

$eventsManager->attach('db', new DbListener(), 150); // Más prioridad
$eventsManager->attach('db', new DbListener(), 100); // Prioridad normal
$eventsManager->attach('db', new DbListener(), 50);  // Menos prioridad
```

## Collecting Responses

The events manager can collect every response returned by every notified listener. This example explains how it works:

```php
<?php

use Phalcon\Events\Manager as EventsManager;

$eventsManager = new EventsManager();

// Establecer el gestor de eventos para recoger respuestas
$eventsManager->collectResponses(true);

// Adjuntar un oyente
$eventsManager->attach(
    'custom:custom',
    function () {
        return 'primer respuesta';
    }
);

// Adjuntar un oyente
$eventsManager->attach(
    'custom:custom',
    function () {
        return 'segunda respuesta';
    }
);

// Ejecutar el evento
$eventsManager->fire('custom:custom', null);

// Obtener todas las respuestas recogidas
print_r($eventsManager->getResponses());
```

The above example produces:

```php
    Array ( [0] => primer respuesta [1] => segunda respuesta )
```

## Implementing your own EventsManager

The [Phalcon\Events\ManagerInterface](api/Phalcon_Events_ManagerInterface) interface must be implemented to create your own EventsManager replacing the one provided by Phalcon.

## List of Events

The events available in Phalcon are:

| Componente         | Evento                               |
| ------------------ | ------------------------------------ |
| ACL                | `acl:afterCheckAccess`               |
| ACL                | `acl:beforeCheckAccess`              |
| Application        | `application:afterHandleRequest`     |
| Application        | `application:afterStartModule`       |
| Application        | `application:beforeHandleRequest`    |
| Application        | `application:beforeSendResponse`     |
| Application        | `application:beforeStartModule`      |
| Application        | `application:boot`                   |
| Application        | `application:viewRender`             |
| CLI                | `dispatch:beforeException`           |
| Collection         | `afterCreate`                        |
| Collection         | `afterSave`                          |
| Collection         | `afterUpdate`                        |
| Collection         | `afterValidation`                    |
| Collection         | `afterValidationOnCreate`            |
| Collection         | `afterValidationOnUpdate`            |
| Collection         | `beforeCreate`                       |
| Collection         | `beforeSave`                         |
| Collection         | `beforeUpdate`                       |
| Collection         | `beforeValidation`                   |
| Collection         | `beforeValidationOnCreate`           |
| Collection         | `beforeValidationOnUpdate`           |
| Collection         | `notDeleted`                         |
| Collection         | `notSave`                            |
| Collection         | `notSaved`                           |
| Collection         | `onValidationFails`                  |
| Collection         | `validation`                         |
| Collection Manager | `collectionManager:afterInitialize`  |
| Console            | `console:afterHandleTask`            |
| Console            | `console:afterStartModule`           |
| Console            | `console:beforeHandleTask`           |
| Console            | `console:beforeStartModule`          |
| Db                 | `db:afterQuery`                      |
| Db                 | `db:beforeQuery`                     |
| Db                 | `db:beginTransaction`                |
| Db                 | `db:createSavepoint`                 |
| Db                 | `db:commitTransaction`               |
| Db                 | `db:releaseSavepoint`                |
| Db                 | `db:rollbackTransaction`             |
| Db                 | `db:rollbackSavepoint`               |
| Dispatcher         | `dispatch:afterExecuteRoute`         |
| Dispatcher         | `dispatch:afterDispatch`             |
| Dispatcher         | `dispatch:afterDispatchLoop`         |
| Dispatcher         | `dispatch:afterInitialize`           |
| Dispatcher         | `dispatch:beforeException`           |
| Dispatcher         | `dispatch:beforeExecuteRoute`        |
| Dispatcher         | `dispatch:beforeDispatch`            |
| Dispatcher         | `dispatch:beforeDispatchLoop`        |
| Dispatcher         | `dispatch:beforeForward`             |
| Dispatcher         | `dispatch:beforeNotFoundAction`      |
| Loader             | `loader:afterCheckClass`             |
| Loader             | `loader:beforeCheckClass`            |
| Loader             | `loader:beforeCheckPath`             |
| Loader             | `loader:pathFound`                   |
| Micro              | `micro:afterHandleRoute`             |
| Micro              | `micro:afterExecuteRoute`            |
| Micro              | `micro:beforeExecuteRoute`           |
| Micro              | `micro:beforeHandleRoute`            |
| Micro              | `micro:beforeNotFound`               |
| Middleware         | `afterBinding`                       |
| Middleware         | `afterExecuteRoute`                  |
| Middleware         | `afterHandleRoute`                   |
| Middleware         | `beforeExecuteRoute`                 |
| Middleware         | `beforeHandleRoute`                  |
| Middleware         | `beforeNotFound`                     |
| Model              | `afterCreate`                        |
| Model              | `afterDelete`                        |
| Model              | `afterSave`                          |
| Model              | `afterUpdate`                        |
| Model              | `afterValidation`                    |
| Model              | `afterValidationOnCreate`            |
| Model              | `afterValidationOnUpdate`            |
| Model              | `beforeDelete`                       |
| Model              | `notDeleted`                         |
| Model              | `beforeCreate`                       |
| Model              | `beforeDelete`                       |
| Model              | `beforeSave`                         |
| Model              | `beforeUpdate`                       |
| Model              | `beforeValidation`                   |
| Model              | `beforeValidationOnCreate`           |
| Model              | `beforeValidationOnUpdate`           |
| Model              | `notSave`                            |
| Model              | `notSaved`                           |
| Model              | `onValidationFails`                  |
| Model              | `prepareSave`                        |
| Models Manager     | `modelsManager:afterInitialize`      |
| Consulta           | `request:afterAuthorizationResolve`  |
| Consulta           | `request:beforeAuthorizationResolve` |
| Router             | `router:beforeCheckRoutes`           |
| Router             | `router:beforeCheckRoute`            |
| Router             | `router:matchedRoute`                |
| Router             | `router:notMatchedRoute`             |
| Router             | `router:afterCheckRoutes`            |
| Router             | `router:beforeMount`                 |
| View               | `view:afterRender`                   |
| View               | `view:afterRenderView`               |
| View               | `view:beforeRender`                  |
| View               | `view:beforeRenderView`              |
| View               | `view:notFoundView`                  |
| Volt               | `compileFilter`                      |
| Volt               | `compileFunction`                    |
| Volt               | `compileStatement`                   |
| Volt               | `resolveExpression`                  |