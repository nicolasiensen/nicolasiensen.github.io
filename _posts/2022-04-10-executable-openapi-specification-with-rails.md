---
layout: post
title:  "Leveraging OpenAPI specification to validate requests and test web applications"
share-description: This tutorial explains how to enforce an OpenAPI specification into an API built with Rails.
tags: ruby rails openapi testing tutorial
---

The OpenAPI Specification (OAS) is a machine-readable standard for describing APIs developed by a consortium under the Linux Foundation.

The benefits of using OAS to describe an API are vast. For example, I have seen teams documenting their APIs using markdown files and going through the hassle of keeping the structure of the files consistent and duplicating parts of the documentation that repeat across endpoints like JSON schemas. Fortunately, tools like [Swagger UI](https://swagger.io/tools/swagger-ui/) and [Redoc](https://github.com/Redocly/redoc) ease the burden of maintaining the API documentation since they can parse OAS and generate interactive API documentation as webpages we can access with a web browser.

I have to confess that API documentation was my primary motivation to use OAS in my projects, but there are many more benefits to using this standard:

- **Quick prototyping**: tools like [APISprout](https://github.com/danielgtaylor/apisprout) spin up a webserver to expose a mocked version of the API, responding to requests with examples described in the specification.
- **Productivity**: services like [APIMatic](https://www.apimatic.io/continuous-code-generation/) can generate SDKs based on the API specification in multiple programming languages.
- **Security**: services like [42Crunch](https://42crunch.com/) scan the specification in search of security breaches.
- **Testability**: services like [Assertible](https://assertible.com/) test and monitors the API by ensuring it behaves as the specification prescribes.

In practical terms, an OAS is a JSON or YAML file that specifies endpoints, object schemas, authentication strategies, and other aspects of the API.

Describing an API with OAS is not hard, as we will see later in this tutorial. However, keeping the specification up to date to follow every change we make in the API implementation is not straightforward.

On top of that, when implementing an API, we must validate the payload of incoming requests and respond with an error when they are invalid. If we implement this logic manually, we end up duplicating the knowledge of what constitutes a valid request between the API specification and the API implementation.

This is where the Ruby gem [Committee](https://github.com/interagent/committee) comes to the rescue.

![rescue](/assets/img/posts/2022-04-10/rescue.jpeg)

We can add this library to any Rack-based web application, and it will intercept every request and response entering and leaving the system.

Committee can then validate whether or not a request or response is valid according to the OAS.

If a request is invalid, Committee will respond with a standard or customized error to the API consumer, preventing the request from reaching the endpoint implementation.

If a response is invalid, Committee can raise an exception or alert an error monitoring platform so the development team can adjust the API implementation to conform to the specification.

In this tutorial, we will create a new API-only Rails application and define an OAS for it. We will then install, configure and test Committee to intercept and validate requests and responses.

## Step 1: Install Committee

Let's start by installing the Committee gem by running the command below:

```
bundler add committee && bundle install
```

## Step 2: Introduce the OpenAPI specification

As far as I know, Rails doesn't prescribe a folder where we should keep the specification of the API we are building with it. So let's go ahead and create the `docs` folder in the root path and add a file called `openapi.yaml` inside.

```yaml
# ./docs/openapi.yaml
openapi: '3.0.2'
info:
  title: My API
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

The OpenAPI specification above defines the schema of the city object that looks like the JSON document below:

```json
{
  "name": "Berlin",
  "latitude": 52.52,
  "longitude": 13.405,
  "demonym": "Berliner",
  "website": "https://www.berlin.de/en/"
}
```

We are also specifying that the API has the endpoint `GET /cities` that returns a response with status `200`, and the response body looks like the following:

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

Step 3: Enable Committee to validate responses

With Committee installed and a first endpoint documented in the specification, we can start validating the responses given by the API by including a middleware to our Rails application:

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

The scaffold generator I executed previously created a few controller tests for my API, so let's see what happens if I test the `CitiesController`:

```shell
./bin/rails test test/controllers/cities_controller_test.rb
```

The result contains a bunch of green tests, but one test fails with the following error:

```
Error:
CitiesControllerTest#test_should_get_index:
Committee::InvalidResponse: #/paths/~1cities/get/responses/200/content/application~1json/schema expected object, but received Array: [{"id"=>298486374, "name"=>"Rio de Janeiro", "latitude"=>-22.911366, "longitude"=>-43.205916, "demonym"=>"Carioca", "website"=>"http://prefeitura.rio/", "created_at"=>"2022-04-10T15:53:25.608Z", "updated_at"=>"2022-04-10T15:53:25.608Z"}, {"id"=>980190962, "name"=>"Berlin", "latitude"=>52.52, "longitude"=>13.405, "demonym"=>"Berliner", "website"=>"https://www.berlin.de/en/", "created_at"=>"2022-04-10T15:53:25.608Z", "updated_at"=>"2022-04-10T15:53:25.608Z"}]
    test/controllers/cities_controller_test.rb:9:in `block in <class:CitiesControllerTest>'
```

The test `CitiesControllerTest#test_should_get_index` makes a request to `GET /cities` and asserts that the API responds successfully.

The test failed because Committee intercepted the response of this endpoint and verified that it didn't match the specification, and raised the exception `Committee::InvalidResponse`.

To fix this test, we can change the implementation of `CitiesController#index` to respond with an object, instead of an array, containing the attribute `cities`:

```ruby
class CitiesController < ApplicationController
  def index
    @cities = City.all

    render json: { cities: @cities }
  end
end
```

We have just witnessed the first benefit provided by Committee, ensuring our API responds according to the specifications.

If we have at least one test targeting each endpoint documented in the OpenAPI specification, Committee will ensure we implemented all endpoints according to their specifications. Otherwise, it will fail the test.

It is worth mentioning that if the response includes an attribute not described in the specification, Committee won't consider that an offense, and it won't raise the exception.

Also, if an endpoint is not documented in the specification, Committee will not raise the exception. That is why all the other tests in `CitiesController` passed.

## Step 4: Add specification for one more endpoint

Let's go back to the specification and document another endpoint:

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

We added the endpoint `POST /cities` to the specification and described the response `201` with a body that looks like the following:

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

We fix our implementation by adding the `city` object to the response:

```ruby
class CitiesController < ApplicationController
  # ...
  def create
    @city = City.new(city_params)

    if @city.save
      # before: render json: @city, status: :created, location: @city
      render json: { city: @city }, status: :created, location: @city
    else
      render json: @city.errors, status: :unprocessable_entity
    end
  end
end
```

## Step 5: Change Committee set up to prevent outages

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

While doing that, I didn't want to risk the API raising exceptions because I made a mistake when describing the endpoint in the specification.

To prevent outages in our existing project, we can change the middleware options to only raise exceptions in the test environment and allow Rails to respond to a request in production even if the response doesn't conform to the specification.

Another option we can take advantage of to configure the middleware is the `error_handler` to which we can pass a lambda to be executed when Committee intercepts an invalid response.

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

If we start the Rails project and perform a request to the endpoint `GET /cities/:id`, we will receive a successful response from the API, although the body doesn't conform with the specification, and Rails will log the error message.
