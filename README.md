[TOC]

# Rails Notes

## 0. 说明

Rails Notes是学习Rails框架的时候，收集的一些知识和片段资料，方便自己平时查找和巩固 Rails 知识。

参考资料：
- [《Ruby on Rails 实战圣经》](https://ihower.tw/rails/index-cn.html)
- [《Ruby on Rails 指南 (v5.1.1)》](https://ruby-china.github.io/rails-guides/)

## 1. 路由

- ### routes.rb

在routes.rb里面，越上面的路由规则越优先。

get 'meetings/:id', :to => 'events#show'
post 'meetings', :to => 'events#create'
这里的events#show表示指向events controller的show action。通常会简写成：

get 'meetings/:id' => 'events#show'
其中有冒号:id的部分，会被转成一个参数params[:id]传进Controller里。

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
 或
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
 或
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

- ### Controller 命名

Rails 控制器的命名约定是，最后一个单词使用复数形式，但也有例外，比如 ApplicationController。

假设有一个stores controller的话：

档名 app/controllers/stores_controller.rb
类别名称 StoresController
如果需要将controllers档案做分类，这时候可以使用Module，将档案放在子目录下，例如后台专用的controllers：

档名 app/controllers/admin/stores_controller.rb
类别名称 Admin::StoresController

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

- ### 日志记录

`Rails.logger.debug("event: #{@event.inspect}")`

- ### 散列和数组参数

params 散列不局限于只能使用一维键值对，其中可以包含数组和嵌套的散列。若想发送数组，要在键名后加上一对空方括号（[]）：

GET /clients?ids[]=1&ids[]=2&ids[]=3
“[”和“]”这两个符号不允许出现在 URL 中，所以上面的地址会被编码成 /clients?ids%5b%5d=1&ids%5b%5d=2&ids%5b%5d=3。多数情况下，无需你费心，浏览器会代为编码，接收到这样的请求后，Rails 也会自动解码。如果你要手动向服务器发送这样的请求，就要留心了。

此时，params[:ids] 的值是 ["1", "2", "3"]。注意，参数的值始终是字符串，Rails 不会尝试转换类型。

- ### 强制使用 HTTPS 协议

有时，基于安全考虑，可能希望某个控制器只能通过 HTTPS 协议访问。为了达到这一目的，可以在控制器中使用 force_ssl 方法：

class DinnerController
  force_ssl
end
与过滤器类似，也可指定 :only 或 :except 选项，设置只在某些动作上强制使用 HTTPS：

class DinnerController
  force_ssl only: :cheeseburger
  或者
  force_ssl except: :cheeseburger
end
注意，如果你在很多控制器中都使用了 force_ssl，或许你想让整个应用都使用 HTTPS。此时，你可以在环境配置文件中设定 config.force_ssl 选项。

- ### 指定Controller的Layout

class EventsController < ApplicationController
   layout "special"
end
这样就会指定Events Controller下的Views都使用app/views/layouts/special.html.erb这个Layout，你可以加上参数:only或:except表示只有特定的Action：

class EventsController < ApplicationController
   layout "special", :only => :index
end
或是

class EventsController < ApplicationController
   layout "special", :except => [:show, :edit, :new]
end
请注意到使用字串和Symbol是不同的。使用Symbol的话，它会透过一个同名的方法来动态决定，例如以下的Layout是透过determine_layout这个方法来决定：

class EventsController < ApplicationController
   layout :determine_layout

	private

	def determine_layout
   	   ( rand(100)%2 == 0 )? "event_open" : "event_closed"
	end
end
除了在Controller层级设定Layout，我们也可以设定个别的Action使用不同的Layout，例如:

def show
   @event = Event.find(params[:id])
	render :layout => "foobar"
end
这样show Action的样板就会套用foobar Layout。更常见的情形是关掉Layout，这时候我们可以写render :layout => false。

## 3. 模型

- ### Model 命名

类别名称使用大写、单数，没有底线。而档名使用小写、单数，用底线。数据库表格名称用小写且为复数。例如：

数据库表格 line_items
档名 app/models/line_item.rb
类别名称 LineItem

- ### 自定义验证
如果内置的数据验证辅助方法无法满足需求，可以选择自己定义验证使用的类或方法。

##### 1 自定义验证类
自定义的验证类继承自 ActiveModel::Validator，必须实现 validate 方法，其参数是要验证的记录，然后验证这个记录是否有效。自定义的验证类通过 validates_with 方法调用。

```
class MyValidator < ActiveModel::Validator
  def validate(record)
    unless record.name.starts_with? 'X'
      record.errors[:name] << 'Need a name starting with X please!'
    end
  end
end

class Person
  include ActiveModel::Validations
  validates_with MyValidator
end
```

在自定义的验证类中验证单个属性，最简单的方法是继承 ActiveModel::EachValidator 类。此时，自定义的验证类必须实现 validate_each 方法。这个方法接受三个参数：记录、属性名和属性值。它们分别对应模型实例、要验证的属性及其值。

```
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
      record.errors[attribute] << (options[:message] || "is not an email")
    end
  end
end

class Person < ApplicationRecord
  validates :email, presence: true, email: true
end
```

如上面的代码所示，可以同时使用内置的验证方法和自定义的验证类。

##### 2 自定义验证方法
你还可以自定义方法，验证模型的状态，如果验证失败，向 erros 集合添加错误消息。验证方法必须使用类方法 validate（API）注册，传入自定义验证方法名的符号形式。

这个类方法可以接受多个符号，自定义的验证方法会按照注册的顺序执行。

valid? 方法会验证错误集合是否为空，因此若想让验证失败，自定义的验证方法要把错误添加到那个集合中。

```
class Invoice < ApplicationRecord
  validate :expiration_date_cannot_be_in_the_past,
    :discount_cannot_be_greater_than_total_value

  def expiration_date_cannot_be_in_the_past
    if expiration_date.present? && expiration_date < Date.today
      errors.add(:expiration_date, "can't be in the past")
    end
  end

  def discount_cannot_be_greater_than_total_value
    if discount > total_value
      errors.add(:discount, "can't be greater than total value")
    end
  end
end
```

默认情况下，每次调用 valid? 方法或保存对象时都会执行自定义的验证方法。不过，使用 validate方法注册自定义验证方法时可以设置 :on 选项，指定什么时候验证。:on 的可选值为 :create 和 :update。

```
class Invoice < ApplicationRecord
  validate :active_customer, on: :create

  def active_customer
    errors.add(:customer_id, "is not active") unless customer.active?
  end
end
```

- ### 自订资料表名称或主键字段

资料表的名称默认就是Model类别名称的复数小写，例如Event的资料表是events、EventCategory的资料表是event_categories。不过英文博大精深，Rails转出来的复数不一定是正确的英文单字，这时候你可以修改config/initializers/inflections.rb进行订正。

如果你的资料表不使用这个命名惯例，例如连接到旧的数据库，或是主键字段不是id，也可以手动指定：
```
class Category < ApplicationRecord
  self.table_name = "your_table_name"
  self.primary_key = "your_primary_key_name"
end
```

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

- ### enum 宏

enum 宏把整数字段映射为一组可能的值。

class Book < ApplicationRecord
  enum availability: [:available, :unavailable]
end
上面的代码会自动创建用于查询模型的对应作用域，同时会添加用于转换状态和查询当前状态的方法。

下面的示例只查询可用的图书
Book.available
或
Book.where(availability: :available)

book = Book.new(availability: :available)
book.available?   # => true
book.unavailable! # => true
book.available?   # => false

- ### inverse_of

Active Record 提供了 :inverse_of 选项，可以通过它明确声明双向关联：

class Author < ApplicationRecord
  has_many :books, inverse_of: 'writer'
end

class Book < ApplicationRecord
  belongs_to :writer, class_name: 'Author', foreign_key: 'author_id'
end
在 has_many 声明中指定 :inverse_of 选项后，Active Record 便能识别双向关联：

a = Author.first
b = a.books.first
a.first_name == b.writer.first_name # => true
a.first_name = 'David'
a.first_name == b.writer.first_name # => true
inverse_of 有些限制：

不支持 :through 关联；
不支持 :polymorphic 关联；
不支持 :as 选项；

- ### Scopes 作用域

Model Scopes是一项非常酷的功能，它可以将常用的查询条件宣告起来，让程式变得干净易读，更厉害的是可以串接使用。例如，我们编辑app/models/event.rb，加上两个Scopes：

class Event < ApplicationRecord
    scope :open_public, -> { where( :is_public => true ) }
    scope :recent_three_days, -> { where(["created_at > ? ", Time.now - 3.days ]) }
end

> Event.create( :name => "public event", :is_public => true )
> Event.create( :name => "private event", :is_public => false )
> Event.create( :name => "private event", :is_public => true )

> Event.open_public
> Event.open_public.recent_three_days
-> {...}是Ruby语法，等同于Proc.new{...}或lambda{...}，用来建立一个匿名方法物件

串接的顺序没有影响的，都会一并套用。我们也可以串接在has_many关联后：

> user.events.open_public.recent_three_days
接着，我们可以设定一个默认的Scope，通常会拿来设定排序：

class Event < ApplicationRecord
    default_scope -> { order('id DESC') }
end
unscoped方法可以暂时取消默认的default_scope：

Event.unscoped do
    Event.all
    # SELECT * FROM events
end
最后，Scope也可以接受参数，例如：

class Event < ApplicationRecord
    scope :recent, ->(date) { where("created_at > ?", date) }

    # 等同于 scope :recent, lambda{ |date| where(["created_at > ? ", date ]) }
    # 或 scope :recent, Proc.new{ |t| where(["created_at > ? ", t ]) }
end

Event.recent( Time.now - 7.days )
不过，笔者会推荐上述这种带有参数的Scope，改成如下的类别方法，可以比较明确看清楚参数是什么，特别是你想给默认值的时候：

class Event < ApplicationRecord
    def self.recent(t=Time.now)
        where(["created_at > ? ", t ])
    end
end

Event.recent( Time.now - 7.days )
这样的效果是一样的，也是一样可以和其他Scope做串接。

all方法可以将Model转成可以串接的形式，方便依照参数组合出不同查询，例如

fruits = Fruit.all
fruits = fruits.where(:colour => 'red') if options[:red_only]
fruits = fruits.limit(10) if limited?
可以呼叫to_sql方法观察实际ORM转出来的SQL，例如Event.open_public.recent_three_days.to_sql

有一点需要特别注意，default_scope 总是在所有 scope 和 where 之前起作用。

## 4. 视图

- ### View 命名

例如一个叫做 People 的 controller，其中的 index action：

档名 app/views/people/index.html.erb
Helper 名称 module PeopleHelper
档名 app/helpers/people_helper.rb

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

- ### content_for 方法

content_for 方法以块的方式把模板内容保存在标识符中，然后我们可以在模板或布局中把这个标识符传递给 yield 方法作为参数来调用所保存的内容。

假如应用拥有标准布局，同时拥有一个特殊页面，这个特殊页面需要包含其他页面都不需要的 JavaScript 脚本。为此我们可以在这个特殊页面中使用 content_for 方法来包含所需的 JavaScript 脚本，而不必增加其他页面的体积。

app/views/layouts/application.html.erb

<html>
  <head>
    <title>Welcome!</title>
    <%= yield :special_script %>
  </head>
  <body>
    <p>Welcome! The date and time is <%= Time.now %></p>
  </body>
</html>
app/views/articles/special.html.erb

<p>This is a special page.</p>
 
<% content_for :special_script do %>
  <script>alert('Hello!')</script>
<% end %>

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

## 6. 性能

- ### N+1 queries

N+1 queries是数据库效能头号杀手。ActiveRecord的Association功能很方便，所以很容易就写出以下的程式：
```
# model
class User < ActieRecord::Base
  has_one :car
end

class Car < ApplicationRecord
  belongs_to :user
end

# your controller
def index
  @users = User.page(params[:page])
end

# view
<% @users.each do |user| %>
 <%= user.car.name %>
<% end %>
```
我们在View中读取user.car.name的值。但是这样的程式导致了N+1 queries问题，假设User有10笔，这程式会产生出11笔Queries，一笔是查User，另外10笔是一笔一笔去查Car，严重拖慢效能。
```
SELECT * FROM `users` LIMIT 10 OFFSET 0
SELECT * FROM `cars` WHERE (`cars`.`user_id` = 1)
SELECT * FROM `cars` WHERE (`cars`.`user_id` = 2)
SELECT * FROM `cars` WHERE (`cars`.`user_id` = 3)
...
...
...
SELECT * FROM `cars` WHERE (`cars`.`user_id` = 10)
```
解决方法，加上includes：
```
# your controller
def index
  @users = User.includes(:car).page(params[:page])
end
```
如此SQL query就只有两个，只用一个就捞出所有Cars资料。
```
SELECT * FROM `users` LIMIT 10 OFFSET 0
SELECT * FROM `cars` WHERE (`cars`.`user_id` IN('1','2','3','4','5','6','7','8','9','10'))
```
如果user还有parts零件的关联资料想要一起捞出来，includes也支援hash写法：`@users = User.includes(:car => :parts ).page(params[:page])`

- ### 计数快取 Counter Cache

如果需要常计算has_many的Model有多少笔资料，例如显示文章列表时，也要显示每篇有多少留言回复。
```
<% @topics.each do |topic| %>
  主题：<%= topic.subject %>
  回复数：<%= topic.posts.size %>
<% end %>
```
这时候Rails会产生一笔笔的SQL count查询：
```
SELECT * FROM `posts` LIMIT 5 OFFSET 0
SELECT count(*) AS count_all FROM `posts` WHERE (`posts`.topic_id = 1 )
SELECT count(*) AS count_all FROM `posts` WHERE (`posts`.topic_id = 2 )
SELECT count(*) AS count_all FROM `posts` WHERE (`posts`.topic_id = 3 )
SELECT count(*) AS count_all FROM `posts` WHERE (`posts`.topic_id = 4 )
SELECT count(*) AS count_all FROM `posts` WHERE (`posts`.topic_id = 5 )
```
Counter cache功能可以把这个数字存进数据库，不再需要一笔笔的SQL count查询，并且会在Post数量有更新的时候，自动更新这个值。

首先，你必须要在Topic Model新增一个字段叫做posts_count，依照惯例是_count结尾，型别是integer，有默认值0。
```
rails g migration add_posts_count_to_topic
```
编辑Migration：
```
class AddPostsCountToTopic < ActiveRecord::Migration[5.1]
  def change
    add_column :topics, :posts_count, :integer, :default => 0

    Topic.pluck(:id).each do |i|
      Topic.reset_counters(i, :posts) # 全部重算一次
    end
  end
end
```
编辑Models，加入:counter_cache => true：
```
class Topic < ApplicationRecord
  has_many :posts
end

class Posts < ApplicationRecord
  belongs_to :topic, :counter_cache => true
end
```

这样同样的@topic.posts.size程式，就会自动变成使用@topic.posts_count，而不会用SQL count查询一次。

- ### Batch finding

如果需要捞出全部的资料做处理，强烈建议最好不要用all方法，因为这样会把全部的资料一次放进内存中，如果资料有成千上万笔的话，效能就坠毁了。解决方法是分次捞，每次几捞几百或几千笔。虽然自己写就可以了，但是Rails提供了Batch finding方法可以很简单的使用：
```
Article.find_each do |a|
  # iterate over all articles, in chunks of 1000 (the default)
end

Article.find_each( :batch_size => 100 ) do |a|
  # iterate over published articles in chunks of 100
end
```
或是
```
Article.find_in_batches do |articles|
  articles.each do |a|
    # articles is array of size 1000
  end
end

Article.find_in_batches( :batch_size => 100 ) do |articles|
  articles.each do |a|
    # iterate over all articles in chunks of 100
  end
end
```

## 7. 快取

- ### View 快取

Fragment caching可以只快取HTML中的一小段元素，我们可以自由选择要快取的区块，例如侧栏或是选单等等，让我们有最大的弹性。也因为这种快取发生在View中，所以我们必须把快取程式放进View中，用cache包起来要快取的Template：
```
<% cache [@events] do %>
  All events:
	<% @events.each do |event| %>
		<%= event.name %>
	<% end %>
<% end %>
```
cache的参数是拿来当作快取Key的物件或名称，我们也可以多加一些名称来识别。Rails会自动将ActiveRecord物件的最后更新时间、你给的客制名称，加上Template的内容杂凑自动产生出一个快取Key。
```
<% cache [:popular, @events] do %>
  All popular events:
<% end %>
```
注意是因为有ActiveRecord的Lazy Load特性，所以写在Controller Action里的ActiveRecord Query才不会立即送出，而是到真正使用的时候(也就是在Fragment cache范围里)才会实际发出SQL查询。如果真没有办法利用到Lazy Load的特性，例如不是ActiveRecord的情况，则可以手动使用fragment_exist?方法在Action里面检查是不是已经有快取，有的话就不要执行，例如：
```
def show
  @event = Event.find(params[:id])
  unless fragment_exist?(@event)
    @result = SomeExpenseQuery.execute(@event)
  end
end

# show.html.erb

<% cache @event do %>
  <%= @event.name %>
  <%= @result %>
<% end %>
```

- ### Russian Doll快取策略

上述cache [:list, @events]的范例中，如果其中一笔资料有更新，会造成整组@events快取资料都要重新计算，这一点很没效率。Rails支援nested的叠套方式让我们可以重用(reuse)其中的快取资料，例如：
```
<% cache [:list, @events] %>
	All events:
	<% @events.each do |event| %>
		<% cache event do %>
			<%= event.name %>
		<% end %>
	<% end %>
<% end %>
```
如果其中一笔event有更新，最外围的快取也会一起更新，但是它不会笨笨的重算每一个小event的快取，只会重算有更新的event而已，其他event则会沿用已经有的快取资料。
