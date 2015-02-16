JSON API helpers for Laravel 5
=====

Make it a breeze to create a [jsonapi.org](http://jsonapi.org/) compliant API with Laravel 5.

Installation
-----

Add `echo-it/laravel-jsonapi` to your composer.json dependency list and run `composer update`.

### Requirements

* PHP 5.4+
* Laravel 5


Using laravel-jsonapi
-----

This library is made with the concept of exposing models in mind, as found in the RESTful API approach.

In few steps you can expose your models:

1. **Create a route to direct the requests**

    In this example, we use a generic route for all models and HTTP methods:

    ```php
    Route::any('{model}/{id?}', 'ApiController@handleRequest');
    ```

2. **Create your controller to handle the request**

    Your controller is responsible to handling input, instantiating a handler class and returning the response.

    ```php
    <?php namespace App\Http\Controllers;

use EchoIt\JsonApi\Request as ApiRequest;
use EchoIt\JsonApi\ErrorResponse as ApiErrorResponse;
use EchoIt\JsonApi\Exception as ApiException;
use Request;

class ApiController extends Controller
{
    public function handleRequest($modelName, $id = null)
    {
        /**
         * Create handler name from model name
         * @var string
         */
        $handlerClass = 'App\\Handlers\\' . ucfirst($modelName) . 'Handler';

        if (class_exists($handlerClass)) {
            $method = Request::method();
            $include = ($i = Request::input('include')) ? explode(',', $i) : $i;

            $request = new ApiRequest($method, $id, $include);
            $handler = new $handlerClass($request);

            // A handler can throw EchoIt\JsonApi\Exception which must be gracefully handled to give proper response
            try {
                $res = $handler->fulfillRequest();
            } catch (ApiException $e) {
                return $e->response();
            }

            return $res->toJsonResponse(str_plural($modelName));
        }

        // If a handler class does not exist for requested model, it is not considered to be exposed in the API
        return new ApiErrorResponse(404, 404, 'Entity not found');
    }
}
    ```

3. **Create a handler for your model**

    A handler is responsible for exposing a single model.

    In this example we have create a handler which supports the following requests:

    * GET /users
    * GET /users/[id]
    * PUT /users/[id]

    ```php
    <?php namespace App\Handlers;

use Symfony\Component\HttpFoundation\Response;
use App\Models\User;

use EchoIt\JsonApi\Exception as ApiException;
use EchoIt\JsonApi\Request as ApiRequest;
use EchoIt\JsonApi\Handler as ApiHandler;

class UsersHandler extends ApiHandler
{
	const ERROR_SCOPE = 1024;

	protected static $exposedRelations = [];

	public function handleGet(ApiRequest $request)
	{
		if (empty($request->id)) {
			return $this->handleGetAll($request);
		}

		return User::find($request->id);
	}

	protected function handleGetAll(ApiRequest $request)
	{
		return User::all();
	}

	public function handlePut(ApiRequest $request)
	{
		if (empty($request->id)) {
			throw new ApiException(
				'No ID provided',
				static::ERROR_SCOPE | static::ERROR_NO_ID,
				Response::HTTP_BAD_REQUEST
			);
		}

		$updates = Request::json('users');
		$user = User::find($request->id);

		if (is_null($user)) return null;

		$user->fill($updates[0]);

		if (!$user->save()) {
			throw new ApiException(
				'An unknown error occurred',
				static::ERROR_SCOPE | static::ERROR_UNKNOWN,
				Response::HTTP_INTERNAL_SERVER_ERROR
			);
		}

		return $user;
	}
}
    ```

    > **Note:** Extend your model from `EchoIt\JsonApi\Model` rather than `Eloquent` to get the proper response for linked resources.


Current features
-----

According to [jsonapi.org](http://jsonapi.org):

* [Resource Representations](http://jsonapi.org/format/#document-structure-resource-representations) as resource objects
* [Resource Relationships](http://jsonapi.org/format/#document-structure-resource-relationships)
   * Only through [Inclusion of Linked Resources](http://jsonapi.org/format/#fetching-includes)
* [Compound Documents](http://jsonapi.org/format/#document-structure-compound-documents)


Wishlist
-----

* Nested requests to fetch relations, e.g. /users/[id]/friends
* [Resource URLs](http://jsonapi.org/format/#document-structure-resource-urls)
* Requests for multiple [individual resources](http://jsonapi.org/format/#urls-individual-resources), e.g. `/users/1,2,3`
* [Sparse Fieldsets](http://jsonapi.org/format/#fetching-sparse-fieldsets)
* [Sorting](http://jsonapi.org/format/#fetching-sorting)
* Some kind of caching mechanism
