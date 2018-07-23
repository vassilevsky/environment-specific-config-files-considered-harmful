# Environment-specific Config Files Considered Harmful

There is a popular technique in software development:
create separate configuration files for different environments that an application can operate in.

Notable instances of this pattern:

### Ruby on Rails

Ruby on Rails framework creates the following files for a new application:

    config/environments/development.rb
    config/environments/production.rb
    config/environments/test.rb

They contain Ruby code like this:

```ruby
Rails.application.configure do
  # Settings specified here will take precedence over those in config/application.rb.

  # Code is not reloaded between requests.
  config.cache_classes = true

  # Eager load code on boot. This eager loads most of Rails and
  # your application in memory, allowing both threaded web servers
  # and those relying on copy on write to perform better.
  # Rake tasks automatically ignore this option for performance.
  config.eager_load = true

  # etc
end
```

### Ruby

There is a popular library called [config](https://rubygems.org/gems/config) that facilitates
reading settings from the following YAML files:

    config/settings/development.yml
    config/settings/prerelease.yml
    config/settings/production.yml
    config/settings/staging.yml
    config/settings/test.yml

They are merged with the default configuration defined in `config/settings.yml` and the resulting structure
can be read from the application.

### Elixir

Elixir projects tend to have the following configuration files (as suggested by `config/congig.exs`):

    config/development.exs
    config/production.exs
    config/staging.exs
    config/test.exs

They are loaded by `config/config.exs` after its own config directives. They contain config "stanzas" like these:

```elixir
config :logger,
  backends: [
    {LoggerFileBackend, :log_file},
    :console
  ]
config :logger, :log_file,
  path: "log/development.log",
  level: :debug,
  format: "$date $time [$level] $node $metadata $message\n",
  metadata: [:module, :pid, :log_id]
```

### Erlang

Erlang applications can be launched with the `-config <name>` argument ([docs](http://erlang.org/doc/man/config.html)).
The application will then be able to read configuration from this file.
Developers tend to create environment-specific config files like these:

    config/development.config
    config/prerelease.config
    config/production.config
    config/staging.config
    config/test.config

They contain Erlang terms like these:

```erlang
[
    {http_fetcher, [
        {connect_timeout, 2000},
        {timeout, 5000}
    ]},
    {web_frontend, [
        {port, 8080}
    ]},
].
```

Erlang config files have no merge mechanism (unlike all previous ones).
Therefore, each file has a copy of all common values in addition to specific ones.

_If you are familiar with other examples, please do add them and send me a PR. Thank you!_

## The Problem

Having maintained apps with environment-specific configs for many years,
I declare this technique a **harmful anti-pattern**.

It all comes from the way applications undergo changes.

During the lifetime of an application, you never need to change the entire configuration. Never.
It is always a change of **one thing**:
* database connection parameters
* redis addresses
* kafka nodes/topics
* logging details
* email sending settings
* some other aspect

It is always some **one aspect** of the application behavior that you need to change.
You change it and you deploy it. The application starts working differently.
Then you can think about some other aspect, change that, and so on.

The problem with env-specific configs is that they dissect application configuration in a completely wrong way.
* They group **unrelated** configuration into files.
* As a result, **related** configuration ends up in **different** files.

There are so many problems coming from this terrible approach!

* **Omissions.**
  * You add some code that should behave differently in different environments.
  * You add its accompanying config values in *some* configs but forget to add them to others
    (because there are too many of them).
  * Your code works fine on your machine.
  * It does **not** work in production. Or worse: it works, but with some *other* config!
* **Confusion.**
  * You wonder why the application behaves in a certain way (because of the config).
  * You want to check all config of **some feature**.
  * You have to open all configuration files, search each of them for related config, and remember it
    (because you can't see them all — they're in *different* files).
    * If you have a wide monitor, you can open them all in vertical splits (I used `vim -O config/*`).
      But if you need to revise configs on an 11" laptop, because production misbehaves, good luck.
  * In some cases you also need to apply *config merging logic* in your head (if you know it).
    * So many people had problems with Ruby config's YAML merging logic!
  * You get super confused. Your brain energy is wasted.
* **Duplication.**
  * You copy configs from one env-specific file to another (because it's the easiest way).
  * Much later, when you need to change some config, you need to find all copies of old config.
  * If you neglect some copies of the old config, they will stay old.
  * The application starts misbehaving in *some* environments.
* **Leftovers.**
  * You delete code that used some config values.
  * You forget to delete the config values from *some* of the config files (because there are too many of them).
  * Much later, you (or somebody else) see the values in *some* configs.
  * You wonder where they're used and whether or not they're needed.
  * You waste unrelated development time and brain energy.

_If you have something to add to this list, please do, and send me a PR._

## The Solution

You need to do what you already do anyway inside your brain when you analyse the configs:

**Put configuration of one aspect in one place.**

I would sometimes do exactly that to understand which config for a certain component I get in the end.
I literally copy-pasted all component configs from all env-specific files into one (temporary) file
to have it all before my eyes, cleaned it up, and then I could understand it.

Why not make it permanent?

There already exist good examples of this component-specific approach!

* Ruby on Rails
  * `config/initializers/*.rb`
    * Predictable place where I can find configuration of each component.
    * One component in one file.
    * I can put any other configuration there.
  * `config/database.yml`
    * All database-related stuff in one place.
  * `config/cable.yml`
    * All ActionCable-related stuff in one place.
  * `config/storage.yml`
    * All ActiveStorage-related stuff in one place.
* Ruby
  * `Gemfile`
    * All dependencies in one file.
    * Sets of gems for different environments can be made with groups.

You need to do the same and put one component configuration into one file.

It's not strictly necessary to dissect configuration into component files though.
You can have it all in one file. It'll be easier to search.

* Elixir
  * `config/config.exs`
    * Mix loads this single file everywhere.
    * Loading of env-specific files is commented out by default.
    * I can put all configuration in this file and delete the env-specific import.
    * I can create component-specific configs and import them instead if I want to.
* Erlang
  * `sys.config`
    * Erlang loads only this file by default.

### Erlang

It is not that easy to get rid of env-specific configs in Erlang projects
because library authors kinda expect them.
They simply read their library configs from project config like so:

```erlang
% hackney/src/hackney_connect.erl
application:get_env(hackney, use_default_pool)

% lager/src/lager_app.erl
application:get_env(lager, handlers)

% pooler/src/pooler_sup.erl
application:get_env(pooler, pools)

% etc
```

They expect to have configuration relevant to the current environment
and so the application maintainer has to "shuffle" config files.

The solution seems to be:
create configuration for such libraries at run time, before starting them,
with `application:set_env`:

```erlang
environment() -> os:getenv("ENV", "development").

start_lager() ->
    application:load(lager),
    application:set_env(lager, handlers, lager_handlers(environment())),
    lager:start().

lager_handlers("development") -> [console_handler(), file_handler()];
lager_handlers(_OtherEnv)     ->                    [file_handler()].

console_handler() -> {lager_console_backend, [debug]}.
file_handler()    -> {lager_file_backend, [{file, "node.log"}, {level, log_level(environment())}]}.

log_level("production") -> info;
log_level(_OtherEnv)    -> debug.
```

If the configuration is bigger, perhaps it's easier to keep it in a file
(in a single one), and "preprocess" it before starting the library application:

```erlang
% application.config
[
  {library, [{production,  [{setting, value},  {other_setting, other_value}]},
             {development, [{setting, value2}, {other_setting, other_value2}]}]}
].

% application.erl
environment() -> os:getenv("ENV", development).

start_library() ->
    EnvSettings = application:get_env(library, environment()),
    [application:set_env(library, Setting, Value) || {Setting, Value} <- EnvSettings ].
```

Or maybe even env-specific values for *some* of the settings:

```erlang
% application.config
[
  {library, [{foo, [{production, value},  {development, safe_value}]},
             {bar, [{production, value2}, {development, safe_value2}]}]}
].

% application.erl
environment() -> os:getenv("ENV", development).

start_library() ->
    Settings = application:get_all_env(library),
    [application:set_env(library, Setting, proplists:get(environment(), EnvValues)) || {Setting, EnvValues} <- Settings ].
```

Even a single big config file is better than many env-specific files.

## Ugliness

The biggest obstacle you'll meet on the path of extracting component-specific configs is **perceived ugliness**.

```ruby
# config/initializers/sidekiq.rb

sidekiq_redis =
  case Rails.env
  when 'staging'     then {url: 'redis://redis.staging.example.io/0'}
  when 'production'  then {url: 'redis://redis.example.io/0'}
  when 'development' then {url: 'redis://localhost/0'}
  when 'test'        then ConnectionPool.new { MockRedis.new }
  else fail(Rails.env)
  end

Sidekiq.configure_client do |config|
  config.redis = sidekiq_redis
end
```

You will inevitably have to write switches on environments in every component config.
They will most likely look ugly.

Hear me out on this one:

**Environmental differences should be ugly.**

Any difference between environments is a **risk**. It is a potential source of bugs and downtime.

An ideal application behaves absolutely identical in any environment.
You must strive to make your applications as close to the ideal as possible.
You will have better sleep.

Therefore, any difference between environments **should stand out**.
It should be ugly, it should pop into your eyes every time you see it, and it should make you want to get rid of it.

Get rid of it not by shoveling the differences into configs where they cannot be seen (they can).
Actually eliminate differences in environments.
For example, use Service Discovery. Ask your ops friends.

## Why Configuration Tho?

A more general thought to consider: why create configs at all?

In software developent, everything should exist for a reason.
Yet, it often seems to me that ever since an application starts to have a config file, everybody starts filling it
with values for absolutely no reason whatsoever.
The logic seems to be "Hey, it's a primitive value, so it goes into config!".

This logic is wrong.

You have to have an answer to the question "Why is this value not in code (but in a separate, "configuration" file)?".

There are a few reasons why a value used in an application should go into a separate file:
  * Bussiness requirements are easier to check against a config file (not all QA people know the programming language).
  * It is easier to show configs (rather than code) to ops people for verification.
  * You need the ability to change the value and reload the application
    * on the server
    * without a deploy
    * without app recompilation.

_If you can think of any other reasons, please do add them to the list and send me a PR._

If, for a particular value, none of the reasons above are true, it is better to put it into source code,
closer to the logic. If you need to change it later, it will be easier to find.


Keep your configs organized, live long, and prosper!

_You can improve this article and send me a PR._
