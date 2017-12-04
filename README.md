# Deploy Akeneo with Capistrano

When deploying an application, multiple steps are necessary (code pulling, dependencies updates, cache clearing, etc.).
In order to facilitate this crucial moment, you can use a tool like Capistrano.

In the following cookbook, you will learn how to deploy your project to one or multiple servers with a source code
hosted on a Git repository.

## Prerequisites
1. Make sure you have write permissions on the server you want to deploy to.
2. Make sure you can connect to the server using SSH.
3. Make sure you can access your Git repository.

## Ruby and Capistrano installation

1. Install Ruby: [installation procedure](https://www.ruby-lang.org/fr/documentation/installation/#apt).
2. Create a Gemfile at the root of the project, with the following content:

```ruby
source "https://rubygems.org"

gem 'capistrano'
gem 'capistrano-symfony'
gem 'capistrano-composer'
   ```
3. Install Bundler and the dependencies from the Gemfile

```shell
gem install bundler
bundle install
```

This file lists the bundles to install from the official Ruby repository. More info about Gemfiles [here](http://bundler.io/v1.16/guides/creating_gem.html).

3. Create a Capfile

```ruby
   # /host/path/to/you/pim/Capfile

   # Load DSL and set up stages
   require "capistrano/setup"

   # Include default deployment tasks
   require "capistrano/deploy"
   require 'capistrano/symfony'

   require "capistrano/scm/git"
   install_plugin Capistrano::SCM::Git

   Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

This file is responsible for the actual loading of the dependencies.  More info about Capfiles [here](https://github.com/capistrano/capistrano#capify-your-project).

File structure
--------------

This is the file structure from the project root.

```shell
.
├── Capfile
├── config
│   ├── deploy
│   │   ├── development.rb
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── Gemfile
```

## Common configuration

In the following configuration, please update `repo_url` to point to your git repository.

```ruby
set :application, 'acme'
set :repo_url, 'git@github.com:acme/project.git'

set :format, :pretty
set :log_level, :info
set :symfony_env,  "staging"
set :symfony_directory_structure, 3
set :sensio_distribution_version, 4
set :app_path, "app"
set :web_path, "web"

set :app_config_path, "app/config"
set :log_path, "var/logs"
set :cache_path, "var/cache"

set :branch, ENV.fetch("BRANCH", "master")
set :composer_install_flags, '--no-dev --prefer-dist --no-interaction --optimize-autoloader'

set :linked_files, %w{app/config/parameters.yml}
set :linked_dirs, %w{var/logs web/uploads}

set :controllers_to_clear, ["app_*.php"]

set :symfony_console_path, "bin/console"
set :symfony_console_flags, "--no-debug"

set :keep_releases, 3

after 'deploy:finishing', 'deploy:cleanup'
after 'deploy:updated', 'symfony:assets:install'

namespace :deploy do
    desc "Regenerating cache, assets and javascript"
    task :javascript do
        on roles(:app) do
            within release_path do
                execute *%w[ rm -rf var/cache/* ./web/bundles/* ./web/css/* ./web/js/* ]
                symfony_console('pim:installer:assets', '--env=prod')
                symfony_console('cache:warmup', '--env=prod')
                execute *%w[ rm -rf node_modules; npm install; npm run ]
                execute *%w[ yarn run webpack ]
            end
        end
    end
end

after 'deploy:updated', 'deploy:javascript'
```

You can adapt other settings according to the configuration of your server.

### Environment-specific configuration

You have to create one file per environment you have.

```ruby
set :stage, :staging

set :deploy_to, '/home/akeneo/'
set :ssh_user, 'akeneo'
server 'akeneo.tld', user: fetch(:ssh_user), roles: %w{web app db}
```
Remote file structure
---------------------
The remote file structure will be the following, with `/deploy/to` being the value of `:deploy_to`.

```shell
├── current -> /deploy/to/releases/20150120114500/
├── releases
│   ├── 20150080072500
│   ├── 20150090083000
│   └── ...
├── repo
│   └── <VCS related data>
├── revisions.log
└── shared
    ├── app
    |   └── config
    |        └── parameters.yml # Create this file.
    ├── var
    |   └── logs
    └── web
        └── uploads
```

**Important** you have to create `parameters.yml` on the remote server before the first deployment.

### Deployment
You can deploy on the staging environment by running

```shell
bundle exec cap staging deploy
```

You can also specify which branch you want to deploy by using

```shell
bundle exec cap staging deploy BRANCH=hotfix
```

### See also
- [Capistrano documentation](http://capistranorb.com)
- [Capistrano repository](https://github.com/capistrano/capistrano)
- [Symfony for Capistrano](https://github.com/capistrano/symfony)
