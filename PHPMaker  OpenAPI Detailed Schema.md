# OpenAPI Full Schema for PHPMaker

This README provides instructions for setting up the OpenAPI full schema for PHPMaker, including file structure, configuration, and API setup.

## File Structure

Ensure you have the following folder structure in your root directory:

```
/app
├── api
│   └── Service.php
├── data
├── lib
│   ├── ApiService.php
├── api.php
└── global.php
```

## Configuration

### Global Code

In PHPMaker, go to Code (Server events) > Global Code and add the following line:

```php
include(dirname(__DIR__, 1) . "/app/global.php");
```

### API Action

In PHPMaker, go to Code (Server events) > API Action and add the following code:

```php
// API Action event
function Api_Action($app)
{
    include(dirname(__DIR__, 1) . "/app/api.php");
}
```

## File Contents

### /app/api.php

```php
<?php
namespace PHPMaker2024\AppSystem; // Change this to your name space
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

$jwtMiddleware = function (Request $request, RequestHandler $handler): Response {
    $authHeader = $request->getHeaderLine('Authorization');
    if (!$authHeader) {
        $authHeader = $request->getHeaderLine('X-Authorization');
    }
    $jwt = str_replace('Bearer ', '', $authHeader);

    // Decode the JWT
    $payload = DecodeJwt($jwt);

    // If the JWT is invalid, return a 401 Unauthorized response
    if (isset($payload['failureMessage']) || !isset($jwt) || empty($jwt)) {
        $response = new \Nyholm\Psr7\Response();
        $response = $response->withHeader('Content-Type', 'application/json');
        $response->getBody()->write(json_encode(['error' => 'Unauthorized']));
        return $response->withStatus(401);
    }

    // Attach user_id to the request
    $user_id = $payload['userid'] ?? null;
    $request = $request->withAttribute('user_id', $user_id);

    // If the JWT is valid, pass the request to the next middleware
    $response = $handler->handle($request);
    return $response;
};

include(dirname(__DIR__, 1) . "/app/api/Service.php");
```

### /app/api/Service.php

```php
<?php
namespace PHPMaker2024\AppSystem;

$app->get("/Service", function ($request, $response, $args) {
    $apiService = new Service\ApiService();
    $result = $apiService->generateOAI();
    return $response->write($result);
}); // ->add($jwtMiddleware)
```

### /app/lib/ApiService.php

```php
<?php

namespace PHPMaker2024\AppSystem\Service;

use PHPMaker2024\AppSystem as App;
use function PHPMaker2024\AppSystem\Container;

class ApiService
{
    public function generateOAI(): string
    {
        $openapi = [
            'openapi' => '3.0.1',
            'info' => [
                'title' => 'REST API',
                'version' => 'v1'
            ],
            'servers' => [
                ['url' => '' /* 'http://' . $_SERVER['HTTP_HOST'] */]
            ],
            'paths' => $this->generatePaths(),
            'components' => [
                'schemas' => $this->generateSchemas(),
                'securitySchemes' => [
                    'Bearer' => [
                        'type' => 'apiKey',
                        'description' => '*Note: Login to get your JWT token first, then enter "Bearer &lt;JWT Token&gt;" below, e.g.<br><em>Bearer 123456abcdef</em>',
                        'name' => 'X-Authorization',
                        'in' => 'header'
                    ]
                ]
            ],
            'security' => [
                ['Bearer' => []]
            ]
        ];

        return json_encode($openapi, JSON_PRETTY_PRINT);
    }

    private function generatePaths(): array
    {
        $entityFiles = glob(__DIR__ . '/../../src/Entity/*.php');

        $paths = [
            '/api/login' => [
                'post' => [
                    'tags' => ['Login'],
                    'requestBody' => [
                        'content' => [
                            'application/x-www-form-urlencoded' => [
                                'schema' => [
                                    'type' => 'object',
                                    'properties' => [
                                        'username' => ['type' => 'string'],
                                        'password' => ['type' => 'string'],
                                        'securitycode' => ['type' => 'string'],
                                        'expire' => ['type' => 'string'],
                                        'permission' => ['type' => 'string']
                                    ],
                                    'required' => ['username', 'password']
                                ]
                            ]
                        ]
                    ],
                    'responses' => [
                        '200' => ['description' => 'Success']
                    ]
                ]
            ],
            '/api/register' => [
                'post' => [
                    'tags' => ['Register'],
                    'summary' => 'Register a new user',
                    'requestBody' => [
                        'required' => true,
                        'description' => '*Note: Enter values as JSON, e.g. {"name1": "value1", "name2": "value2", ... }, make sure you double quote the field names.',
                        'content' => [
                            'application/json' => [
                                'schema' => [
                                    '$ref' => '#/components/schemas/user'
                                ]
                            ]
                        ]
                    ],
                    'responses' => [
                        '200' => ['description' => 'Success']
                    ]
                ]
            ],
            '/api/upload' => [
                'post' => [
                    'tags' => ['Upload'],
                    'summary' => 'Upload file(s)',
                    'requestBody' => [
                        'content' => [
                            'multipart/form-data' => [
                                'schema' => [
                                    'type' => 'object',
                                    'properties' => [
                                        'files[]' => [
                                            'type' => 'array',
                                            'items' => [
                                                'type' => 'string',
                                                'format' => 'binary'
                                            ]
                                        ]
                                    ]
                                ],
                                'encoding' => [
                                    'files[]' => ['allowReserved' => true]
                                ]
                            ]
                        ],
                        'responses' => [
                            '200' => ['description' => 'Success']
                        ]
                    ]
                ]
            ]
        ];

        foreach ($entityFiles as $file) {
            $entityDetails = $this->extractPropertiesFromEntity($file);
            $table = $entityDetails['table'];
            $properties = $entityDetails['properties'];

            $paths["/api/list/{$table}"] = [
                'get' => [
                    'tags' => ['List'],
                    'summary' => "List records from the {$table} table",
                    'parameters' => [
                        ['name' => 'page', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                        ['name' => 'start', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/view/{$table}/{key}"] = [
                'get' => [
                    'tags' => ['View'],
                    'summary' => "View a record from the {$table} table",
                    'parameters' => [
                        ['name' => 'key', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/add/{$table}"] = [
                'post' => [
                    'tags' => ['Add'],
                    'summary' => "Add a record to the {$table} table",
                    'requestBody' => [
                        'required' => true,
                        'description' => '*Note: Enter values as JSON, e.g. {"name1": "value1", "name2": "value2", ... }, make sure you double quote the field names.',
                        'content' => [
                            'application/json' => [
                                'schema' => [
                                    '$ref' => "#/components/schemas/{$table}"
                                ]
                            ]
                        ]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/delete/{$table}/{key}"] = [
                'get' => [
                    'tags' => ['Delete'],
                    'summary' => "Delete a record from the {$table} table",
                    'parameters' => [
                        ['name' => 'key', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/delete/{$table}"] = [
                'post' => [
                    'tags' => ['Delete'],
                    'summary' => "Delete multiple records from the {$table} table",
                    'requestBody' => [
                        'required' => true,
                        'content' => [
                            'application/x-www-form-urlencoded' => [
                                'schema' => [
                                    'type' => 'object',
                                    'properties' => [
                                        'key_m[]' => [
                                            'type' => 'array',
                                            'items' => ['type' => 'string']
                                        ]
                                    ]
                                ],
                                'encoding' => [
                                    'key_m[]' => ['allowReserved' => true]
                                ]
                            ]
                        ],
                        'responses' => ['200' => ['description' => 'Success']]
                    ]
                ]
            ];

            $paths["/api/edit/{$table}/{key}"] = [
                'post' => [
                    'tags' => ['Edit'],
                    'summary' => "Edit a record in the {$table} table",
                    'parameters' => [
                        ['name' => 'key', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                    ],
                    'requestBody' => [
                        'required' => true,
                        'description' => '*Note: Enter values as JSON, e.g. {"name1": "value1", "name2": "value2", ... }, make sure you double quote the field names.',
                        'content' => [
                            'application/json' => [
                                'schema' => [
                                    '$ref' => "#/components/schemas/{$table}"
                                ]
                            ]
                        ]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/file/{$table}/{field}/{key}"] = [
                'get' => [
                    'tags' => ['File'],
                    'summary' => "Get file(s) info from the {$table} table by primary key",
                    'parameters' => [
                        ['name' => 'field', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']],
                        ['name' => 'key', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/file/{$table}/{fn}"] = [
                'get' => [
                    'tags' => ['File'],
                    'summary' => "Get a file from the {$table} table by encrypted file path",
                    'parameters' => [
                        ['name' => 'fn', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];

            $paths["/api/export/{$table}/{key}"] = [
                'get' => [
                    'tags' => ['Export'],
                    'summary' => "Export records from the {$table} table",
                    'parameters' => [
                        ['name' => 'key', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']],
                        ['name' => 'page', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                        ['name' => 'recperpage', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                        ['name' => 'filename', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                        ['name' => 'save', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                        ['name' => 'output', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']]
                    ],
                    'responses' => ['200' => ['description' => 'Success']]
                ]
            ];
        }

        $paths['/api/export/search'] = [
            'get' => [
                'tags' => ['Export'],
                'summary' => 'Search export log',
                'parameters' => [
                    ['name' => 'limit', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                    ['name' => 'type', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                    ['name' => 'tablename', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                    ['name' => 'filename', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                    ['name' => 'datetime', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']],
                    ['name' => 'output', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']]
                ],
                'responses' => ['200' => ['description' => 'Success']]
            ]
        ];

        $paths['/api/export/{id}'] = [
            'get' => [
                'tags' => ['Export'],
                'summary' => 'Get Exported file',
                'parameters' => [
                    ['name' => 'id', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']],
                    ['name' => 'filename', 'in' => 'query', 'required' => false, 'schema' => ['type' => 'string']]
                ],
                'responses' => ['200' => ['description' => 'Success']]
            ]
        ];

        $paths['/api/permissions/{userlevel}'] = [
            'get' => [
                'tags' => ['Permissions'],
                'summary' => 'Get permissions of a user level',
                'parameters' => [
                    ['name' => 'userlevel', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                ],
                'responses' => ['200' => ['description' => 'Success']]
            ],
            'post' => [
                'tags' => ['Permissions'],
                'summary' => 'Update permissions of a user level',
                'parameters' => [
                    ['name' => 'userlevel', 'in' => 'path', 'required' => true, 'schema' => ['type' => 'string']]
                ],
                'requestBody' => [
                    'required' => true,
                    'description' => '*Note: Enter values as JSON, e.g. {"name1": value1, "name2": value2, ... }, make sure you double quote the table names.',
                    'content' => [
                        'application/json' => ['schema' => ['type' => 'object']]
                    ]
                ],
                'responses' => ['200' => ['description' => 'Success']]
            ]
        ];

        return $paths;
    }

    private function generateSchemas(): array
    {
        $schemas = [];
        $entityFiles = glob(__DIR__ . '/../../src/Entity/*.php');

        foreach ($entityFiles as $file) {
            $entityDetails = $this->extractPropertiesFromEntity($file);
            $table = $entityDetails['table'];
            $properties = $entityDetails['properties'];

            $schemas[$table] = [
                'type' => 'object',
                'properties' => $properties
            ];
        }

        return $schemas;
    }

    private function extractPropertiesFromEntity($file): array
    {
        $properties = [];
        $table = '';

        $content = file_get_contents($file);

        // Match the #[Table(name: "tablename")] annotation
        if (preg_match('/\#\[Table\(name\s*:\s*\"([^\"]+)\"\)\]/', $content, $tableMatch)) {
            $table = $tableMatch[1];
        }

        // Match the #[Column(...)] annotations in the file content
        if (preg_match_all('/\#\[Column\((.*?)\)\]/s', $content, $matches)) {
            foreach ($matches[1] as $match) {
                $property = $this->parseColumnAnnotation($match);
                if ($property) {
                    $properties[$property['name']] = $property['schema'];
                }
            }
        }

        return ['table' => $table, 'properties' => $properties];
    }

    private function parseColumnAnnotation($annotation): array
    {
        $property = [];
        $schema = [];

        // Match name="value"
        if (preg_match('/name\s*:\s*\"([^\"]+)\"/', $annotation, $nameMatch)) {
            $property['name'] = $nameMatch[1];
        } else {
            return [];
        }

        // Match type="value"
        if (preg_match('/type\s*:\s*\"([^\"]+)\"/', $annotation, $typeMatch)) {
            $type = $typeMatch[1];
            switch ($type) {
                case 'integer':
                    $schema['type'] = 'integer';
                    break;
                case 'string':
                    $schema['type'] = 'string';
                    break;
                case 'date':
                    $schema['type'] = 'string';
                    $schema['format'] = 'date';
                    break;
                case 'datetimetz':
                    $schema['type'] = 'string';
                    $schema['format'] = 'date-time';
                    break;
                case 'smallint':
                    $schema['type'] = 'integer';
                    break;
                case 'text':
                    $schema['type'] = 'string';
                    break;
                case 'bigint':
                    $schema['type'] = 'integer';
                    break;
                case 'decimal':
                    $schema['type'] = 'number';
                    $schema['format'] = 'float';
                    break;
                default:
                    break;
            }
        } else {
            return [];
        }

        // Match nullable=true or nullable=false
        if (preg_match('/nullable\s*:\s*(true|false)/', $annotation, $nullableMatch)) {
            $nullable = $nullableMatch[1] === 'true';
            if ($nullable) {
                $schema['nullable'] = true;
            }
        }

        // Match unique=true
        if (preg_match('/unique\s*:\s*true/', $annotation)) {
            $schema['unique'] = true;
        }

        $property['schema'] = $schema;
        return $property;
    }
}

```

## Usage

After setting up the files and configurations, you can access the service at:

```
domain.com/api/Service
```

## Notes

- Make sure to implement the `generateOAI()` method in the `ApiService` class.
- The JWT middleware is commented out in the Service route. Uncomment it if you want to use JWT authentication for this endpoint.
- Implement other necessary methods and classes as required by your application.

For more detailed information or specific implementation details, please refer to the individual files and PHPMaker documentation.