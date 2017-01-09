---
layout: post
title: 权限管理系统
text: 配置了用户/角色/权限的权限管理系统
lang: ruby
type: project 
github: https://github.com/zhousning/permission-system-source
status: public
categories: projects 
---

# devise+rolify+cancancan动态权限配置

参考文档 https://blog.joshsoftware.com/2012/10/23/dynamic-roles-and-permissions-using-cancan/

​		https://github.com/CanCanCommunity/cancancan/wiki/Abilities-in-Database

其他权限管理gem参考资料：

pundit

https://ruby-china.org/topics/26642 

https://www.sitepoint.com/straightforward-rails-authorization-with-pundit/

https://github.com/RolifyCommunity/rolify

Devise CanCanCan rolify Tutorial

https://github.com/RolifyCommunity/rolify/wiki/Devise---CanCanCan---rolify-Tutorial

http://blog.jex.tw/blog/2015/04/12/rails-user/

http://mgleon08.github.io/blog/2016/01/21/devise-rolify-cancan/

```ruby
gem 'devise'
gem 'cancancan'
gem 'rolify'

#devise详细配置请参考devise讲解
$ rails generate devise:install
$ rails generate devise User

$ rails generate cancan:ability
$ rails generate rolify Role User
#添加与角色关联的权限permission
rails g model permission name:string subject_class:string subject_id:integer action:string description:text
rails g model role_permissionship role_id:integer permission_id:integer

$ rake db:migrate
```

## 权限规则

1.非超级管理员不能管理角色

## model

角色——用户：多对多		角色——权限：多对多

```ruby
Migration  



create_table :users do |t|
   t.references :role
end
  
  
class CreatePermissions < ActiveRecord::Migration
  def change
    create_table :permissions do |t|
      t.string :name
      t.string :subject_class
      t.integer :subject_id
      t.string :action
      t.text :description

      t.timestamps null: false
    end
  end
end
  
  
class CreateRolePermissionships < ActiveRecord::Migration
  def change
    create_table :role_permissionships do |t|
      t.integer :role_id
      t.integer :permission_id

      t.timestamps null: false
    end
  end
end

Model
class User < ActiveRecord::Base
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable
  belongs_to :role

  def super_admin?
    self.role.name == "Super Admin"
  end
end

class Role < ActiveRecord::Base
  has_many :users
  has_many :role_permissionships
  has_many :permissions, :through => :role_permissionships

  def set_permissions(permissions)
    permissions.each do |id|
      permission = Permission.find(id)
      self.permissions << permission
    end
  end
end
  
class Permission < ActiveRecord::Base
  has_many :role_permissionships
  has_many :roles, :through => :role_permissionships
end
  
class RolePermissionship < ActiveRecord::Base
  belongs_to :role
  belongs_to :permission
end
  
```

## Ability配置

#### 1.application_controller

一些字符串方法https://ihower.tw/rails/activesupport.html

[constantize](http://apidock.com/rails/String/constantize) tries [to](http://apidock.com/rails/String/to) find a declared constant with the name specified in the string. It raises a [NameError](http://apidock.com/rails/NameError) when the name is not in CamelCase or is not initialized.

```ruby
class ApplicationController < ActionController::Base
  include CanCan::ControllerAdditions

  rescue_from CanCan::AccessDenied do |exception|
    flash[:alert] = exception.message
    redirect_to root_url
  end

  protected

    def self.permission
      #提取model名
      #ProductsController转为Product
      return name = controller_name.classify.constantize rescue nil
    end

    def current_ability
      @current_ability ||= Ability.new(current_user)
    end
       
    def load_permissions
      @current_permissions = current_user.role.permissions.collect{|i| [i.subject_class, i.action]}
    end
end
```

#### 2.定义action规则

1.非action的写在私有函数中  或  2.非action的定义标识名

#### 3.创建add_permissions.rake

读取所有controller所有action。执行rake db:add_permissions向数据库写入所有controller的action

```ruby
namespace 'permissions' do
  desc "Loading all models and their methods in permissions table."
  task(:add_permissions => :environment) do
    arr = []
    controllers = Dir.new("#{Rails.root}/app/controllers").entries
    controllers.each do |entry|
      if entry =~ /_controller/
        #字符串常量化“products_controller.rb"——”ProductsController“——ProductsController
        arr << entry.camelize.gsub('.rb', '').constantize
      #扫描二级目录,有需要才加
      elsif entry =~ /^[a-z]*$/
        Dir.new("#{Rails.root}/app/controllers/#{entry}").entries.each do |x|
          if x =~ /_controller/
            arr << "#{entry.titleize}::#{x.camelize.gsub('.rb', '')}".constantize
          end
        end
      end
    end

    arr.each do |controller|
      if controller.respond_to?(:permission)
        ##给每一个controller添加可管理所有的权限
        write_permission(controller.permission, "manage", 'manage', 'manage all action in this controller') 
        controller.action_methods.each do |method|
          #过滤不是权限控制的action,如内部的辅助函数
          if method =~ /^([A-Za-z\d*]+)+([\w]*)+([A-Za-z\d*]+)$/ #add_user, add_user_info, Add_user, add_User
            name, cancan_action, action_desc = eval_cancan_action(method)
            write_permission(controller.permission, cancan_action, name, action_desc)  
          end
        end
      end
    end
  end
end

def eval_cancan_action(action)
  case action.to_s
  when "index"
    name = 'list'
    cancan_action = "index"
    action_desc = I18n.t :index
  when "show"
    name = 'show'
    cancan_action = "show"
    action_desc = I18n.t :show
  when "new", "create"
    name = 'create'
    cancan_action = "create"
    action_desc = I18n.t :create
  when "edit", "update"
    name = 'edit'
    cancan_action = "edit"
    action_desc = I18n.t :update
  when "delete", "destroy"
    name = 'delete'
    cancan_action = "destroy"
    action_desc = I18n.t :destroy
  else
    name = action.to_s
    cancan_action = action.to_s
    action_desc = "Other: " << cancan_action
  end
  return name, cancan_action, action_desc
end


def write_permission(class_name, cancan_action, name, description)
  permission  = Permission.where(["subject_class = ? and action = ?", class_name, cancan_action]).first 
  unless permission
    permission = Permission.new
    permission.subject_class =  class_name
    permission.action = cancan_action
    permission.name = name
    permission.description = description
    permission.save
  else
    permission.name = name
    permission.description = description
    permission.save
  end
end
```

#### 4.ability.rb

```
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new
    user.role.permissions.each do |permission|
      if permission.subject_class == "all"
        can permission.action.to_sym, permission.subject_class.to_sym
      else
        can permission.action.to_sym, permission.subject_class.constantize
      end
    end
  end
end
```

#### 5.seed.rb创建超级管理员

```ruby
#create super admin: admin
Role.create!(:name => "Super Admin")

Permission.create!(:subject_class => "all", :action => "manage")

role = Role.find_by_name('Super Admin')
role.permissions << Permission.where(:subject_class => 'all', :action => "manage")

user = User.new(:name => "admin", :email => "admin@qq.com", :password => "admin@qq.com", :password_confirmation => "admin@qq.com")
user.role = role
user.save!

Role.create!(:name => "Staff")

User.create(:name => "zhouning", :email => "zhouning@qq.com", :password => "zhouning@qq.com", :password_confirmation => "zhouning@qq.com", :role_id => Role.find_by_name('Staff').id)
```

#### 6.RollerController

```ruby
class RolesController < ApplicationController
  before_action :authenticate_user!
  before_filter :is_super_admin?

  def new
    @role = Role.new
    @permissions = Permission.all
    @role_permissions = @role.permissions.collect{|p| p.id}
  end

  def create
    @role = Role.new(role_params)
    @role.set_permissions(params[:permissions])
    if @role.save
      redirect_to @role
    else
      render :new
    end
  end

  def index
    @roles = Role.all
  end

  def show
    @role = Role.find(params[:id])
    @permissions = @role.permissions
  end

  def edit
    @role = Role.find(params[:id])
    @permissions = Permission.all
    @role_permissions = @role.permissions.collect{|p| p.id}
  end

  def update
    @role = Role.find(params[:id])
    @role.set_permissions(params[:permissions])
    if @role.save
      redirect_to roles_path
    end
    @permissions = Permission.all
    render 'edit'
  end

  private

    def role_params
      params.require(:role).permit(:name)
    end

    def is_super_admin?
      redirect_to root_path and return unless current_user.super_admin?
    end
end
```

#### 7.form

```ruby
= form_for @role do |f|
  .field
    = f.label :name
    = f.text_field :name
  %table
    - @permissions.each do |permission|
      %tr
        %td= permission.subject_class
        %td= permission.action
        %td= check_box_tag 'permissions[]', permission.id, @role_permissions.include?(permission.id), {array: true, class: "check_box"} 
  .actions
    = f.submit 'Save'
```

#### 8.设置权限

在需要配置权限的controller中添加

```ruby
load_and_authorize_resource
提供了load_permissions和current_ability两个辅助方法，在需要的地方添加即可
```
