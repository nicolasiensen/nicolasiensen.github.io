---
layout: post
title:  "Running Rails' system tests with Docker"
share-description: This tutorial explains how to run Rails' system tests using a Docker container.
tags: ruby rails docker tutorial system-testing
---

Rails offers many options to cover the codebase with automated tests, granting developers a high level of confidence when making changes to the project.

One of the options is [system testing](https://guides.rubyonrails.org/testing.html#system-testing). System tests exercise the Rails application from the end-user perspective, which we can use to automate most of the quality assurance process of a product, if not its entirety.

Running system tests from a containerized Rails application is not straightforward since, most likely, the Docker image used to run the Rails application won't have the necessary software required to execute the system test like a web browser.

In this tutorial, we will walkthrough the creation of a Docker-friendly setup to run Rails' system tests.

[My previous blog post](https://nicolasiensen.github.io/2022-02-01-creating-a-new-rails-application-with-docker/) goes through how to generate a new Rails application using Docker, and one could use it to get a Rails application up and running in a Docker container.

Link to Avidi's blog post about all components that are part of the system test

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
