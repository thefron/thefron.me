---
title: "Bundle Install Using Docker"
date: 2016-09-28T15:23:39+09:00
draft: false
---

여러개의 ruby 프로젝트를 작업하다보면 rbenv, rvm 등으로 버전별, 프로젝트별 ruby와 gem들을(rvm의 경우 gemset) 관리해야하는 귀찮음이 생긴다.

지난번에 로컬에 프로젝트 관련 gem을 깔지 않기로 결심했으니 rubygems의 디펜던시를 기록하는 `Gemfile.lock` 파일 역시 docker를 이용해 업데이트를 하기로 했다.

아래와 같은 `Gemfile`이 있다고 하자.

```ruby
# Gemfile
ruby '2.3.1'
source 'https://rubygems.org'

gem 'sinatra'
```

위 Gemfile에서 ruby 버전을 2.3.1로 명시하고 있기 때문에 rvm이든 rbenv든 이용해서 로컬에 해당 버전을 설치해 줘야 `bundle install`을 실행할 수 있다.
하지만 docker를 이용하면 원하는 ruby 버전으로 one-off container를 만들어 현재 경로를 mount한 채로 `bundle install`을 실행하면 `Gemfile.lock`을 올바르게 업데이트 할 수 있다.

```bash
docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.3.1 bundle install --jobs 2
```

```text
GEM
  remote: https://rubygems.org/
  specs:
    rack (1.6.4)
    rack-protection (1.5.3)
      rack
    sinatra (1.4.7)
      rack (~> 1.5)
      rack-protection (~> 1.4)
      tilt (>= 1.3, < 3)
    tilt (2.0.5)

PLATFORMS
  ruby

DEPENDENCIES
  sinatra

RUBY VERSION
   ruby 2.3.1p112

BUNDLED WITH
   1.13.1
```

매번 커맨드를 타이핑하기 귀찮으니 아래와 같이 쉘 함수로 넣어놨다.

```sh
# .zshrc
function docker_bundle_install () {
  docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:${1:-2.3.1} bundle install --jobs 4
}
```
