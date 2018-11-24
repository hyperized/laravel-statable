# Statable trait for Laravel Eloquent models

This trait provides drop-in functionality to manage state and state history of an existing Eloquent Model based on [winzou/state-machine](https://github.com/winzou/state-machine) using [sebdesign/laravel-state-machine](https://github.com/sebdesign/laravel-state-machine) service provider.

## Installation

Use composer to pull in the package:
```
$ composer require iben12/laravel-statable
```
Publish the database migration and state machine config:
```
$ php artisan vendor:publish --provider="Iben\Statable\ServiceProvider"
```
Migrate the database:
```
$ php artisan migrate
```
This migration creates the table for storing history of your models as a polymorphic relation.

## Usage

#### Prerequisites
* Model class with some property holding state (we use `last_state` in the example)

#### Setup

For this manual we will use a `Post` model as example.

First you configure the SM graph. Open `config/state-machine.php` and define a new graph:
```php
return [
    'post' => [
        'class' => App\Post::class,
        'graph' => 'post',

        'property_path': 'last_state', // should extist on model

        'states' => [
            'draft',
            'published',
            'archived'
        ],
        'transitions' => [
            'publish' => [
                'from' => ['draft'],
                'to' => 'published'
            ],
            'unpublish' => [
                'from' => ['published'],
                'to' => 'draft'
            ],
            'archive' => [
                'from' => ['draft', 'published'],
                'to' => 'archived'
            ],
            'unarchive' => [
                'from' => ['archived'],
                'to' => 'draft'
            ]
        ],
        'callbacks' => [
            'history' => [
                'do' => 'StateHistoryManager@storeHistory'
            ]
        ]
    ]
]

```

Now you have to edit the `Post` model:
```php
namespace App;

use \Illuminate\Database\Eloquent\Model;
use \Iben\Statable\Statable;

class Post extends Model
{
    use Statable;

    protected function getGraph()
    {
    	return 'post'; // the SM config to use
    }
}
```

And that's it!

#### Usage
You can now access the following methods on your entity:
```php
$post = \App\Post::first();

$post->last_state; // returns current state

try {
    $post->apply('publish'); // applies transition
} catch (\SM\SMException $e) {
    abort(500, $e->getMessage()); // if transition is not allowed, throws exception
}

$post->apply('publish'); // returns boolean

$post->stateHistory()->get(); // returns PostState collection for the given Post

$post->stateHistory()->where('user_id', \Auth::id())->get(); // you can query history as any Eloquent relation
```

NOTE: The history saves the currently authenticated user, when applying a transition. This makes sense in most cases, but if you do not use the default Laravel authentication you can override the `getActorId` method to store the user with the history.

```php
class Post extends Model
{
	// ...
	
	public function getActorId()
	{
		// return id;
	}
}
```
If the model is newly created (never been saved), so it does not have an `id` when applying
a transition, history will not be saved. If you want to be sure that all transitions
are saved in history, you can add this method to your model:
```php
    protected function saveBeforeTransition()
    {
        return true;
    }
```

#### State machine
If you want to interact directly with the `StateMachine` object, call `$model->stateMachine()`.
You can find the documentation [here](https://github.com/sebdesign/laravel-state-machine).

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
