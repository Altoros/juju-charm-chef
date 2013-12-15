This repository contains a skeleton of Juju Charm that uses chef-solo for server configuration.

This README assumes that you are familiar with Juju, Juju charms and Chef.

# Conventions

To add a hook you have to create a symlink to **stub** file

```shell
ln -s stub hook_name
```

**stub** assumes that your service cookbook is cookbooks/charm_name and relation cookbooks are cookbooks/relation-name-relation (example: cookbooks/nginx and cookbooks/reverse-proxy-relation)

# Juju helpers

This charm provides **juju-helpers** cookbook which includes some libraries and definition you can use in your hooks:

## Methods:

```ruby
juju_unit_name # => ""
juju_relation # => ""
juju_remote_uni # => ""
relation_ids(relation_name = nil) # => []
relation_list(relation_id = nil) # => []
relation_get(unit_name = nil, relation_id = nil) # => {}
config_get # => {}
unit_get(key) # => ""
juju_log(text) # => ""
```

## Definitions:

```ruby
relation_set do
  relation_id 'relation_id' # optional
  variables({})
end
```

```ruby
juju_port "port[/protocol]" do
  action :open # or :close
end
```

# Example

Here is example for Nginx charm that is able to work as a reverse proxy.

To start you have to clone this repo to your charms dir and rename it:

```shell
cd ~/charms/precise
git clone https://github.com/Altoros/juju-charm-chef
mv juju-charm-chef nginx
cd nginx
```

Edit **metadata.yaml**:

```yaml
name: nginx
summary: Small, but very powerful and efficient web server and mail proxy
maintainer: Pavel Pachkovskij <pavel.pachkovskij@altoros.com>
description: |
  <Multi-line description here>
categories:
  - cache-proxy
requires:
  reverse-proxy:
    interface: http
```

Add **config.yaml**:

```yaml
options:
  port:
    type: int
    default: 80
    description: Default nginx port.
```

Edit **CHARM_NAME** variable in **hooks/stub** file. It should match name of the Chef cookbook which would handle Juju hooks.

```shell
CHARM_NAME=nginx
```

Rename hooks/cookbooks/charm-name:

```shell
mv hooks/cookbooks/charm-name hooks/cookbooks/nginx
```

Edit **hooks/cookbooks/nginx/metadata.rb**:

```ruby
maintainer       "Altoros Systems, Inc."
maintainer_email "pavel.pachkovskij@altoros.com"
license          "GPL-3"
description      "JuJu Helpers"

version          "0.1"
name             "nginx"
depends          "juju-helpers"
```

Edit **hooks/cookbooks/nginx/recipes/install.rb**:

```ruby
package 'nginx' do
  action :install
end

juju_port config_get['port'] do
  action :open
end
```

Edit **hooks/cookbooks/nginx/recipes/start.rb**

```ruby
service 'nginx' do
  action :start
end
```

Edit **hooks/cookbooks/nginx/recipes/stop.rb**

```ruby
service 'nginx' do
  action :stop
end
```

Create definition for nginx sites **hooks/cookbooks/nginx/definitions/nginx_site.rb**:

```ruby
define :nginx_site, :action => :enable do
  site_available_path = "/etc/nginx/sites-available/#{params[:name]}"
  site_enabled_path = "/etc/nginx/sites-enabled/#{params[:name]}"

  if params[:action] == :enable
    link site_enabled_path do
      to site_available_path
      not_if { File.exists?(site_enabled_path) }
      only_if { File.exists?(site_available_path) }
      action :create
    end
  elsif params[:action] == :disable
    file site_enabled_path do
      only_if { File.exists?(site_enabled_path) }
      action :delete
    end
  end
end
```

Create site template **hooks/cookbooks/nginx/templates/default/site.conf.erb**:

```erb
upstream reverse_proxy {
<% @servers.each do |server| %>
  server <%= server['hostname'] %>:<%= server['port'] %>;
<% end %>
}

server {
  listen <%= @port %> default_server;

  keepalive_timeout 5;

  access_log /var/log/nginx.access.log;
  error_log /var/log/nginx.error.log;

  location / {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_pass http://reverse_proxy;
  }
}
```

Rename **hooks/cookbooks/relation-name-relation**:

```shell
mv hooks/cookbooks/relation-name-relation hooks/cookbooks/reverse-proxy-relation
```

Rename Juju hooks:

```shell
mv hooks/relation-name-relation-broken hooks/reverse-proxy-relation-broken
mv hooks/relation-name-relation-changed hooks/reverse-proxy-relation-changed
mv hooks/relation-name-relation-departed hooks/reverse-proxy-relation-departed
mv hooks/relation-name-relation-joined hooks/reverse-proxy-relation-joined
```

To be able to use definitions from nginx cookbook add to **hooks/cookbooks/reverse-proxy-relation/metadata.rb**:

```ruby
maintainer       "Altoros Systems, Inc."
maintainer_email "pavel.pachkovskij@altoros.com"
license          "GPL-3"
description      "JuJu Helpers"

version          "0.1"
name             "relation-name-relation"
depends          "juju-helpers"
depends          "nginx"
```

Edit **hooks/cookbooks/reverse-proxy-relation/recipes/changed.rb**

```ruby
servers = relation_list.map { |relation_id| relation_get(relation_id) }.reject do |server|
  server['hostname'].blank? || server['port'].blank?
end

if servers.present?
  template "/etc/nginx/sites-available/#{juju_relation}" do
    cookbook 'nginx'
    source 'site.conf.erb'
    owner 'root'
    group 'root'
    variables({
      servers: servers,
      port: config_get['port']
    })
  end

  nginx_site juju_relation do
    action :enable
  end

  nginx_site 'default' do
    action :disable
  end

  service 'nginx' do
    action :restart
  end
end
```

Edit **hooks/cookbooks/reverse-proxy-relation/recipes/broken.rb**

```ruby
nginx_site juju_relation do
  action :disable
end

nginx_site 'default' do
  action :enable
end

service 'nginx' do
  action :restart
end
```

Edit **hooks/cookbooks/reverse-proxy-relation/recipes/departed.rb**

```ruby
servers = relation_list.map { |relation_id| relation_get(relation_id) }.reject do |server|
  server['hostname'].blank? || server['port'].blank?
end

if servers.present?
  template "/etc/nginx/sites-available/#{juju_relation}" do
    cookbook 'nginx'
    source 'site.conf.erb'
    owner 'root'
    group 'root'
    variables({
      servers: servers,
      port: config_get['port']
    })
  end

  service 'nginx' do
    action :restart
  end
end
```

## Using it

```shell
juju bootstrap
juju deploy wordpress
juju deploy mysql
juju deploy --repository ~/charms local:nginx
juju add-relation wordpress mysql
juju add-relation wordpress nginx
juju expose nginx
```
