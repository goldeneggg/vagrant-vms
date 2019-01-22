Vagrantfile for Ubuntu 18 mysql8 server
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

