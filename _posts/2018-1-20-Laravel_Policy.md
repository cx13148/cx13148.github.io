# Laravel Policy 实现权限控制

### 准备工作

1. 打开并设置laravel自带的auth中间件，可以将route分组一次性添加。

        Route::middleware(['auth',])->group(function () {
            //route code here
        });

2. 为权限系统建roles表（id，name）和对应的seeder，为user表添加一个外键，user和role的关系为多对一。 注意这时新建用户默认为id=0的default role，因为这个default值是mysql的动作，所以一定要确保roles表中default的id=0。下面这个seeder写法有点蠢，主要是auto increment的id在create时不受控制，只能先加进去再update了，暂时也没想到更好的。。。

        class RoleSeeder extends Seeder {
            public function run() {
                DB::table('roles')->delete();
                \App\Role::create([
                    'name' => 'default'
                ]);
                //set default role to use id = 0
                \App\Role::where('name','default')->update(['id'=>0]);

                \App\Role::create([
                    'name' => 'admin'
                ]);
                \App\Role::create([
                    'name' => 'guest'
                ]);
            }
        }


### 创建Policy
1. 用 `php artisan make:policy GroupPolicy --model=Group`，生成Group model增删改查的policy文件，生成的文件在`/App/Policies`中。
2. 在`/App/Providers/AuthServiceProvider`中注册policy：  

        protected $policies = [
            'App\Model' => 'App\Policies\ModelPolicy',
            'App\Group' => 'App\Policies\GroupPolicy',
        ];
3. 修改policy文件，其中的方法逻辑很简单，批准用户行为返回true，否则false。例如：

        public function create(User $user) {
            return $user->role == Role::where('name','admin')->first();
        }
        
### 使用Policy
1. **基本用法：（直接将授权判断加在各个模块中，耦合程度高，同时也比较自由）**  
    policy有以下四种使用方法，涵盖了route，controller，和view各个部分。注意$user实例会自动传给policy。

        Gate 门面：Gate::allows('update articles', $article) 和 Gate::denies('update articles', $article)。
        Controller：$this->authorize('update articles', $article)。
        Blade 模板：@can('update articles', $article) 和 @cannot('update articles', $article) 指令。
        User Model 实例：$user->can('update articles', $article) 和 $user->cannot('update articles', $article)。

    对应的，用gate需要引入`Illuminate\Support\Facades\Gate`,用user model则需要引入`use Illuminate\Support\Facades\Auth`然后用`Auth::user()`拿到当前用户的实例。
    对应的demo如下：
    **Gate：**

        $user = Auth::user();
        $group = Group::find($id);
        if(Gate::denies('update',$group)) {
            abort(403, $user->role->name);
        }

    **Controller:**

        $group = Group::find($id);
        if(!$this->authorize('view',$group)){
            abort(403, $user->role->name);
        }

    **Blade 模板:**

        @can('delete',$group)
        <form id="delete{{ $group->id }}" action="/groups/{{ $group->id }}" class="deleteResource" method="POST">{{method_field('DELETE')}}{{csrf_field()}}</form>
        <button onclick="layer.confirm('确认删除？', {btn: ['确认','取消']},
                                        function(){document.getElementById('delete{{ $group->id }}').submit();},
                                        function(){layer.msg('canceled', {});
                                    });"class="layui-btn layui-btn-danger layui-btn-sm" >删除</button>
        @endcan
    **在policy中定义before方法可以在所有策略之前执行**

        public function before($user, $ability)
        {
            if ($user->isSuperAdmin()) {
                return true;
            }
        }

    **User Model 实例:**

        $user = Auth::user();
        if($user->cant('create',Group::class)){
            abort(403, $user->role->name);
        }



    **通过中间件**

        use App\Post;
        Route::put('/post/{post}', function (Post $post) {
            // 当前用户可以更新博客...
        })->middleware('can:update,post');

2. **resource型用法：（在控制器的构造器中添加）**  
    目前看这个最方便，不用修改业务代码。

        public function __construct()
        {
            $this->authorizeResource(Group::class);
        }
--------

