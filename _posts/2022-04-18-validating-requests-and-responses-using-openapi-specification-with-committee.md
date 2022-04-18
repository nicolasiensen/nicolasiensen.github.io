---
layout: post
title:  "Validating requests and responses using OpenAPI specification with Committee"
share-description: This tutorial shows how to reuse the OpenAPI specification to validate requests and responses in a Rails application using the gem Committee.
tags: ruby rails openapi apis tutorial
---

The [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) (OAS) is a machine-readable standard for describing APIs developed by a consortium under the Linux Foundation.

There are many benefits to using OAS to describe an API. For example, I have had the experience of documenting APIs using markdown files and going through the hassle of keeping the structure of the files consistent and duplicating JSON schemas that repeat across different endpoints. Fortunately, tools like [Swagger UI](https://swagger.io/tools/swagger-ui/) and [Redoc](https://github.com/Redocly/redoc) can ingest an OAS and output consistent and interactive API documentation in HTML format.

I have to confess that API documentation was my primary motivation to use OAS in my projects, but there are many more benefits to using this standard:

- **Quick prototyping**: we can use tools like [APISprout](https://github.com/danielgtaylor/apisprout) to spin up a webserver to expose a mocked version of the API that responds to requests with examples described in the specification.
- **Productivity**: we can subscribe to services like [APIMatic](https://www.apimatic.io/continuous-code-generation/) to generate SDKs based on the API specification in multiple programming languages, something that could take weeks or even months of work from an engineering team.
- **Security**: we can bring over services like [42Crunch](https://42crunch.com/) to scan our specifications in search of security breaches.
- **Testability**: we can rely on services like [Assertible](https://assertible.com/) to test and monitor our API by ensuring it behaves as its specification prescribes.

Hopefully, my elevator pitch has convinced you that using OAS is an excellent idea for any API project. With that out of our way, let's look at a couple of challenges I faced when introducing OAS to my Ruby projects and how I overcame them.

When we use OAS, we create a JSON or YAML file following a specific structure to define endpoints, object schemas, authentication strategies, and other aspects of the API.

Creating these files is not hard, as we will see later in this tutorial. However, avoiding duplication of knowledge between the specification and the implementation of the API is not straightforward.

In one of the projects we built at [door2door](https://door2door.io/en/), the API validated the payload of incoming requests before processing them. This validation happened inside the handler of the request that checks for the presence of required attributes. After introducing OAS to this project, we noticed that the schema that specifies a valid request was duplicated between the request validation in the Ruby code and the specification.

In the same project, we had automated tests to make requests to the API and check if the response matched the expected schema using [json-schema](https://github.com/voxpupuli/json-schema). With the introduction of OAS to the project, we duplicated the response schemas between the tests and the specification.

The duplication of knowledge between implementation and specification had a pretty [bad smell](https://martinfowler.com/bliki/CodeSmell.html), but, fortunately, we came across the Ruby gem [Committee](https://github.com/interagent/committee), which was capable of transforming the OAS into the single source of truth for our API, and so obliterating all knowledge duplication.

We can add Committee to any Rack-based web application, and it will intercept every request and response entering and leaving the system.

Committee can then judge whether or not a request or response is valid according to an OAS we provide.

If a request is invalid, Committee will respond with a standard or customized error to the API consumer, preventing the request from reaching the endpoint implementation.

If a response is invalid, Committee can raise an exception or execute a block of code, giving the development team complete control over occurrences when the API responds with an unexpected schema.

In this tutorial, we will create a new API-only Rails application and define an OAS for it. We will then install, configure and test Committee to intercept and validate requests and responses.

## Part 1: Setting things up

In this first part, we will generate a new Rails application, introduce OAS to it, and install Committee to the project.

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
bundle add committee && bundle install
```

## Part 2: Validating responses with Committee

In this part, we will enable Committee to validate the API responses, catch invalid responses with our automated tests, and explore the library configuration to allow us to notify developers when their API is not behaving as expected in production.

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

For the TDD enthusiasts like me, we can use the exceptions raised by the `ResponseValidation` middleware in our tests to let the specification guide the development. The process could look like this:
1. Change the OAS
2. Run a test that exercises the endpoint in question and see it failing
3. Write the code to make the test pass
4. Refactor
5. Go back to step 1

### Step 3: Use Committee as a monitoring agent

Let's repeat the process from the previous steps and document one more endpoint in the specification:

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

We set `ignore_error: true` to prevent Committee from responding to the request with an error message.

If we start the Rails project and perform a request to the endpoint `GET /cities/:id`, we will receive a successful response from the API, even though the body doesn't conform to the specification, and Rails will log the error message.

We can further customize the `ResponseValidation` middleware with other options we can find in [Committee's official documentation](https://github.com/interagent/committee#configuration-options-1).

It is worth mentioning that if the response includes an attribute not described in the specification, Committee won't consider that an offense, and it won't raise the exception.

The validation of responses is complete. As a result, we no longer have to manually compare an endpoint's response with the expected schema in our tests. Committee will do that for us as long as we have a test making at least one request to the endpoint.

## Part 3: Validating requests with Committee

In this final part, we will enable Committee to validate incoming requests, test this capability, and customize the response given to the consumers of the API when they make an invalid request.

### Step 1: Enable Committee to validate requests

To validate requests, we have to include the `RequestValidation` middleware in the project:

```ruby
# ./config/application.rb

module App
  class Application < Rails::Application
    # ...
    config.middleware.use(Committee::Middleware::RequestValidation, schema_path: 'docs/openapi.yaml')
  end
end
```

### Step 2: Describe the request body of `POST /cities`

Similar to the response validation, Committee won't validate requests of endpoints that don't have their request described in the OAS. So let's go ahead and specify the expected request body for creating a city:

```yaml
paths:
  /cities:
    get:
      # ...
    post:
      responses:
        # ...
      requestBody:
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

If we start the Rails application and try to make a request to `POST /cities` with a missing attribute in the city object like in the payload below:

```json
{
    "city": {
        "latitude": 52.52,
        "longitude": 13.405,
        "demonym": "Berliner",
        "website": "https://www.berlin.de/en/"
    }
}
```

The API will respond with a status code `400 (Bad request)` and the following body:

```json
{
    "id": "bad_request",
    "message": "#/components/schemas/city missing required parameters: name"
}
```

### Step 3: Customize the error response

We can customize the error response the API returns to invalid requests by providing the option `error_class` in the middleware configuration.

This option takes a class that must respond to the method `#render`, which has to return an array containing three elements: the response's status code, headers, and body. We can implement our own class to customize the error response inheriting from `Committee::ValidationError`:

```ruby
# ./config/application.rb

module App
  class ValidationError < Committee::ValidationError
    def render
      [
        status,
        { "Content-Type" => "application/json" },
        [JSON.generate(error_hash)]
      ]
    end

    private

    def error_hash
      {
        error: {
          status: id,
          detail: message
        }
      }
    end
  end
  # ...
end
```

When we inherit from `Committee::ValidationError`, the instances of our error class will have access to three methods:
- `status`: the HTTP status code assigned by Committee to the response (e.g., 400).
- `id`: the description of `status` (e.g. `bad_request`).
- `message`: the error message generated by the middleware.

We can then use the class we created above in the middleware configuration:

```ruby
# ./config/application.rb

module App
  class Application < Rails::Application
    # ...
    config.middleware.use(
      Committee::Middleware::RequestValidation,
      schema_path: 'docs/openapi.yaml',
      error_class: ValidationError
    )
  end
end
```

If we restart the server and retry the same request as in step 2, we will receive a different response from the API:

```json
{
    "error": {
        "status": "bad_request",
        "detail": "#/components/schemas/city missing required parameters: name"
    }
}
```

That's it for validating requests. We can now rely on Committee to intercept invalid incoming requests and respond with an appropriate message. We no longer have to implement the validation logic in our endpoints, eliminating the duplication and making the OAS the single source of truth for request schemas.

## Final thoughts

Using OAS to describe an API is a natural choice, and with Committee, we can stop duplicating knowledge between the specification and the implementation.

We can also use Committee to support our TDD process by writing the specification first, seeing a test fail because the API response doesn't conform to the OAS, and finally changing the code to make the test pass.

I haven't benchmarked the performance impact that Committee has on my projects. Since I'm using both middlewares to intercept requests and responses, performance must have a toll, but it has been negligible.

At the end of the day, Committee is simple to set up, it is easy to understand, it includes a short list of dependencies, and the project is under active development by the community. Everything that makes a library great to add to your project.
