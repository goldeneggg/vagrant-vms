Vagrantfile for Ubuntu 17 mysql5.7 server
====================================

## Setup

* prepare `provisioners.yml`

```
$ cp provisioners_full.yml provisioners.yml
```

## Start

```ruby
$ vagrant up --provision
$ vagrant reload
```

## Stop

```
$ vagrant halt
```

