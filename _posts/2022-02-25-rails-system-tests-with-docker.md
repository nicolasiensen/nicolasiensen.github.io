---
layout: post
title:  "Running Rails' system tests with Docker"
share-description: This tutorial explains how to run Rails' system tests using a Docker container.
tags: ruby rails docker tutorial system-testing
---

Rails natively supports many different types of automated tests to give developers a safety net to prevent breaking existing behavior when enacting changes to the project.

The [Rails Guides website](https://guides.rubyonrails.org/testing.html) provides detailed documentation of all types of automated tests supported by the framework.

[System testing](https://guides.rubyonrails.org/testing.html#system-testing) is one of the options we have to cover our codebase with automated testing. System tests exercise the Rails application from the end-user perspective using a web browser. Together with other tests, we can use system testing to automate most of the quality assurance of a product, if not its entirety.

In his excellent blog post ["Rails 6 System Tests, from Top to Bottom"](https://avdi.codes/rails-6-system-tests-from-top-to-bottom/), the great [Avdi Grimm](https://twitter.com/avdi) unravels all the components involved in a system test and how they collaborate to test Rails applications from end-to-end.

The list below attempts to summarize the blog post mentioned above, but reading the original text is highly recommended:

- [MiniTest](https://github.com/seattlerb/minitest): the default test framework shipped in Rails.
- [Capybara](https://github.com/teamcapybara/capybara): a Ruby gem that can start and stop the Rails application under test and provides an interface to interact with the browser.
- [selenium-webdriver](https://rubygems.org/gems/selenium-webdriver/versions/2.53.4): a Ruby gem that provides a lower-level interface, compared to Capybara's, to interact with the browser.
- [WebDriver](https://www.selenium.dev/documentation/webdriver/): a software program that interacts with a browser. Each browser has its own WebDriver implementation, like [ChromeDriver](https://chromedriver.chromium.org/), [GeckoDriver (for Firefox)](https://firefox-source-docs.mozilla.org/testing/geckodriver/) and others. Rails includes a gem called "webdrivers" which installs the WebDrivers in the host machine.
- The web browser: Chrome, Firefox, and others.

Running system tests from a containerized Rails application is not straightforward since, most likely, the Docker image used to run the Rails application won't have a web browser installed to execute the system test.

In this tutorial, we will walk through creating a Docker-friendly setup to run Rails' system tests.

[In the previous blog post](https://nicolasiensen.github.io/2022-02-01-creating-a-new-rails-application-with-docker/) we go through how to generate a new Rails application using Docker, which we will use as a basis to get a Rails application up and running in a Docker container.

Using `rails g scaffold` to generate a CRUD for an entity gives us a few system tests we can use for the purpose of this tutorial.

```
bin/rails test:system
```

```
Failed to find Chrome binary. (Webdrivers::BrowserNotFound)
```

---

Remove

```
gem "webdrivers"
```

```
Unable to find chromedriver. Please download the server from
https://chromedriver.storage.googleapis.com/index.html and place it somewhere on your PATH.
```

---

```
driven_by :selenium, using: :chrome, screen_size: [1400, 1400], options: {
  browser: :remote,
  url: "http://chrome-server:4444"
}
```

```
Failed to open TCP connection to chrome-server:4444 (getaddrinfo: Name or service not known)
```

---

```
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

```
Selenium::WebDriver::Error::UnknownError: unknown error: net::ERR_CONNECTION_REFUSED
```

---

```
Capybara.server_host = "0.0.0.0"
Capybara.app_host = "http://#{ENV.fetch("HOSTNAME")}:#{Capybara.server_port}"
```
