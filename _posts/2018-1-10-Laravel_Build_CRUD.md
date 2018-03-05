# 从头开始建立crud流程

## 数据库部分

**laravel中设计字段时一定要标准**

**尽量多用artisan自动建文件，不熟悉的操作用 php artisan help ... 查询（如php artisan help make:controller）**

### 建表

1. 配置`.env`文件，如果要用新的数据库则先在mysql中建立。

2. 运行`php artisan migrate:install`，在数据库中建立laravel的migrations记录表。

3. 运行`php artisan make:module Group`,建立Group的Model类，添加关联。

3. ~~运行`php artisan make:migration create_group_table` 建立表的生成文件。~~ 可以在上一步中的指令最后添加-m 自动生成。

4. 修改`database/migration/create_groups_table.php`中的up和down分别对应migrate时动作和rollback时动作。

5. 运行`php artisan migrate`进行迁移。

    _可能mysql会报 Syntax error 1071 Specified key was too long;参考以下解决_

    A. As outlined in the Migrations guide to fix this all you have to do is edit your AppServiceProvider.php file and inside the boot method set a default string length:

        use Illuminate\Support\Facades\Schema;
        public function boot()
        {
            Schema::defaultStringLength(191);
        }

    B. Inside config/database.php, replace this line for mysql`'engine' => null'` with `'engine' => 'InnoDB ROW_FORMAT=DYNAMIC'`,Instead of setting a limit on your string lenght.

### 填充数据
1. 用`php artisan make：seeder GroupSeeder` 建立seeder文件，修改seeder文件run方法来生成数据

        public function run()
        {
            DB::table('groups')->delete();
            for ($i = 0; $i < 50; $i++) {
                \App\Group::create([
                    'name_cn' => '中文名'.$i,
                    'name_en' => 'john doe'.$i;
                    'email' => 'jd'.$i'@123.com',
                    'group_id' => 1000 + $i%10,
                ]);
            }
        }

2. 在`/database/seeds/DatabaseSeeder.php`中的`run`方法添加要加载的seeder:    

        $this->call(GroupSeeder::class);
3. 运行`composer dump-autoload`命令把 GroupSeeder.php 加入自动加载系统，避免找不到类的错误.

4. 执行`php artisan db:seed`运行seed

### Eloquent

注意为模型添加关联时，belongsTo(other)意味着self维护外键，hasOne(other)意味着other维护外键。

简单教程 ref：https://lvwenhan.com/laravel/423.html

官方文档 ref: https://d.laravel-china.org/docs/5.5/eloquent#relationships

## Route部分

1. 添加对应的route：  
    `Route::resource('groups', 'GroupController');`  
    `Route::resource('mqusers', 'MqUserController');`  
2. 运行`php artisan make:controller GroupController -r` 建立resource模板控制器。

3. 为增删改查分别建readme.md立对应的view。
