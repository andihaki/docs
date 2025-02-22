---
layout: default
language: 'zh-cn'
version: '4.0'
title: 'Phalcon\Factory'
---

* [Phalcon\Factory\AbstractFactory](#factory-abstractfactory)
* [Phalcon\Factory\Exception](#factory-exception)

<h1 id="factory-abstractfactory">Class Phalcon\Factory\AbstractFactory</h1>

[Source on GitHub](https://github.com/phalcon/cphalcon/tree/v{{ page.version }}.0/phalcon/factory/abstractfactory.zep)

| Namespace | Phalcon\Factory | | Uses | Phalcon\Config, Phalcon\Factory\Exception |

This file is part of the Phalcon Framework.

(c) Phalcon Team [&#x74;e&#97;&#x6d;&#64;&#x70;&#104;&#x61;&#108;&#x63;&#111;&#110;&#x70;&#104;&#x70;&#46;&#x63;&#111;&#x6d;](&#109;&#x61;&#105;&#x6c;&#116;&#x6f;&#58;&#x74;e&#97;&#x6d;&#64;&#x70;&#104;&#x61;&#108;&#x63;&#111;&#110;&#x70;&#104;&#x70;&#46;&#x63;&#111;&#x6d;)

For the full copyright and license information, please view the LICENSE.txt file that was distributed with this source code.

## Properties

```php
/**
 * @var array
 */
protected mapper;

/**
 * @var array
 */
protected services;

```

## Methods

```php
protected function checkConfig( mixed $config ): array;
```

Checks the config if it is a valid object

```php
protected function checkService( string $name ): void;
```

Checks if a service exists and throws an exception

```php
abstract protected function getAdapters(): array;
```

Returns the adapters for the factory

```php
protected function init( array $services = [] ): void;
```

AdapterFactory constructor.

<h1 id="factory-exception">Class Phalcon\Factory\Exception</h1>

[Source on GitHub](https://github.com/phalcon/cphalcon/tree/v{{ page.version }}.0/phalcon/factory/exception.zep)

| Namespace | Phalcon\Factory | | Extends | \Phalcon\Exception |

This file is part of the Phalcon Framework.

(c) Phalcon Team [&#x74;e&#97;&#x6d;&#64;&#x70;&#104;&#x61;&#108;&#x63;&#111;&#110;&#x70;&#104;&#x70;&#46;&#x63;&#111;&#x6d;](&#109;&#x61;&#105;&#x6c;&#116;&#x6f;&#58;&#x74;e&#97;&#x6d;&#64;&#x70;&#104;&#x61;&#108;&#x63;&#111;&#110;&#x70;&#104;&#x70;&#46;&#x63;&#111;&#x6d;)

For the full copyright and license information, please view the LICENSE.txt file that was distributed with this source code.