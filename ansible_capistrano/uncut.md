# Ansible et Capistrano

La Bo[a]te - 23 septembre 2015

## Scénario

* Fin avril Heroku annonce la fin des "free dynos" tels que nous les connaissons
* Notre site tourne sur cet hébergement depuis plus d'un an

## Contexte

* On ne peut faire pointer un enregistrement DNS A de manière durable
* Le déploiement prend une plombe
  * @mmaayylliiss déploie tous les jours, #CopierCreer

## Le déploiement tranquille

* Automatisons la mise en place du serveur
* Automatisons le déploiement du site

### Mes dépendances logicielles

* Nginx
* Ruby
* Node.js
* Git
* Varnish

### Ansible

L'installation de serveurs sans les mains !

Installer via pip :

`pip install ansible`

#### Ansible Galaxy

https://galaxy.ansible.com

On installe un rôle comme ceci : 

`ansible-galaxy install username.rolename`

---
layout: code
title: Mon playbook
---

- hosts: wearemd
  sudo: yes
  vars_files:
    - vars.yml
  roles:
    - {role: HanXHX.nginx}
    - nginx-default
    - {role: joshualund.ruby-common}
    - {role: geerlingguy.git}
    - {role: laggyluke.nodejs}
    - {role: mauricios.Varnish}

---
layout: code
title: Des variables
---

ruby_version: 'ruby-2.1.1'
ruby_checksum: 'c843df31ae88ed49f5393142b02b9a9f5a6557453805fd489a76fbafeae88941..'
ruby_download_location: 'https://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.1.tar.gz'

varnish:
  conf_path: /etc/default/varnish
  install_dir: /etc/varnish
  packages:
    - varnish    
  version: 3
  port: 80
  storage: "malloc,256m"
  default_vcl: /etc/varnish/default.vcl
  send_timeout: 60
  vcl_dir: /etc/varnish

---
layout: code
title: Inventaire
---

[wearemd]
vps192633.ovh.net ansible_ssh_user=foo

### Vagrant

* Pour tester notre playbook de playbooks :)
* La flemme :
  * `vagrant box add https://atlas.hashicorp.com/debian/boxes/jessie64`

---
layout: code
title: Playbook vagrant
---

- hosts: all
  sudo: yes
  vars_files:
    - vars.yml
  roles:
    - {role: HanXHX.nginx}
    - nginx-default
    - {role: joshualund.ruby-common}
    - {role: geerlingguy.git}
    - {role: laggyluke.nodejs}
    - {role: mauricios.Varnish}

### Lancer la VM

`vagrant up`

### Capistrano WTF ?

### Comment ça se passe dans le cas de cette application

---
layout: code
title: Gemfile
---

gem 'capistrano'
gem 'capistrano-unicorn-nginx', github: 'Awea/capistrano-unicorn-nginx'
gem 'capistrano-bundler', github: 'capistrano/bundler'

---
layout: code
title: deploy.rb
---

lock '3.4.0'

set :application, 'wearemd_mini'
set :repo_url, 'git@github.com:wearemd/wearemd_mini.git'

set :nginx_listen, '127.0.0.1:8080'

namespace :deploy do
  before :publishing, :clear_cache do
    on roles(:all) do
      within release_path do
        execute :rake, 'RACK_ENV=production', 'assets:precompile'
      end
    end
  end

  after :published, 'varnish:clear_cache'
end

---
layout: code
title: vagrant.rb
---

server 'ip-server', user: 'vagrant', roles: %w{web app}

 set :ssh_options, {
   keys: %w(~/.ssh/id_rsa),
   forward_agent: true
 }

### Les commandes qui vont bien

```
ssh-copy-id vagrant@ip-serv
ssh-add ~/.ssh/id_rsa
bundle exec cap vagrant setup
bundle exec cap vagrant deploy
```

## L'heure de la démo !