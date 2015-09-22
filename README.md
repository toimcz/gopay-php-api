
# Gopay's PHP SDK for Payments REST API

[![License](https://poser.pugx.org/gopay/payments-php-sdk/license)](https://packagist.org/packages/gopay/payments-php-sdk)
[![Latest Stable Version](https://poser.pugx.org/gopay/payments-php-sdk/v/stable)](https://packagist.org/packages/gopay/payments-php-sdk)
[![Dependency Status](https://www.versioneye.com/user/projects/55ff8ef0601dd900150001e5/badge.svg?style=flat)](https://www.versioneye.com/user/projects/55ff8ef0601dd900150001e5)

## Requirements

- PHP >= 5.4.0
- enabled extension `curl`, `json`

## Installation

```bash
composer require gopay/payments-sdk-php --update-no-dev
```

This command requires you to have Composer installed globally, as explained
in the [installation chapter](https://getcomposer.org/doc/00-intro.md)
of the Composer documentation.

### Private repository

Add this line to `composer.json`

```json
{
    "repositories": [
        { "type": "vcs", "url": "git@bitbucket.org:edgedesigncz/gop-016-sdk-php.git" }
    ]
}
```

## Basic usage

```php
$gopay = GoPay\payments([
    'clientId' => 'my id',
    'clientSecret' => 'my secret',
    'isProductionMode' => false,
    'scope' => GoPay\Token\TokenScope::ALL
]);
```

### Configuration

Required field | Data type | Documentation |
-------------- | --------- | ----------- |
`clientId` | string | https://doc.gopay.com/en/?shell#oauth |
`clientSecret` | string | https://doc.gopay.com/en/?shell#oauth |
`isProductionMode` | boolean | [test or production environment?](https://help.gopay.com/en/s/ey) |
`scope` | [`GoPay\Token\TokenScope`](src/Token/TokenScope.php) constant | https://doc.gopay.com/en/?shell#scope |

### Available methods

API | SDK method |
--- | ---------- |
[Create standard payment](https://doc.gopay.com/en/#standard-payment) | `$gopay->createPayment(array $payment)` |
[Status of the payment](https://doc.gopay.com/en/#status-of-the-payment) | `$gopay->getStatus($id)` |
[Refund of the payment](https://doc.gopay.com/en/#refund-of-the-payment-(cancelation)) | `$gopay->refund($id, $amount)` |
[Create recurring payment](https://doc.gopay.com/en/#recurring-payment) | `$gopay->createPayment(array $payment)` |
[Recurring payment on demand](https://doc.gopay.com/en/#recurring-payment-on-demand) | `$gopay->recurrenceOnDemand($id, array $payment)` |
[Cancellation of the recurring payment](https://doc.gopay.com/en/#cancellation-of-the-recurring-payment) | `$gopay->recurrenceVoid($id)` |
[Create pre-authorized payment](https://doc.gopay.com/en/#pre-authorized-payment) | `$gopay->createPayment(array $payment)` |
[Charge of pre-authorized payment](https://doc.gopay.com/en/#charge-of-pre-authorized-payment) | `$gopay->preauthorizedCapture($id)` |
[Cancellation of the pre-authorized payment](https://doc.gopay.com/en/#cancellation-of-the-pre-authorized-payment) | `$gopay->preauthorizedVoid($id)` |

### SDK response? Has my call succeed?

SDK returns wrapped API response. Every method returns
[`GoPay\Http\Response` object](src/Http/Response.php). Structure of `json/__toString`
should be same as in [documentation](https://doc.gopay.com/en).
SDK throws no exception. Please create an issue if you catch one. 

```php
$response = $gopay->createPayment([/* define your payment  */]);
if ($response->hasSucceed()) {
    echo "hooray, API returned {$response}";
    return $response->json['gw_url']; // url for initiation of gateway
} else {
    echo "oop, API returns {$response->statusCode}: {$response}";
    // errors format: https://doc.gopay.com/en/?shell#http-result-codes
    echo var_dump($response->json);
}

```

Method | Description |
------ | ---------- |
`$response->hasSucceed()` | checks if API returns status code _200_ |
`$response->json` | decoded response, returned objects are converted into associative arrays |
`$response->statusCode` | HTTP status code |
`$response->__toString()` | raw body from HTTP response |

### Are required fields and allowed values validated?

**No.** API [validates fields](https://doc.gopay.com/en/?shell#return-errors) pretty extensively
so there is no need to duplicate validation in SDK. It would only introduce new type of error.
Or we would have to perfectly simulate API error messages. That's why SDK just calls API which
behavior is well documented in [doc.gopay.com](https://doc.gopay.com/en).

*****

## Advanced usage

### Framework integration

* [Symfony2](/examples/symfony.md)

### Cache access token

Access token expires after 30 minutes so it's expensive to use new token for every request.
Unfortunately it's default behavior of [`GoPay\Token\InMemoryTokenCache`](src/Token/InMemoryTokenCache.php).
But you can implement your cache and store tokens in Memcache, Redis, files, ... It's up to you.

Your cache must implement template methods from [`GoPay\Token\TokenCache`](src/Token/TokenCache.php).
Be aware that there are two [scopes](https://doc.gopay.com/en/?shell#scope) (`TokenScope`).
So token must be cached for each scope. 
Below you can see example implementation of caching tokens in file (@todo test it :):


```php
$gopay = GoPay\payments(['..config...'], new PrimitiveFileCache());
```

```php
<?php

use GoPay\Token\TokenCache;
use GoPay\Token\AccessToken;

class PrimitiveFileCache extends TokenCache
{
    private $file;

    public function setScope($scope)
    {
        $this->file = __DIR__ . "/{$scope}";
    }

    public function setAccessToken(AccessToken $t)
    {
        file_put_contents($this->file, serialize($t);
    }

    public function getAccessToken()
    {
        if (file_exists($this->file)) {
            return unserialize(file_get_contents($this->file));
        }
        return null; // you can return null, this method is called only if `isExpired` return false
    }
}

```