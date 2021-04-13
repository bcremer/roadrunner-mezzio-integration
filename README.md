# Mezzio integration for RoadRunner


How to integrate an existing [Mezzio](https://docs.mezzio.dev/) application with [RoadRunner](https://roadrunner.dev/).

## Installation

Create worker entry point `bin/roadrunner-worker.php`:

```php
<?php

declare(strict_types=1);

chdir(dirname(__DIR__));
require 'vendor/autoload.php';

$app = (static function (): \Mezzio\Application {
    $container = require 'config/container.php';
    assert($container instanceof Psr\Container\ContainerInterface);

    $app = $container->get(\Mezzio\Application::class);
    $factory = $container->get(\Mezzio\MiddlewareFactory::class);
    (require 'config/pipeline.php')($app, $factory, $container);
    (require 'config/routes.php')($app, $factory, $container);

    return $app;
})();

$worker = new Spiral\RoadRunner\Http\PSR7Worker(
    Spiral\RoadRunner\Worker::create(),
    new Laminas\Diactoros\ServerRequestFactory(),
    new Laminas\Diactoros\StreamFactory(),
    new Laminas\Diactoros\UploadedFileFactory(),
);

while ($req = $worker->waitRequest()) {
    try {
        $response = $app->handle($req);
        $worker->respond($response);
    } catch (\Throwable $e) {
        $worker->getWorker()->error((string)$e);
    }
}
```

Require the RoadRunner PHP library:

```bash
composer require spiral/roadrunner
```

Download the RoadRunner Server binary: 

```bash
composer require spiral/roadrunner
vendor/bin/rr get --location bin/
chmod +x bin/rr
```

Create `.rr.yaml`:

```yaml
server:
  command: "php -dopcache.enable_cli=1 -dopcache.validate_timestamps=0 bin/roadrunner-worker.php"

http:
  address: "0.0.0.0:8080"
  pool:
    num_workers: 8

logs:
  mode: production
  channels:
    http:
      level: info # Log all http requests, set to info to disable
    server:
      level: debug # Everything written to worker stderr is logged
    metrics:
      level: error

# Uncomment to use rpc integration
# rpc:
#   listen: tcp://127.0.0.1:6001

# Uncomment to use metrics integration
# metrics:
#   # prometheus client address (path /metrics added automatically)
#   address: "0.0.0.0:9180"
```

Create `.rr.dev.yaml`:

```yaml
server:
  command: "php bin/roadrunner-worker.php"

http:
  address: "0.0.0.0:8080"

logs:
  mode: development
  channels:
    http:
      level: debug # Log all http requests, set to info to disable
    server:
      level: debug # Everything written to worker stderr is logged
    metrics:
      level: debug

reload:
  enabled: true
  interval: 1s
  patterns: [".php", ".yaml"]
  services:
    http:
      dirs: ["."]
      recursive: true
```

## Start the server

Run RoadRunner with `./bin/rr serve` or `./bin/rr serve -c .rr.dev.yaml` (watch/dev mode).

Open http://localhost:8080

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
