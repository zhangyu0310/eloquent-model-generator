# Eloquent Model Generator

Eloquent Model Generator is a tool based on [Code Generator](https://github.com/krlove/code-generator) for generating Eloquent models.

## 说明

这个仓库是从原开发者那边fork的，基于版本1.3.7

由于开发中遇到了一些小问题，对源码进行了少量修改。

修改如下：

1. 工具会默认忽略表的主键，不会将其添加到`$fillable`字段中。（原因应该是大部分使用者的数据库主键是Auto Increment ）

我目前的项目中，这个字段并不是自增的，如果不将主键增加到`$fillable`中，没办法使用`fill($request->all())`之类的方法直接填充。

> 解决办法：
>
> 我增加了一个参数 `--pk-fillable` (Primary Key fillable)，是一个`VALUE_NONE`。 指定了这个参数后，主键也会被添加到`$fillable`中。

2. 工具在检查外键关系时，会扫描其它的表（`table-name`指定之外的表）

在这个时候，如果其它表中的列，有一些很特殊的类型，例如`bit`类型。程序就会抛出异常，没有办法正常工作。我发现这个问题，是因为这边使用了`Liquibase`生成数据库表，而`Liquibase`会生成一张名为`databasechangeloglock`的表：
```sql
CREATE TABLE `databasechangeloglock` (
  `ID` int(11) NOT NULL,
  `LOCKED` bit(1) NOT NULL,
  `LOCKGRANTED` datetime DEFAULT NULL,
  `LOCKEDBY` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
根本原因是调用`DBAL`的`listTables()`时，对于其不支持的类型（`bit`类型）抛出的异常。

> 解决办法：
> 
> 我这里的解决办法比较简单粗暴，仅适合没有使用外键的业务。（从一个前数据库开发人员的角度来说，外键还是能不用就不用，效率很差，而且很多分布式数据库不支持外键）
>
> `--ignore-fk` (Ignore Foreign Key)，是一个`VALUE_NONE`。指定了这个参数后，工具会跳过外键关系的检查。

3. 这个可以算是一点点小特性。生成的model，它头部会有字段的说明。我在这个说明后面加了数据库表列上的COMMENT。会生成类似这样的model文件：
```php
/**
 * @property integer $ServerID // 服务器ID
 * @property string $Name // 服务器名称
 * @property string $IP // IP地址
 * @property int $Port // 端口号
 */
```
不过这个功能需要配合我Git下的 [Code Generator](https://github.com/zhangyu0310/code-generator) 仓库里的代码生成器使用。（也是修改了一点点内容）

## Installation

Step 1. Add Eloquent Model Generator to your project:
```
composer require krlove/eloquent-model-generator --dev
```
Step 2. Register `GeneratorServiceProvider`:
```php
'providers' => [
    // ...
    Krlove\EloquentModelGenerator\Provider\GeneratorServiceProvider::class,
];
```
If you are using Laravel version 5.5 or higher this step can be omitted since this project supports [Package Discovery](https://laravel.com/docs/5.5/packages#package-discovery) feature.

Step 3. Configure your database connection.

## Usage
Use
```
php artisan krlove:generate:model User
```
to generate a model class. Generator will look for table with name `users` and generate a model for it.

### table-name
Use `table-name` option to specify another table name:
```
php artisan krlove:generate:model User --table-name=user
```
In this case generated model will contain `protected $table = 'user'` property.

### output-path
Generated file will be saved into `app` directory of your application and have `App` namespace by default. If you want to change the destination and namespace, supply the `output-path` and `namespace` options respectively:
```
php artisan krlove:generate:model User --output-path=/full/path/to/output/directory --namespace=Some\\Other\\NSpace
```
`output-path` can be absolute path or relative to project's `app` directory. Absolute path must start with `/`:
- `/var/www/html/app/Models` - absolute path
- `Models` - relative path, will be transformed to `/var/www/html/app/Models` (assuming your project app directory is `/var/www/html/app`)


### base-class-name
By default generated class will be extended from `Illuminate\Database\Eloquent\Model`. To change the base class specify `base-class-name` option:
```
php artisan krlove:generate:model User --base-class-name=Some\\Other\\Base\\Model
```

### backup
Save existing model before generating a new one
```
php artisan krlove:generate:model User --backup
```
If `User.php` file already exist, it will be renamed into `User.php~` first and saved at the same directory. After than a new `User.php` will be generated.

### Other options
There are several useful options for defining several model's properties:
- `no-timestamps` - adds `public $timestamps = false;` property to the model
- `date-format` - specifies `dateFormat` property of the model
- `connection` - specifies connection name property of the model

### Overriding default options globally

Instead of spcifying options each time when executing the command you can create a config file named `eloquent_model_generator.php` at project's `config` directory with your own default values. Generator already contains its own config file at `Resources/config.php` with following options:
```php
<?php

return [
    'namespace'       => 'App',
    'base_class_name' => \Illuminate\Database\Eloquent\Model::class,
    'output_path'     => null,
    'no_timestamps'   => null,
    'date_format'     => null,
    'connection'      => null,
    'backup'          => null,
];
```
You can override them by defining `model_defaults` array at `eloquent_model_generator.php`:
```php
<?php

return [
    'model_defaults' => [
        'namespace'       => 'Some\\Other\\Namespace',
        'base_class_name' => 'Some\\Other\\ClassName',
        'output_path'     => '/full/path/to/output/directory',
        'no_timestamps'   => true,
        'date_format'     => 'U',
        'connection'      => 'other-connection',
        'backup'          => true,
    ],
];
```
### Registring custom database types
If running a command leads to an error
```
[Doctrine\DBAL\DBALException]
Unknown database type <ANY_TYPE> requested, Doctrine\DBAL\Platforms\MySqlPlatform may not support it.
```
it means that you must register your type `<ANY_TYPE>` with Doctrine.

For instance, you are going to register `enum` type and want Doctrine to treat it as `string` (You can find all existing Doctrine's types [here](http://doctrine-orm.readthedocs.io/projects/doctrine-dbal/en/latest/reference/types.html#mapping-matrix)). Add next lines at your `config/eloquent_model_generator.php`:
```
return [
    // ...
    'db_types' => [
        'enum' => 'string',
    ],
];
```
### Usage example
Table `user`:
```mysql
CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(50) NOT NULL,
  `email` varchar(100) NOT NULL,
  `role_id` int(10) unsigned NOT NULL,
  PRIMARY KEY (`id`),
  KEY `role_id` (`role_id`),
  CONSTRAINT `user_ibfk_1` FOREIGN KEY (`role_id`) REFERENCES `role` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
Command:
```
php artisan krlove:generate:model User  --table-name=user
```
Result:
```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;

/**
 * @property int $id
 * @property int $role_id
 * @property mixed $username
 * @property mixed $email
 * @property Role $role
 * @property Article[] $articles
 * @property Comment[] $comments
 */
class User extends Model
{
    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'user';

    /**
     * @var array
     */
    protected $fillable = ['role_id', 'username', 'email'];

    /**
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
     */
    public function role()
    {
        return $this->belongsTo('Role');
    }

    /**
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function articles()
    {
        return $this->hasMany('Article');
    }

    /**
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function comments()
    {
        return $this->hasMany('Comment');
    }
}
```