<p align="center">
<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/9a/Laravel.svg/1200px-Laravel.svg.png" width="90">
<img src="https://cdn3.iconfinder.com/data/icons/ui-icons-5/16/plus-small-01-512.png" width="90">
<img src="https://avatars1.githubusercontent.com/u/33319474?s=400&v=4" width="90">
</p>

<p align="center">
<a href="https://travis-ci.org/laravel/framework"><img src="https://travis-ci.org/laravel/framework.svg" alt="Canil Status"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://poser.pugx.org/laravel/framework/license.svg" alt="License"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://awesome.re/mentioned-badge.svg" alt="Mentioned in Awesome Laravel"></a>
</p>

# Desenvolvimento de aplicação _Multi-tenant_ usando o pacote [Tenancy](https://tenancy.dev/)

## 4. Estrutura dos _tenants_
### 4.1. Estruturando nossa aplicação<a name='secao4.1'></a>
As classes necessários para que os _tenants_ funcionem são: **Colaborador**, **Cargo**, **Produto** e **Salário**, e elas se relacionam da seguinte forma:
- cada **Colaborador** cadastrado terá um **Cargo**;
- cada **Cargo** pode ser ocupado por mais de um **Colaborador**;
- os **Produtos** que cada colaborador terá acesso dependem de seu **Cargo**;
- cada **Produto** pode estar disponível a mais de um **Cargo**;
- e é claro, como os colaboradores não são relógios (pra trabalhar de graça), cada colaborador terá seu próprio **salário**.

Agora vamos "traduzir" os itens acima para a lógica de relacionamentos no Banco de Dados:
1. **cargos x colaboradores**: cada cargo pode ter vários colaboradores (`hasMany`) e cada colaborador pertence a um cargo (`belongsTo`). Basicamente se trata de uma relação do tipo 1:N e a tabela 'colaboradores' terá uma chave estrangeira apontando para a tabela 'cargos';
2. **cargos x produtos**: cada cargo pode ter vários produtos associados (`belongsToMany`), assim como cada produto pode pertencer a diferentes cargos (`belongsToMany`). Esta  é uma relação do tipo N:N, o que significa que precisaremos de uma tabela pivô para relacionar os _id_'s;
3. **colaboradores x salarios**: cada colaborador tem apenas um salário (`hasOne`) e cada salário é específico para o colaborador (`belongs`). Aqui temos a relação mais simples possível, 1:1, então precisamos apenas de uma chave estrangeira na tabela 'salarios' apontando para 'colaboradores'.

### 4.2. CRUDs
#### 4.2.1. Cargos<a name='secao4.2.1'></a>
##### 4.2.1.1 _Model_ e _migration_
Vamos começar pelo CRUD de cargos, sendo o primeiro passo criar uma _model_ com o nome `Cargo` (no singular). As _models_ são criadas por padrão na pasta `app` e com o intuito de organizar melhor os arquivos do nosso projeto, vamos alocar as _models_ relativas aos _tenants_ dentro da pasta `Models/Tenants`.

Como o prural do nome desta _model_ (convenção para nomear as tabelas) é obtido apenas adicionando a letra "S" ao final, podemos usar o comando abaixo para criar a _migration_ ao mesmo tempo.
```sh
project$ php artisan make:model Models/Tenants/Cargo -m
```
Lembre-se que as migrations comuns aos _tenants_ devem ficar na pasta `database/migrations/tenant`, então devemos movê-la. Suprimindo possíveis comentários, os arquivos criados como apresentados abaixo:
- `app/Models/Tenants/Cargo.php`
```php
<?php

namespace App\Models\Tenants;

use Illuminate\Database\Eloquent\Model;

class Cargo extends Model
{
    //
}
```
- `database/migrations/tenant/2019_11_03_170700_create_cargos_table.php`
```php
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateCargosTable extends Migration
{
    public function up(){
        Schema::create('cargos', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->timestamps();
        });
    }

    public function down(){
        Schema::dropIfExists('cargos');
    }
}
```

Agora podemos começar a adicionar código aos arquivos criados e começando pela _migration_ definimos as colunas da tabela 'cargos' no método `up()`:
```php
...
    public function up(){
        Schema::create('cargos', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('nome');
            $table->string('codigo')->unique();
            $table->string('descricao')->nullable();
            $table->boolean('status')->default(true)->nullable();
            $table->boolean('requerente')->default(true)->nullable();

            $table->softDeletes();
            $table->timestamps();
        });
    }
...
```

Na _model_ apenas adicionamos o uso do `SoftDeletes` e um vetor chamado `$fillabe` com as colunas de 'cargos' que queremos fazer atribuição em massa. Como essa é a primeira tabela que criamos, não há necessidade de definir as relações com as outras tabelas.
```php
<?php

namespace App\Models\Tenants;

use Illuminate\Database\Eloquent\Model;

class Cargo extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'nome',
        'codigo',
        'descricao',
        'status',
        'requerente',
    ];
}
```
Agora geramos as tabelas em todos os _tenants_ que ja tenham sido criados rodando o comando:
```sh
project$ php artisan tenancy:migrate
```

##### 4.2.1.2. _Controller_
Para finalizar o CRUD criamos o _controller_ de 'cargos'. Por questão da organização vamos criá-lo em uma pasta especifica apenas para os _controllers_ dos _tenants_. O próximo comando o cria dentro da pasta `Tenant` no local padrão. Adicionamos os métodos padrões e o resultado é apresentado na sequência:
```sh
project$ php artisan make:controller Tenant/CargoController
```
- `app/Http/Controllers/Tenants/CargoController.php`
```php
<?php

namespace App\Http\Controllers\Tenants;

use Illuminate\Http\Request;
use App\Http\Controllers\Controller;
use App\Http\Requests\Tenants\Cargo\DestroyCargoRequest;
use App\Http\Requests\Tenants\Cargo\ShowCargoRequest;
use App\Http\Requests\Tenants\Cargo\StoreCargoRequest;
use App\Http\Requests\Tenants\Cargo\UpdateCargoRequest;
use App\Models\Tenants\Cargo;

class CargoController extends Controller
{
    public function store(StoreCargoRequest $request){
        $cargo = Cargo::create( $request->all() );

        $cargo->save();

        return response()->json( [ 'Cargo criado.', $cargo ], 200);
    }

    public function show(ShowCargoRequest $request){
        $cargo = Cargo::find( $request->id );
        if( !$cargo ){
            return response()->json( [ 'Cargo não encontrado.', $cargo ], 400);
        }

        return $cargo;
    }

    public function update(UpdateCargoRequest $request){
        $cargo = Cargo::find( $request->id );
        if( !$cargo ){
            return response()->json( [ 'Cargo não encontrado.', $cargo ], 400);
        }

        $cargo->nome = $request->nome ? $request->nome : $cargo->nome;
        $cargo->descricao = $request->descricao ? $request->descricao : $cargo->descricao;
        $cargo->codigo = $request->codigo ? $request->codigo : $cargo->codigo;
        if( $request->status != null ){
            if( $cargo->status != $request->status ){
                $cargo->status = $request->status;
            }
        }
        if( $request->requerente != null ){
            if( $cargo->requerente != $request->requerente ){
                $cargo->requerente = $request->requerente;
            }
        }
        $cargo->save();

        return response()->json( [ 'Cargo atualizado.', $cargo ], 200);
    }

    public function destroy(DestroyCargoRequest $request){
        $cargo = Cargo::find( $request->id );
        if( !$cargo ){
            return response()->json( [ 'Cargo não encontrado.', $cargo ], 400);
        }

        $cargo->delete();

        return response()->json( [ 'Cargo deletado.', $cargo ], 200);
    }

    public function index(){
        $cargos = Cargo::all();
        if( $cargos->count() > 0 ){
            return response()->json( [ 'Cargos.', $cargos ], 200);
        }
        return response()->json( [ 'Cargos não encontrados.', $cargos ], 400);
    }

}
```
Note que para cada método deve ser criado um _formrequest_ e esse procedimento já foi realizado na **Seção 3.1** quando fizemos o _controller_ do sistema principal, portanto não vou repeti-lo. Apenas se atente a criá-los de modo a manter os arquivos organizados, no meu caso, criei os _formrequests_ de 'cargo' em uma pasta específica para eles.

##### 4.2.1.3. Rotas
Crie as rotas no arquivo `routes/web.php` como mostrado abaixo:
```php
Route::get('createCargo', 'Tenants\CargoController@store');
Route::get('showCargo', 'Tenants\CargoController@show');
Route::get('updateCargo', 'Tenants\CargoController@update');
Route::get('deleteCargo', 'Tenants\CargoController@destroy');
Route::get('toListCargos', 'Tenants\CargoController@index');
```
Para acessá-las basta seguir o padrão:
```
http://batatinha-curitiba.projeto-tenancy.local.br/<nome da rota>?<field>=<valor>&<field>=<valor>...
```

Caso você queira conferir os arquivos originais, eles podem ser acessados no _commit_ [0791ef3c27a37c63c14b82bbe17ea5a4f1d10241](https://github.com/brnocesar/multi-tenancy/commit/0791ef3c27a37c63c14b82bbe17ea5a4f1d10241). Note que no [_commit_](https://github.com/brnocesar/multi-tenancy/commit/9c3e0272924c5c22567d7a68e4b5777f30860121) seguinte a coluna 'nome' foi retirada da tabela 'cargos' e a coluna 'descricao' assumiu este papel.

#### 4.2.2. Colaboradores<a name='secao4.2.2'></a>
Vamos iniciar agora o CRUD de colaboradores, em que será praticamento refeito o procedimento da seção anterior, apenas com uma pequena difereça na criação do _model_ e da _migration_. Ao contrário do objeto anterior em que obtivemos o seu plural acrescentando a letra 's', agora precisamos adicionar as letras 'es' ao final de colaborador para obtermos seu plural, e precisamos indicar isso ao Laravel (na verdade acho que o correto é dizer que "devemos indicar isso ao _Eloquent_", mas não tenho certeza). Portanto vamos criar _model_ e _migration_ separadas, explicitando o nome 'colaboradores' em ambas.

Primeiro criamos a _model_ com o comando:
```sh
project$ php artisan make:model Models/Tenants/Colaborador
```
Após isso criamos a migration passando o nome da tabela no plural:
```sh
project$ php artisan make:migration create_colaboradores_table
```
Movemos este arquivo para a pasta `tenant`, definimos as colunas, seus tipos e valores padrão como na seção anterior, com a diferença de que precisamos definir uma coluna para a chave estrangeira que irá relacionar 'colaboradores' com 'cargos'. Além disso, note que na terceira linha do bloco abaixo temos o nome da coluna escrito corretamento no plural em português.
```php
    public function up()
    {
        Schema::create('colaboradores', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('matricula')->unique();
            $table->string('nome');
            $table->date('admissao')->default( date('Y-m-d H:i:s') );
            $table->string('cracha')->nullable();
            $table->string('cpf')->nullable();
            $table->date('nascimento')->nullable();
            $table->string('centro_custo')->nullable();
            $table->boolean('status')->default(true);
            $table->boolean('requerente')->default(false);
            $table->unsignedBigInteger('cargo_id');
            $table->foreign('cargo_id')->references('id')->on('cargos');

            $table->softDeletes();
            $table->timestamps();
        });
    }
```

Agora voltamos ao _model_, onde iremos adicionar o 'softDeletes', o nome da tabela no plural em português, o _fillable_ e a relação com a tabela 'cargos':
```php
<?php

namespace App\Models\Tenants;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use  App\Models\Tenants\Cargo;

class Colaborador extends Model
{
    use SoftDeletes;

    protected $table = 'colaboradores';

    protected $fillable = [ 'matricula', 'nome', 'empresa_id' ];

    public function cargo(){
        return $this->belongsTo(Cargo::class);
    }
}
```

Além disso devemos definir a relação com a tabela 'colaboradores' no _model_ de cargos (`app/Models/Tenants/Cargo.php`) e nos certificar de adicionar os devidos _namespaces_ em ambos os arquivos. Ao final destas alterações, basta rodar a _migration_ para todos os _tenats_ que já foram criadas.

Para o _controller_ de 'colaboradores' realizamos o mesmo procedimento da seção anterior: criamos o _controller_ no diretório `app/Http/Controllers/Tenants` e os _formrequests_ em `project/app/Http/Requests/Tenants/Colaborador`.

Criamos as rotas da mesma forma que na seção anterior:
```php
Route::get('createColaborador', 'Tenants\ColaboradorController@store');
Route::get('showColaborador', 'Tenants\ColaboradorController@show');
Route::get('updateColaborador', 'Tenants\ColaboradorController@update');
Route::get('deleteColaborador', 'Tenants\ColaboradorController@destroy');
Route::get('toListColaboradores', 'Tenants\ColaboradorController@index');
```

Você pode acessar os arquivos originais no _commit_ [5b9b57e30e2cf63068406b8fe960c0f12ca2e906](https://github.com/brnocesar/multi-tenancy/commit/5b9b57e30e2cf63068406b8fe960c0f12ca2e906).
