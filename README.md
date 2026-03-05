---
description: Get started with AppSignal under a few minutes
icon: bolt
metaLinks:
  canonical: /broken/pages/PbYb0GukRhiS4qCHdRal
---

# Getting Started

This page walks you through installing AppSignal in your application and authenticating it with your account so you can start monitoring right away.

***

## Adding a New Application

You can follow an installation wizard to guide you through each installation step.&#x20;

1. [Sign in](https://appsignal.com/users/sign_in) to your AppSignal account before you start this guide. \
   You can use [this link](https://appsignal.com/redirect-to/organization?to=sites/new) to start the installation wizard

{% hint style="info" %}
The wizard will first ask you what your application's language is. We currently support Ruby, Elixir & Node.js for error and performance monitoring and JavaScript for front-end error monitoring.
{% endhint %}

2. Alternatively, select the "Add app" button on the Applications overview page:![The image shows how to find the Add app button from AppSignal's Applications dashboard](<.gitbook/assets/image (1) (1).png>)

## Installation

Choose your language below and follow the steps to add AppSignal to your project.

{% tabs %}
{% tab title="Ruby" %}
```ruby
# Step 1 — Add AppSignal to your Gemfile
# Gemfile
source "https://rubygems.org"
gem "appsignal"

# Step 2 — Install the gem
bundle install

# Step 3 — Run the installer (replace with your actual Push API key)
bundle exec appsignal install YOUR_PUSH_API_KEY
```
{% endtab %}

{% tab title="Elixir" %}
```elixir
# Step 1 — Add AppSignal to your mix.exs dependencies
def deps do
  [
    {:appsignal, "~> 2.0"}
  ]
end

# Step 2 — Fetch dependencies
# mix deps.get

# Step 3 — Run the installer (replace with your actual Push API key)
# mix appsignal.install YOUR_PUSH_API_KEY
```
{% endtab %}

{% tab title="Node.js" %}
```javascript
// Step 1 — install AppSignal by adding @appsignal/nodejs to your package.json
// npm install @appsignal/nodejs

// Step 2 — Create an appsignal.cjs file to require and configure AppSignal
// appsignal.cjs
const { Appsignal } = require("@appsignal/nodejs");
 
new Appsignal({
  active: true,
  name: "<YOUR APPLICATION NAME>",
  pushApiKey: "<YOUR API KEY>",
});

//  Run your application using the --require flag 
// node --require './appsignal.cjs' index.js
```
{% endtab %}

{% tab title="Python" %}
```python
# Method 1 — Add appSignal to requirements.txt:
# appsignal

# Run the following to install dependencies
# pip install -r requirements.txt

# Method 2 — Manual installation
# Create the __appsignal__.py file
from appsignal import Appsignal

appsignal = Appsignal(
    name="My app name",
    push_api_key="my-push-api-key",
    active=True
)
```
{% endtab %}

{% tab title="Front-end (JS)" %}
```javascript
// Step 1 — add the @appsignal/javascript package to your package.json
// Then, run: 
// npm install --save @appsignal/javascript
// or: yarn add @appsignal/javascript

// Step 2 — Initialize AppSignal
// Note: use your Front-end error monitoring key here,
// NOT your server-side Push API key (found under App Settings → Push & Deploy)

// For ES Module via npm/yarn, or with import maps
import Appsignal from "@appsignal/javascript";
// For CommonJS module via npm/yarn
const Appsignal = require("@appsignal/javascript").default;
// With the JSPM.dev CDN
import Appsignal from "https://jspm.dev/@appsignal/javascript";
 
export const appsignal = new Appsignal({
  key: "YOUR FRONTEND API KEY",
});

// You can then import and use the Appsignal object in your app
// by placing this at the top to import the file created before
import { appsignal } from "./appsignal";
 
// Place your app code below this
```
{% endtab %}
{% endtabs %}

***

## Configuration & Authentication

For applications to report data to AppSignal the following minimal configurations are required. If they are not present, AppSignal will not send any data to AppSignal.com.

{% tabs %}
{% tab title="Ruby" %}
```ruby
# config/appsignal.rb
Appsignal.configure do |config|
  config.activate_if_environment(:development, :staging, :production)
  config.name = "My app"
  config.push_api_key = "1234-1234-1234"
end
```
{% endtab %}

{% tab title="Elixir" %}
```elixir
# config/config.exs
import Config

config :appsignal, :config,
  otp_app: :my_app,
  name: System.get_env("APPSIGNAL_APP_NAME"),
  push_api_key: System.get_env("APPSIGNAL_PUSH_API_KEY"),
  env: Mix.env(),
  active: true
```
{% endtab %}

{% tab title="Node.js" %}
```javascript
// Load environment variables before initializing AppSignal
require("dotenv").config(); // reads .env file before process.env is accessed

const { Appsignal } = require("@appsignal/nodejs");

const appsignal = new Appsignal({
  active: true,
  name: process.env.APPSIGNAL_APP_NAME,
  pushApiKey: process.env.APPSIGNAL_PUSH_API_KEY,
});
```
{% endtab %}

{% tab title="Python" %}
```python
import os
from dotenv import load_dotenv
from appsignal import Appsignal

load_dotenv()  # ← reads the .env file BEFORE os.getenv runs

appsignal = Appsignal(
    active=True,
    # In this example, we create an AI agent using RAG
    name="rag_agent",
    push_api_key=os.getenv("APPSIGNAL_PUSH_API_KEY"),
)
```
{% endtab %}

{% tab title="Front-end (JS)" %}
```javascript
// Front-end JS does not support file-based or env-var config.
// Pass the key inline. Use your Front-end error monitoring key
// (not the server-side Push API key).

import Appsignal from "@appsignal/javascript";

export const appsignal = new Appsignal({
  key: "YOUR_FRONTEND_API_KEY",
});
```
{% endtab %}
{% endtabs %}

### Environment variables

Your `.env` file should define the following values. The `APPSIGNAL_APP_ENV` variable controls which environment, for example `development`, `staging`, and `production`. Make sure it matches your actual deployment environment.

```shellscript
export APPSIGNAL_PUSH_API_KEY=a1b2c3d4-e5f6-7890-abcd-ef12
export APPSIGNAL_APP_NAME=AI_RAG_agent
export APPSIGNAL_APP_ENV=development
```

{% hint style="warning" %}
**Different features use different API keys.** The `APPSIGNAL_PUSH_API_KEY` (prefixed `ls-`) is your application's server-side key. If you want to ship logs to AppSignal via **syslog**, you need the application-specific syslog key instead (e.g. `pi_key=ls-89fghi23-7j45-2k9m-3n6p-4qr817s2ttw8` found in your Log Management settings). Similarly, front-end error monitoring uses its own separate key. Always double-check which key a feature requires before configuring it.
{% endhint %}

## Finding Your Keys

The images below shows where to locate and manage your keys inside the AppSignal dashboard.&#x20;

### Push key

You can find you push keys under **App Settings → Push & Deploy**.

<figure><img src=".gitbook/assets/image (1).png" alt="Image showing AppSignal&#x27;s app settings dashboard with push &#x26; deploy keys"><figcaption><p>App Settings dashboard</p></figcaption></figure>

### Log sources API keys

&#x20;If you want to ship logs to AppSignal via **syslog**, you need the application-specific log key found under **Logging → Manage sources**

{% embed url="https://www.figma.com/design/vNCKkA883JKL4l7oNIHaHZ/Rag_agent?t=JG3igSo8KIu4cYjj-1" %}
