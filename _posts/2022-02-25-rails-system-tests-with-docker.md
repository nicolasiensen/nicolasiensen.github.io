---
layout: post
title:  "Running Rails' system tests with Docker"
share-description: This tutorial explains how to run Rails' system tests using a Docker container.
tags: ruby rails docker tutorial system-testing
---

System tests allow developers to test the user interaction with the web application.

In this tutorial, we will

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
