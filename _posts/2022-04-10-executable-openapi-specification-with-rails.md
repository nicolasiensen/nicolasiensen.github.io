---
layout: post
title:  "Leveraging OpenAPI specification to validate requests and test web applications"
share-description: This tutorial explains how to enforce an OpenAPI specification into an API built with Rails.
tags: ruby rails openapi testing tutorial
---

The [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) (OAS) is a machine-readable standard for describing APIs developed by a consortium under the Linux Foundation.

The benefits of using OAS to describe an API are vast. For example, I have had the experience of documenting APIs using markdown files and going through the hassle of keeping the structure of the files consistent and duplicating JSON schemas that repeat across different endpoints. Fortunately, tools like [Swagger UI](https://swagger.io/tools/swagger-ui/) and [Redoc](https://github.com/Redocly/redoc) can ingest an OAS and output consistent and interactive API documentation in HTML format.

I have to confess that API documentation was my primary motivation to use OAS in my projects, but there are many more benefits to using this standard:

- **Quick prototyping**: we can use tools like [APISprout](https://github.com/danielgtaylor/apisprout) to spin up a webserver to expose a mocked version of the API that responds to requests with examples described in the specification.
- **Productivity**: we can subscribe to services like [APIMatic](https://www.apimatic.io/continuous-code-generation/) to generate SDKs based on the API specification in multiple programming languages, something that could take weeks or even months of work from an engineering team.
- **Security**: we can bring services like [42Crunch](https://42crunch.com/) over to scan our specifications in search of security breaches.
- **Testability**: we can rely on services like [Assertible](https://assertible.com/) to test and monitor our API by ensuring it behaves as its specification prescribes.

Hopefully, my elevator pitch has convinced you that using OAS is an excellent idea for any API project. With that out of our way, let's look at a couple of challenges I faced when introducing OAS to my Ruby projects and how I overcame them.

When we use OAS to describe an API, we create a JSON or YAML file following a specific structure to define endpoints, object schemas, authentication strategies, and other aspects of the API.

Creating these files is not hard, as we will see later in this tutorial. However, avoiding duplication of knowledge between the specification and the implementation of the API is not straightforward.

In one of my projects, the API checked incoming requests to validate their payload before processing them. However, after introducing OAS to this project, we noticed that the schema that specifies a valid request was duplicated between the request validation and the specification.

In the same project, we had an extended suite of automated tests that would make requests to the API and check if the response matched the expected schema. With the introduction of OAS to the project, we duplicated the response schemas between the tests and the specification.

This is where the Ruby gem [Committee](https://github.com/interagent/committee) comes to the rescue.

![rescue](/assets/img/posts/2022-04-10/rescue.jpeg)

We can add this library to any Rack-based web application, and it will intercept every request and response entering and leaving the system.

Committee can then validate whether or not a request or response is valid according to an OAS we provide.

If a request is invalid, Committee will respond with a standard or customized error to the API consumer, preventing the request from reaching the endpoint implementation.

If a response is invalid, Committee can raise an exception or alert an error monitoring platform so the development team can adjust the API implementation to conform to the specification.

In this tutorial, we will create a new API-only Rails application and define an OAS for it. We will then install, configure and test Committee to intercept and validate requests and responses.

If you are only interested in adding Committee to your project, you can skip to step 3.

## Setting up

### Step 1: Generate a new API-only Rails application

I assume you already have a development environment ready to work with Rails. If not, you can refer to the [Rails Guides](https://guides.rubyonrails.org/getting_started.html#creating-a-new-rails-project) to get started.

Remember that Committee works with other Rack-based frameworks like [Sinatra](http://sinatrarb.com/), [Hanami](https://hanamirb.org/), or [Padrino](http://padrinorb.com/).

Let's go ahead and generate our new Rails application:

```shell
rails new --api cities-api
```
Next, let's use Rails' scaffold generator to add a few endpoints to manage a database of cities:

```shell
./bin/rails g scaffold city name:string latitude:float longitude:float demonym:string website:string
```
We finalize this step by running the database migrations:

```shell
./bin/rails db:migrate
```

### Step 2: Introduce the OAS

As far as I know, Rails doesn't prescribe a folder where we should keep the specification of the API. So let's go ahead and create a new folder called `docs` in the root path and add a file named `openapi.yaml` inside.

```yaml
# ./docs/openapi.yaml
openapi: '3.0.2'
info:
  title: Cities API
  version: '1.0'
servers:
  - url: https://localhost:3000

components:
  schemas:
    city:
      type: object
      properties:
        name:
          type: string
        latitude:
          type: number
          format: float
        longitude:
          type: number
          format: float
        demonym:
          type: string
        website:
          type: string
      required:
        - name
        - latitude
        - longitude
        - demonym
        - website

paths:
  /cities:
    get:
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  cities:
                    type: array
                    items:
                      $ref: "#/components/schemas/city"
                required:
                  - cities
```

The OAS above defines the schema of the city object that gives us a JSON document like the following:

```json
{
  "name": "Berlin",
  "latitude": 52.52,
  "longitude": 13.405,
  "demonym": "Berliner",
  "website": "https://www.berlin.de/en/"
}
```

We are also specifying that the API has the endpoint `GET /cities` that returns a response with status `200`, and the response body should look like the following:

```json
{
  "cities": [
    {
      "name": "Berlin",
      "latitude": 52.52,
      "longitude": 13.405,
      "demonym": "Berliner",
      "website": "https://www.berlin.de/en/"
    }, {
      "name": "Rio de Janeiro",
      "latitude": -22.911366,
      "longitude": -43.205916,
      "demonym": "Carioca",
      "website": "http://prefeitura.rio/"
    }
  ]
}
```

### Step 3: Install Committee

The command below will add Committee to the `Gemfile` and install it:

```shell
bundler add committee && bundle install
```

## Validating responses with Committee

### Step 1: Enable Committee to validate responses

With Committee installed and a first endpoint documented in the specification, we can start validating the responses of the API by including a middleware to our Rails application:

```ruby
# ./config/application.rb
module App
  class Application < Rails::Application
    # Add the line below to include a middleware to validate responses
    config.middleware.use Committee::Middleware::ResponseValidation, schema_path: 'docs/openapi.yaml', raise: true
  end
end
```

Setting the option `raise: true` will make the middleware raise an exception whenever the response from the Rails application doesn't conform to the specification.

The scaffold generator we executed previously created a few tests for the `CitiesController`. Let's see what happens if we run them now:

```shell
./bin/rails test test/controllers/cities_controller_test.rb
```

The result contains a bunch of green tests, but one test fails with the following error:

```
Error:
CitiesControllerTest#test_should_get_index:
Committee::InvalidResponse: #/paths/~1cities/get/responses/200/content/application~1json/schema expected object, but received Array: [{"id"=>298486374, "name"=>"MyString", "latitude"=>1.5, "longitude"=>1.5, "demonym"=>"MyString", "website"=>"MyString", "created_at"=>"2022-04-15T09:50:04.043Z", "updated_at"=>"2022-04-15T09:50:04.043Z"}, {"id"=>980190962, "name"=>"MyString", "latitude"=>1.5, "longitude"=>1.5, "demonym"=>"MyString", "website"=>"MyString", "created_at"=>"2022-04-15T09:50:04.043Z", "updated_at"=>"2022-04-15T09:50:04.043Z"}]
    test/controllers/cities_controller_test.rb:9:in `block in <class:CitiesControllerTest>'
```

The test `CitiesControllerTest#test_should_get_index` makes a request to `GET /cities` and asserts that the API responds successfully.

The test failed because Committee intercepted the response of this endpoint and verified that it didn't match the specification, and raised the exception `Committee::InvalidResponse`.

To fix this test, we can change the implementation of `CitiesController#index` to respond with an object, instead of an array, containing the attribute `cities`:

```ruby
class CitiesController < ApplicationController
  def index
    @cities = City.all

    # Replace "render json: @cities" with:
    render json: { cities: @cities }
  end
end
```

The test that failed should now pass.

### Step 2: Add specification for one more endpoint

Let's repeat the same process for the endpoint `POST /cities`:

```yaml
# ./docs/openapi.yaml
# ...
paths:
  /cities:
    get:
      # ...
    post:
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                type: object
                properties:
                  city:
                    $ref: "#/components/schemas/city"
                required:
                  - city
```

We added the endpoint `POST /cities` to the specification with a successful response `201` that will include a body that looks like the following:

```json
{
  "city": {
    "name": "Berlin",
    "latitude": 52.52,
    "longitude": 13.405,
    "demonym": "Berliner",
    "website": "https://www.berlin.de/en/"
  }
}
```

If we go ahead and run the tests of `CitiesController`, we will see a new test failing with the following message:

```
Error:
CitiesControllerTest#test_should_create_city:
Committee::InvalidResponse: #/paths/~1cities/post/responses/201/content/application~1json/schema missing required parameters: city
    test/controllers/cities_controller_test.rb:15:in `block (2 levels) in <class:CitiesControllerTest>'
    test/controllers/cities_controller_test.rb:14:in `block in <class:CitiesControllerTest>'
```

Similar to the previous step, we fix our implementation by adding the `city` object to the response:

```ruby
class CitiesController < ApplicationController
  # ...
  def create
    @city = City.new(city_params)

    if @city.save
      # Replace "render json: @city, status: :created, location: @city" with:
      render json: { city: @city }, status: :created, location: @city
    else
      render json: @city.errors, status: :unprocessable_entity
    end
  end
end
```

Notice that Committee only validates the responses of the endpoints listed in the specification. This allows us to introduce OAS and Committee to an existing project and gradually create specifications for the endpoints.

### Step 3: Change Committee set up to be more sensible

Let's repeat the process from the previous step and document one more endpoint in the specification:

```yaml
# ./docs/openapi.yaml
# ...
paths:
  /cities:
    # ...
  /cities/{id}:
    get:
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  city:
                    $ref: "#/components/schemas/city"
                required:
                  - city
```

We have now documented the endpoint `GET /cities/:id` following the same approach, where all attributes are wrapped in the `city` object.

The only professional experience I have had with Committee so far was introducing the library to existing projects.

While doing that, I didn't want to risk the API raising exceptions in production because I made a mistake when describing the endpoint in the specification or because I didn't consider an edge case where the endpoint responds with a different schema.

To prevent outages in our existing project, we can change the middleware options to only raise exceptions in the test environment and allow Rails to respond to a request in production even if the response doesn't conform to the specification.

```ruby
# ./config/application
module App
  class Application < Rails::Application
    config.middleware.use(
      Committee::Middleware::ResponseValidation,
      schema_path: 'docs/openapi.yaml',
      raise: Rails.env.test?
    )
  end
end
```

Another option we can take advantage of to configure the middleware is the `error_handler` to which we can pass a block to be executed when Committee intercepts an invalid response.

In the example below, we use the `error_handler` to log the error found by Committee. However, in an actual project, we could use the `error_handler` to report the error to an error monitoring service like Raygun or Sentry so the development team would be informed that an endpoint is not conforming to the specification when used by real clients.

```ruby
# ./config/application
module App
  class Application < Rails::Application
    config.middleware.use(
      Committee::Middleware::ResponseValidation,
      schema_path: 'docs/openapi.yaml',
      raise: Rails.env.test?,
      ignore_error: true,
      error_handler: lambda { |error| Rails.logger.error(error) }
    )
  end
end
```

We set `ignore_error` to true to prevent Committee from responding to the request with an error message.

If we start the Rails project and perform a request to the endpoint `GET /cities/:id`, we will receive a successful response from the API, even though the body doesn't conform with the specification, and Rails will log the error message.

We can further customize the `ResponseValidation` middleware with other options we can find in [Committee's official documentation](https://github.com/interagent/committee#configuration-options-1).

It is worth mentioning that if the response includes an attribute not described in the specification, Committee won't consider that an offense, and it won't raise the exception.

## Validating requests with Committee

### Step 1:

## Final thoughts

---

If we have at least one test targeting each endpoint documented in the OpenAPI specification, Committee will ensure we implemented all endpoints according to their specifications. Otherwise, it will fail the test.

In my experience, the duplication between specification and implementation can happen in two places, when validating incoming requests and assessing the schema of the API responses in automated tests.

The first type of duplication happens when the API validates the schema of incoming requests while the OAS defines the same schema of the same request. In cases like this, we end up duplicating the schema of the request in the validation and the specification.

The first type of duplication happens when we use OAS to define the schema of a request while we validate incoming requests in the API implementation to verify they conform to the same schema.

The second type of duplication happens when we write automated tests to assess that the API response matches a predefined schema while using OAS to define the same schema of the same response. In this scenario, we duplicate the schema of the response in the test and the specification.
