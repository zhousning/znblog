---
layout: post
title: cancancan
text: cancancan权限控制
lang: ruby
type: gem 
categories: gems 
---

# cancancan

https://github.com/CanCanCommunity/cancancan

http://deveede.logdown.com/posts/206943-use-deviserolify-cancan-control-permissions

https://github.com/CanCanCommunity/cancancan/wiki/Separate-Role-Model

```ruby
gem 'cancancan'
```

## 配置参数

cancancan默认配置参数

​	create 指 new  create

​	read 指 index show

​	destroy 指  delete destroy

​	edit 指  edit update

```ruby
alias_action :index, :show, :to => :read
alias_action :new, :to => :create
alias_action :edit, :to => :update
```

load_and_authorize_resource 登陆且授权

load_resource 登陆  可以替代devise的authenticate user！

authorize_resource 授权

skip_load_and_authorize_resource 登陆且授权

skip_load_resource 登陆

skip_authorize_resource 授权

## 定义权限

参考文档：https://github.com/CanCanCommunity/cancancan/wiki/defining-abilities

配置了权限的controller，需要有权限才能访问，没配置的不需要权限。可以这样理解：load_auth使得所有的action不允许访问，通过can设置哪些可以被访问

```ruby
rails g cancan:ability
```

```ruby
abillity.rb
def initialize(user)
  user ||= User.new
  alias_action :create, :read, :update, :destroy, :to => :crud

  can :crud, User
  can :invite, User
  can [:update, :destroy], [Article, Comment]  多个权限
end
```

```ruby
class User
  load_and_authorize_resource
  
  def index
  end
end
```

## 配置+处理未授权访问

```ruby
未授权访问抛出异常
	框架生成的错误信息
if cannot?(action, subject, *args)
   message ||= unauthorized_message(action, subject)
   raise AccessDenied.new(message, action, subject)
end

自己配置处理未经授权访问
class ApplicationController < ActionController::Base
  include CanCan::ControllerAdditions
  rescue_from CanCan::AccessDenied do |exception|
    redirect_to root_url, :alert => exception.message
  end
end
```

## 授权方法

```ruby
单独授权：在各个action中添加
def show
  @article = Article.find(params[:id])
  authorize! :read, @article
end

应用到某一个controller的所有的action
class ArticlesController < ApplicationController
  load_and_authorize_resource

  def show
  end
end

应用到所有controller的所有action
class ApplicationController < ActionController::Base
  check_authorization
end

view
<% if can? :update, @article %>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>
```

