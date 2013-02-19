
重构臃肿 ActiveRecord 模型的 7 种方式
=====================================

当团队使用 [Code Climate](https://codeclimate.com/) 来提高 Rails 程序的代码质量时，他们就会学习到如何防止模型慢慢变得臃肿。“胖模型( Fat models )” 在大应用中会导致维护问题。它仅仅比那种充斥着各种业务逻辑的凌乱的控制器好一点，但它们都违反了[单一权责原则(SRP)](http://www.objectmentor.com/resources/articles/srp.pdf)。“任何有关用户做什么” 这种并不是单一权责。

刚开始， 单一权责很容易做到。 ActiveRecord 类只处理持久化，关联关系，并不管其它东西。但是，一点点地，他开始增长。原本应该只负责持久化的对象实际上也包含了其它的业务逻辑。所以，一两年后，你的 User 类超过了 500 行，有上百个公共方法。邪恶的回调问题开始出现。

随着你的程序越来越复杂（功能越来越多), 你的目标是在一些协调的，细小的封装对象（从更高层次来说就是模块）中传递信息，就像是在平底锅底抹面粉块一样。胖模型就像是你放入锅里面的一大块面团。你要将它重构成小块，均匀地分摊业务逻辑。不断地重复这个过程，最终你会得到一系列和谐工作在一起的简单对象。

你可能觉得：

> Rails 让正确实践面向对象编程(OOP)变困难了

我过去也是这样认为的。但是做了一些探索和实践之后，我发现 Rails (这个框架)完全没有阻止我们实践面向对象编程。其实是 Rails 的约定(convention)没有刻意强调这个，或者说是，它除了 [ActiveRecord 模型](http://martinfowler.com/eaaCatalog/activeRecord.html)能处理的情况外，缺乏管理更复杂的情况的约定。幸运的是，我们能够找到 Rails 缺少的, 如何应用基于面向对象原则的最佳实践。

## 不要从胖模型中抽离混入(Mixins)

为什么呢？我避免将一个大的 ActiveRecord 类里面的一部分方法放到某个关联类或者模块里面，然后将它们混入。我有一次听到这样的说法：

> Any application with an app/concerns directory is concerning.

我同意，组合优于继承。但是，像这样使用混入就像是将混乱放到 6 个不同的抽屉然后关上它。确实，它表面上看去干净多了。但是垃圾抽屉的做法实际上使得它难以识别并且难以分解和提取业务模型。

现在，让我们开始。

+ ## 分离值对象(value Objects)

[值对象](http://c2.com/cgi/wiki?ValueObject)是一种依赖于其值而不是他的类型的简单对象。它们通常是不变的。 Date , URI 和 Pathname 是 Ruby 标准库里面的值对象例子，你也可以在你的程序里面定义自己的值对象（当然应该可以）。将它们分离出 ActiveRecord 是一个比较容易做到的重构果子。

在 Rails 里面，值对象特别适用于那些有一些关联逻辑的属性值或者属性值组合。所有不是简单的文本或者计数的情况都值得提取成值对象。

比如说，我以前工作过的一个文本消息应用就有一个叫做 PhoneNumber 的值对象。一个电子商务应用需要一个 Money 类。 Code Climate 有一个叫做 Rating 的值对象，他处理每个类或者模块的 A-F 的等级。我过去用 Ruby 的 String 对象来做，但是 Rating 类让我可以将数据和行为放到一起：

```ruby
class Rating
  include Comparable

  def self.from_cost(cost)
    if cost <= 2
      new("A")
    elsif cost <= 4
      new("B")
    elsif cost <= 8
      new("C")
    elsif cost <= 16
      new("D")
    else
      new("F")
    end
  end

  def initialize(letter)
    @letter = letter
  end

  def better_than?(other)
    self > other
  end

  def <=>(other)
    other.to_s <=> to_s
  end

  def hash
    @letter.hash
  end

  def eql?(other)
    to_s == other.to_s
  end

  def to_s
    @letter.to_s
  end
end
```

每个 ConstantSnapshot 类都暴露出一个 Rating 的实例对象。

```ruby
class ConstantSnapshot < ActiveRecord::Base
  # …

  def rating
    @rating ||= Rating.from_cost(cost)
  end
end
```

除了给 ConstantSnapshot 类减肥以外，它还有其它好处：

1. #worse_than? 和 better_than? 方法提供了一个比 Ruby 内置的操作（比如说 < 和 > ）更好的方式来比较 Rating 。

2. 定义了 #hash 和 #eql? 方法，使我们可以使用 Rating 来作为 hash 的键值(key)。 Code Climate 习惯按照 Rating 来给常量分组。

3. #to_s 方法使我们方便地将 Rating 插入到字符串中。

4. 类的定义给了我们一个很好的地方来放置工厂方法，让我们可以根据给定的”补救时间（修复所有坏味道的预期时间）”返回对应的 Rating 对象。



+ ## 分离出服务对象（Service Objects）

一个系统中的有些 action 需要一个服务对象来封装它们的操作。如果一个 action 满足以下的某个条件，我会使用服务对象。

- action 非常复杂（比如说： 会议结束后合上书本）

- action 关联了好几个模型（比如说：一个电子商务系统中下单过程使用了 Order ， CreditCard 和 Customer 对象）

- action 和其它外部系统有交互（比如说：在社交网络上发贴）

- action 不是根本模型的核心关注点（比如说：一段时间后清除过时数据）

- 有很多方式可以实现这个 action（比如说： 使用 token 或者密码验证用户）。也就是四人帮的[策略模式](http://en.wikipedia.org/wiki/Strategy_pattern)。

我们可以举一个 UserAuthenticator 的 User#authenticate 的例子：

```ruby
class UserAuthenticator
  def initialize(user)
    @user = user
  end

  def authenticate(unencrypted_password)
    return false unless @user

    if BCrypt::Password.new(@user.password_digest) == unencrypted_password
      @user
    else
      false
    end
  end
end
```

SessionsController 就像这样：

```ruby
class SessionsController < ApplicationController
  def create
    user = User.where(email: params[:email]).first

    if UserAuthenticator.new(user).authenticate(params[:password])
      self.current_user = user
      redirect_to dashboard_path
    else
      flash[:alert] = "Login failed."
      render "new"
    end
  end
end
```

+ ## 分离出表单对象(Form Objects)

当一个表单需要更新很多个 ActiveRecord 模型时，一个表单对象可以很好的实现封装。这样比使用 [accepts_nested_attributes_for](http://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html) 要清晰多了， 后者在我看来应该过时了。一个普遍的例子是一个注册的表单，他可能需要创建 Company 和 User 对象：

```ruby
class Signup
  include Virtus

  extend ActiveModel::Naming
  include ActiveModel::Conversion
  include ActiveModel::Validations

  attr_reader :user
  attr_reader :company

  attribute :name, String
  attribute :company_name, String
  attribute :email, String

  validates :email, presence: true
  # … more validations …

  # Forms are never themselves persisted
  def persisted?
    false
  end

  def save
    if valid?
      persist!
      true
    else
      false
    end
  end

private

  def persist!
    @company = Company.create!(name: company_name)
    @user = @company.users.create!(name: name, email: email)
  end
end
```

我们使用 [Virtus](https://github.com/solnic/virtus) 来让这些对象获得 ActiveRecord 一样的功能属性。这个表单对象就像 ActiveRecord 一样。所以，控制器还是和原来差不多。

```ruby
class SignupsController < ApplicationController
  def create
    @signup = Signup.new(params[:signup])

    if @signup.save
      redirect_to dashboard_path
    else
      render "new"
    end
  end
end
```

这样做对于简单的情况是适用的，但是如果表单里面的持久化逻辑非常复杂的话，你可以和服务对象一起使用。另外一个好处是，因为验证逻辑是上下文相关的，它可以定义在关心它的地方，而不是都放在 ActiveRecord 里面。

+ ## 分离出查询对象(Query Objects)

对于弄乱你的 ActiveRecord 子类（比如说 scope 或者类方法）的复杂的查询语句，可以考虑使用查询对象。每个查询对象只负责根据业务规则返回结果集。比如说：一个找出废弃的试验的查询对象可以这样写：

```ruby
class AbandonedTrialQuery
  def initialize(relation = Account.scoped)
    @relation = relation
  end

  def find_each(&block)
    @relation.
      where(plan: nil, invites_count: 0).
      find_each(&block)
  end
end
```

你可以在后台任务里面用它来发邮件：

```ruby
AbandonedTrialQuery.new.find_each do |account|
  account.send_offer_for_support
end
```

自从 ActiveRecord::Relation 实例变成 Rails 3 的一等公民以后，查询对象的参数传递变得更加友好。它让你可以使用组合来合并查询条件：

```ruby
old_accounts = Account.where("created_at < ?", 1.month.ago)
old_abandoned_trials = AbandonedTrialQuery.new(old_accounts)
```

不要担心这样单独的类会变得难以测试。使用测试来将这些对象和数据库合在一起来保证它返回正确的结果，并且关联和预加载都正常工作。（比如：避免 N + 1 查询问题)。


+ ## 介绍 View Objects

如果逻辑仅仅用于显示，那它就不应该归属于模型。问问你自己，“如果在实现这个应用的了一个接口，比如说基于语音的用户界面，我是否需要它？”，如果不是，那就把它放到 helper 或者一个 View Objects 里面。

比如说： Code Climate 里面的环形图打破根据代码库(比如说： [Code Climate 里面的 Rails](https://codeclimate.com/github/rails/rails) )里面的快照算出来的类的 rating 并且封装成一个视图：

```ruby
class DonutChart
  def initialize(snapshot)
    @snapshot = snapshot
  end

  def cache_key
    @snapshot.id.to_s
  end

  def data
    # pull data from @snapshot and turn it into a JSON structure
  end
end
```

我经常发现视图和 ERB(或者 Haml/Slim) 模板是一一对应的。这让我尝试去找出如何将 [Two Step View](http://martinfowler.com/eaaCatalog/twoStepView.html) 模式应用到 Rails 里面，但我还没有找到好的办法。

注意： *这个术语“ Presenter ”是在 Ruby 社区里面提出来的，但我讨厌它，因为他很笨重，使用起来容易和其它东西冲突*
“ Presenter ”这个术语是 [Jay Fields 提出](http://blog.jayfields.com/2007/03/rails-presenter-pattern.html)用来描述我前面说的表单对象的，但是， 不幸的是，Rails 使用“ view ” 来描述不同于“ templates ”以外的东西。为了避免二义性，我有时候把 View Objects 叫做 View Models  。 

+ ## 分离出 Policy Objects

有时候，复杂的读操作需要分别处理它们自己的对象，这时候，我会用 Policy Objects 。这样可以让你将逻辑切片，像找出哪些是活跃用户来达到分析的目的，和你的核心业务对象分离开。比如：

```ruby
class ActiveUserPolicy
  def initialize(user)
    @user = user
  end

  def active?
    @user.email_confirmed? &&
    @user.last_login_at > 14.days.ago
  end
end
```

这个 Policy Objects 封装了一个业务规则：如果一个用户已经验证过邮箱，并且两周以内登录过，则认为他是活跃用户。你也可以使用 Policy Objects 来封装一组业务规则，比如用 Authorizer 来管理一个用户可以处理的数据。

Policy Objects 和服务对象很相似，但是，我用服务对象来完成写操作， Policy Objects 来完成读操作。它们和查询对象也很相似，但是查询对象关注执行查询语句并返回结果集，然后 Policy Objects 对一个已经加载到内存中的模型操作。

+ ## 分离装饰器

装饰器让你可以对现有操作分层，所以它和回调有点像。当回调逻辑仅仅只在某些环境中使用或者将它包含在模型里会给模型增加太多权责，装饰器是很有用的。

给一篇博文加一条评论会触发在某人的 facebook 墙上发一条帖子，但这并不意味着需要将这个逻辑硬编码到 Comment 类。一个你给回调加了太多权责的信号是：测试变得很慢并且很脆弱或者你恨不得将所有不相关的测试屏蔽掉。

这里展示了你如何将 Facebook 发贴的逻辑提取到装饰器里面：

```ruby
class FacebookCommentNotifier
  def initialize(comment)
    @comment = comment
  end

  def save
    @comment.save && post_to_wall
  end

private

  def post_to_wall
    Facebook.post(title: @comment.title, user: @comment.author)
  end
end
```

控制器这样使用：

```ruby
class CommentsController < ApplicationController
  def create
    @comment = FacebookCommentNotifier.new(Comment.new(params[:comment]))

    if @comment.save
      redirect_to blog_path, notice: "Your comment was posted."
    else
      render "new"
    end
  end
end
```

装饰器之所以和服务对象不同，是因为它对权责分层。一旦加上装饰器，使用者就就可将 FacebookCommentNotifier 实例看作 Comment 。在标准库里面， Ruby 利用元编程提供了很多[工具来构建装饰器](http://robots.thoughtbot.com/post/14825364877/evaluating-alternative-decorator-implementations-in)。


## 结束语

即使在 Rails 应用里面， 也有很多工具可以在模型层处理处理复杂性。它们都不需要你抛弃 Rails 。 ActiveRecord 是一个奇怪的库， 如果你严格按照它来做，任何模式都会被打破。尝试将你的 ActiveRecord 限定在持久化存储。在你的业务模型里面使用一些这样的技术来处理逻辑，你会写出一个非常可维护的应用。

你可能意识到了，这里集中模式都介绍得很简单。这些对象都是换种方式来使用 Ruby 原生对象。这就是这部分的观点，也是面向对象编程美的的地方。不需要每个问题都让框架来解决，命名就是一个大问题。

你认为我上面提到的这 7 种方式怎么样？你喜欢那个？为什么？我是不是漏掉了什么？如果是，在评论里面告诉我。


## 进一步阅读

* [Objects on Rails](http://objectsonrails.com/)
* [Crazy, Heretical, and Awesome: The Way I Write Rails Apps](http://jamesgolick.com/2010/3/14/crazy-heretical-and-awesome-the-way-i-write-rails-apps.html)
* [ActiveRecord (and Rails) Considered Harmful](http://blog.steveklabnik.com/posts/2011-12-30-active-record-considered-harmful)
* [Single Responsibility Principle on Rails Explained](http://solnic.eu/2012/07/09/single-responsibility-principle-on-rails-explained.html)

