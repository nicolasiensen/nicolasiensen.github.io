---
layout: post
title:  "Running Rails' system tests with Docker"
share-description: This tutorial explains how to run Rails' system tests using a Docker container.
tags: ruby rails docker tutorial system-testing
---

Rails supports many different types of automated tests to give developers a safety net to prevent breaking existing behavior when enacting changes to the project.

The [Rails Guides website](https://guides.rubyonrails.org/testing.html) provides detailed documentation of all types of automated tests supported by the framework.

[System testing](https://guides.rubyonrails.org/testing.html#system-testing) is one of the options we have to cover our codebase with automated testing. System tests exercise the Rails application from the end-user perspective using a web browser, making it the slowest type of test we can write. Still, system tests give us a higher confidence level that the application works as expected compared to the other tests.

There are many moving parts involved when running a system test. [Avdi Grimm](https://twitter.com/avdi) did a great job in the blog post ["Rails 6 System Tests, from Top to Bottom"](https://avdi.codes/rails-6-system-tests-from-top-to-bottom/) where he unravels all the components involved in a system test and how they collaborate to test Rails applications from end-to-end.

The list below summarizes the blog post mentioned above, but reading the original text is highly recommended.

The list of components involved in running system tests are:

- The web browser: Chrome, Firefox, and others.
- [WebDriver](https://www.selenium.dev/documentation/webdriver/): a software program that allows us to interact with a browser programmatically. Each browser has its own WebDriver implementation, like [ChromeDriver](https://chromedriver.chromium.org/), [GeckoDriver (for Firefox)](https://firefox-source-docs.mozilla.org/testing/geckodriver/) and others. The gem [webdrivers](https://github.com/titusfortner/webdrivers) installs the WebDrivers in the host machine, and Rails includes this gem by default in the Gemfile when generating an application.
- [selenium-webdriver](https://rubygems.org/gems/selenium-webdriver/versions/2.53.4): a Ruby gem that provides an interface to interact with any WebDriver.
- [Capybara](https://github.com/teamcapybara/capybara): a Ruby gem that runs the Rails application under test and provides a better interface on top of `selenium-webdriver` to write tests that interact with a web browser.
- [MiniTest](https://github.com/seattlerb/minitest): the default test framework shipped in Rails.

Running system tests from a containerized Rails application is not straightforward since, most likely, the Docker image used to run the Rails application won't have a web browser installed to execute the system test.

For example, we could install a web browser in the Docker image of the Rails application or configure Rails to use a web browser installed in the host machine. Avdi Grimm wrote about the later approach in the blog post ["Run Rails 6 system tests in Docker using a host browser"](https://avdi.codes/run-rails-6-system-tests-in-docker-using-a-host-browser/).

This tutorial will follow a different approach. We will define a separate container to host the web browser and its respective WebDriver. The gem `selenium-webdriver` will point to the address of the new container so it can interact with the browser.

![app-container-web-browser-container](/assets/img/posts/2022-02-25-rails-system-tests-with-docker/app-container-web-browser-container.png)

The image above illustrates how this strategy will work. The application container will host two processes, one running the Rails application (initialized by Capybara) and the other running the system tests. First, the system test will reach out to the web browser container to send a command (1). Then, in the web browser container, the WebDriver will receive this command and execute the action in the web browser, which will perform a request to the application container (2). Finally, the Rails application will receive and respond to the request (3).

This approach is superior to the others mentioned above because
We don't bloat the Docker image of the Rails application with web browsers that are only needed to run system tests.
We don't need to install web browsers on the host machine.

The following image illustrates where each component that plays a role in running system tests will reside:

![app-container-web-browser-container-detailed](/assets/img/posts/2022-02-25-rails-system-tests-with-docker/app-container-web-browser-container-detailed.png)

[In the previous blog post](https://nicolasiensen.github.io/2022-02-01-creating-a-new-rails-application-with-docker/) we generated a new Rails application using Docker, and we will use that as a basis to get a Rails application up and running in a Docker container.

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/usr/src/app
    command: rails s -b 0.0.0.0
```

## Step 1: Prepare the Rails application for testing

We will use Rails' scaffold generator to bootstrap the [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) features to create, view, update and delete posts from our application together with its respective system tests:

```
bin/rails g scaffold post title:string body:text published:boolean
```

Once we run the command above, we can migrate the database to create the posts table:

```
bin/rails db:migrate
```

We can now try to run the system tests generated by Rails:

```
bin/rails test:system
```

The output is an error message saying, `Failed to find Chrome binary. (Webdrivers::BrowserNotFound)`. The gem `webdrivers` raised this error message because it couldn't find Chrome installed in the container.

## Step 2: Exclude the gem `webdrivers` from the project's dependencies

Since we will install Chrome and its respective WebDriver in a separate container, we don't need the gem `webdrivers` listed under the project's dependencies:

```ruby
# Remove the line below from the Gemfile
gem "webdrivers"
```

After removing the dependency and rebuilding the Docker container, we can try to rerun the system tests:

```
bin/rails test:system
```

The output this time will have a different error saying `Unable to find chromedriver. Please download the server from...`.

## Step 3: Point `selenium-webdriver` to a remote server

By default, `selenium-webdriver` will try to connect to a WebDriver in *localhost*. So let's change that to point it to a remote address:

```ruby
# test/application_system_test_case.rb
driven_by :selenium, using: :chrome, screen_size: [1400, 1400], options: {
  browser: :remote,
  url: "http://chrome-server:4444"
}
```

The host `chrome-server:4444` doesn't exist yet, but this will be the address of the Chrome WebDriver we will make available for our tests.

Let's try to rerun the system tests:

```
bin/rails test:system
```

Once more, we get an error, but this time it says `Failed to open TCP connection to chrome-server:4444 (getaddrinfo: Name or service not known)`, progress!

## Step 4: Introduce a Docker container to run the web browser

The Selenium team creates and maintains a [list of Docker images](https://hub.docker.com/u/selenium) capable of running web browsers inside containers.

The image [selenium/standalone-chrome](https://hub.docker.com/r/selenium/standalone-chrome) packages Google Chrome together with its WebDriver. We will use this image to define a new container we will call `chrome-server` and link it together with the `web` container:

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/usr/src/app
    command: rails s -b 0.0.0.0
    links:
      - chrome-server
  chrome-server:
    image: selenium/standalone-chrome:96.0
```

Let's rerun the system tests:

```
docker-compose run web bin/rails test:system
```

We should see Rails running each system test created by the scaffold command we executed early on, but they will all fail with a message like below:

```
Error:
PostsTest#test_should_create_post:
Selenium::WebDriver::Error::UnknownError: unknown error: net::ERR_CONNECTION_REFUSED
  (Session info: chrome=96.0.4664.110)
    test/system/posts_test.rb:14:in `block in <class:PostsTest>'
```

The error is coming from the `selenium-webdriver` gem. It tells us that the Chrome browser installed in the container `chrome-server` fails to access the Rails application running in the container `web`.

MiniTest captures a screenshot whenever a system test fails, so we can check what the web browser was displaying when the test failed. Rails keeps these screenshots in the folder `tmp/screenshots`, and this is the screenshot of one of the failing tests we had above:

## Step 5: Fix Capybara's configuration



```
Capybara.server_host = "0.0.0.0"
Capybara.app_host = "http://#{ENV.fetch("HOSTNAME")}:#{Capybara.server_port}"
```
