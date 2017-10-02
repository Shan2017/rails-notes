[TOC]

# Rails Notes

## 0. 说明

Rails Notes是学习Rails框架的时候，收集的一些知识和片段资料，方便自己平时查找和巩固 Rails 知识。

参考资料：
- [《Ruby on Rails 实战圣经》](https://ihower.tw/rails/index-cn.html)

## 1. 路由

- ### 重定向

在路由中可以直接设定转向：

```
get "/foo" => redirect("/bar")#!/usr/bin/env
get "/ihower" => redirect("https://ihower.tw")
```

- ### Scope 规则

scope方法可以让我们DRY我们的路由规则，将共通的controller、constraints、网址前置path和URL Helper前置名称移到scope成为参数。例如

```
get 'foo/meetings/:id', :to => 'events#show'
post 'foo/meetings', :to => 'events#create'
```

可以改写成
```
scope :controller => "events", :path => "/foo", :as => "bar" do
  get 'meetings/:id' => :show, :as => "meeting"
  post 'meetings' => :create	, :as => "meetings"
end
```
其中as会产生URL helper是bar_meeting_url和bar_meetings_url。

- ### Namespace

Namespace是Scope的一种特定应用，特别适合例如后台接口，这样就整组controller、网址path、URL Helper前置名称`都影响到：
```
namespace :admin do
  resources :projects
end
```
如此controller会是Admin::ProjectsController，网址如/admin/projects，而URL Helper如admin_projects_path这样的形式。

在Namespace下也可以设定它的首页，例如：
```
namespace :admin do
	root "projects#index"
end
```
就样连http://localhost:3000/admin/就会使用ProjectsController index action了。

- ### Collection

除了惯例中的七个Actions外，如果你需要自定群集的Action，可以这样设定：
```
resources :products do
  collection do
    get  :sold
    post :on_offer
  end
```
  # 或
```
  get  :sold, :on => :collection
  post :on_offer, :on => :collection
```
如此便会有sold_products_path和on_offer_products_path这两个URL Helper，产生出如products/sold和products/on_offer这样的网址。

- ### Member

如果需要自定对特定元素的Action：
```
resources :products do
  member do
    get :sold
  end
```
  # 或
```
  get :sold, :on => :member
```
如此会有sold_product_path(@product)这个URL Helper，产生出如products/123/sold这样的网址。

- ### 特殊条件限定

我们可以利用:constraints设定一些参数限制，例如限制:id必须是整数。
```
match "/events/show/:id" => "events#show", :constraints => {:id => /\d/}
```
另外也可以限定subdomain子网域：
```
namespace :admin do
  constraints subdomain: 'admin' do
   		 resources :photos
  end
end
```
甚至可以限定IP位置：
```
constraints(:ip => /(^127.0.0.1$)|(^192.168.[0-9]{1,3}.[0-9]{1,3}$)/) do
    match "/events/show/:id" => "events#show"
end
```

- ### RESTful路由

复数资源
```
resources :events
```

单数资源Singular Resoruce
除了一般复数型Resources，在单数的使用情境下也可以设定成单数Resource：
```
resource :map
```
特别之处在于那就没有index action了，所有的URL Helper也皆为单数形式，显示出来的网址也是单数。

但是Singular resource的档案命名仍为复数，例如maps_controller.rb

- ### 指定Controller

resource默认采用同名的controller，我们可以改指定，例如
```
resources :projects do
  resources :tasks, :controller => "project_tasks"
end
```

## 2. 控制器

- ### rescue_from

rescue_from可以在Controller中宣告救回特定的例外，改用你指定的方法处理，例如：
```
class ApplicationController < ActionController::Base

    rescue_from ActiveRecord::RecordInvalid, :with => :show_error

    protected

    def show_error
        # render something
    end

end
```

那些没有被拦截到的错误例外，使用者会看到Rails默认的500错误画面。一般来说比较常会用到rescue_from的时机，可能会是使用某些第三方函式库，该函式库可能会丢出一些例外是你想要做额外的错误处理。例如在pundit这个检查权限的套件，如果发生权限不够的情况，会丢出Pundit::NotAuthorizedError的例外，这时候就可以捕捉这个例外，改成回到首页：
```
rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

protected

def user_not_authorized
 flash[:alert] = I18n.t(:user_not_authorized)
 redirect_to(request.referrer || root_path)
end
```

- ### HTTP Basic Authenticate

Rails内建支援HTTP Basic Authenticate，可以很简单实作出认证功能：
```
class PostsController < ApplicationController
    before_action :authenticate

    protected

    def authenticate
     authenticate_or_request_with_http_basic do |username, password|
       username == "foo" && password == "bar"
     end
    end
end
```
或是这样写：
```
class PostsController < ApplicationController
    http_basic_authenticate_with :name => "foo", :password => "bar"
end
```

## 3. 模型

- ### 自订资料表名称或主键字段

资料表的名称默认就是Model类别名称的复数小写，例如Event的资料表是events、EventCategory的资料表是event_categories。不过英文博大精深，Rails转出来的复数不一定是正确的英文单字，这时候你可以修改config/initializers/inflections.rb进行订正。

如果你的资料表不使用这个命名惯例，例如连接到旧的数据库，或是主键字段不是id，也可以手动指定：
```
class Category < ApplicationRecord
  self.table_name = "your_table_name"
  self.primary_key = "your_primary_key_name"
end
```

- ### 如何自定 validation？

使用 validate 方法传入一个同名方法的 Symbol 即可。
```
validate :my_validation

private

def my_validation
    if name =~ /foo/
        errors.add(:name, "can not be foo")
    elsif name =~ /bar/
        errors.add(:name, "can not be bar")
    elsif name == 'xxx'
        errors.add(:base, "can not be xxx")
    end
end
```
在你的验证方法之中，你会使用到 errors 来将错误讯息放进去，如果这个错误是因为某一属性造成，我们就用那个属性当做 errors 的 key，例如本例的 :name。如果原因不特别属于某一个属性，照惯例会用 :base。

- ### 序列化Serialize

序列化(Serialize)通常指的是将一个物件转换成一个可被数据库储存及传输的纯文字形态，反之将这笔资料从数据库读出后转回物件的动作我们就称之为反序列(Deserialize)，Rails提供了serialize让你指定需要序列化资料的字段，任何物件在存入数据库时就会自动序列化成YAML格式，而当从数据库取出时就会自动帮你反序列成原先的物件。这个字段通常用text型态，有比较大的空间可以储存资料，然后将一个Hash物件序列化之后存进去。

常用的情境例如杂七杂八的使用者settings：
```
class User < ApplicationRecord
  serialize :settings
end

> user = User.create(:settings => { "sex" => "male", "url" => "foo" })
> User.find(user.id).settings # => { "sex" => "male", "url" => "foo" }
```
或是一些不需要数据库索引和正规化的一整包资料，例如KML轨迹资料等等。

虽然序列化很方便可以让你储存任意的物件，但是缺点是序列化资料就失去了透过数据库查询索引的功效，你无法在SQL的where条件中指定序列化后的资料。

- ### Store

Store又在包裹了序列化功能，是个简单又实用的功能，让你可以将某个字段指定储存为Hash值。举例来说，上一节的settings也可以改用store来设定：
```
class User < ApplicationRecord
  store :settings, :accessors => [:sex, :url]
end
```
特别的是其中accessors用来设定可以直接存取的属性，这样就可以像平常一样那样操作sex和url这两个属性，让我们进console实验看看:
```
> user = User.new(:sex => "male", :url => "http://example.com")
> user.sex
 => "male"
> user.url
 => "http://example.com"
> user.settings
 => {:sex => "male", :url => "http://example.com"}
```
因为store就像使用hash一样，你也可以直接操作它，加入新的资料：
```
> user.settings[:food] = "pizza"
> user.settings
 => {:sex => "male", :url => "http://example.com", :food => "pizza"}
```

## 4. 视图

- ### 自定Layout内容

除了<%= yield %>会加载Template内容之外，我们也可以预先自定一些其他的区块让Template可以置入内容。例如，要在Layout中放一个侧栏用的区块，取名叫做:sidebar：
```
<div id="sidebar">
    <%= yield :sidebar %>
</div>
<div id="content">
    <%= yield %>
</div>
```
那么在Template样板中，任意地方放:
```
<%= content_for :sidebar do %>
   <ul>
       <li>foo</li>
       <li>bar</li>
   </ul>
<% end %>
```
那么这段内容就会被置入到Layout的<%= yield :sidebar %>之中。

- ### Collection

如果是阵列的资料，像是tr或li这类会一直重复的Template元素，我们可以使用collection参数来处理，例如像以下的程式：
```
<ul>
    <% @people.each do |person| %>
        <%= render :partial => "person", :locals => { :person => person } %>
    <% end %>
<ul>
```
我们可以改写成使用collection参数来支援阵列形式：
```
<ul>
    <%= render :partial => "person", :collection => @people, :as => :person %>
<ul>
```
在_person.html.erb这个partial中，会有一个额外的索引变量person_counter纪录编号。

使用collection的好处不只是少打字而已，还有执行效能上的大大改善，Rails内部会针对这种形式做执行效率最佳化。


## 5. Turbolinks

事实上，Rails默认让每个换页都用上了Ajax技巧，这一招叫做Turbolinks，在默认的Gemfile中可以看到gem "turbolinks"，以及Layout中的data-turbolinks-track。

它的作用是让每一个超连结都只用Ajax的方式将整个body内容替换掉，这样换页时就不需要重新加载head部份的标籤，包括JavaScript和CSS等等，只重新入加载 boby 的部分，目的是可以改善换页时的速度。

也因为它没有整页重新加载，所以如果有放在application.js里面的 jQuery ready 事件处理，会变成只有第一次加载页面才执行到，换页时就失效了，所以必须改成 Turbolinks 的 turbolinks:load 事件，也就是：
```
$(document).ready(function(){
  //...
}
```
都要改写成
```
$(document).on("turbolinks:load", function(){
 //...
})
```

另外，它的快取也会影响页面内的 JavaScript，你放在 body 内的 javascript，在浏览回来时会执行两遍，并且官方认为这是 feature 不是 bug，有些 Javascript 重复执行两遍没关系，但是有些就有问题。如果想关掉这个快取功能：放个 <meta name="turbolinks-cache-control" content="no-cache"> 在 layout 的 head 之中可以关掉 Turbolinks 快取功能。

如果要针对特定的超连结关闭 Turbolinks，可以加上 data-turbolinks="false" 的属性来：
```
<div id="some-div" data-turbolinks="false">
  <a href="/">Home (without Turbolinks)</a>
</div>
```

因为 Turbolinks 影响了JavaScript的Event Bindings行为，所以在搭配一些JavaScript比较吃重的应用程式，搭配使用JavaScript 前端框架时，就会尽量移除不要使用，以免互相影响。

⚠ 总之，如果你碰到 js 灵异现象(贴上来的js code 换页回来后不执行，但是重新整理就没问题。或是跳页回来重复执行了两次等等，可以试试看拆掉 Turbolinks：把 Gemfile、applicatio.js 和 layout head 里面相关的 Turbolink 代码拿掉即可，就可以直接绕过这个大坑。

## 6. 