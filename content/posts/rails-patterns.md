---
title: "RoR patterns to familiarize"
date: 2019-09-14T17:43:33+05:30
draft: false
comments: true
tags:
  - rails
---

Ruby on Rails uphold the philosophy of *convention over configuration*. And once we get the hold of the framework, it becomes really easy to navigate any other codebases which use Rails. The complexity we'd face would be more of the domain at hand rather than the framework. As a Rails developer, we work with and build on top of the abstractions that the framework provides.

When starting with Rails, I grew fond of the abstractions within the framework. We could achieve so much with a few lines of code.
I needlessly tried to write more of abstract code wherever possible - until a senior dev advised me to write *simple and dumb code* instead. This sounded absurd to me at first but then ended up sticking with it.

> “There’s a sweet spot that represents the perfect compromise between comprehension and changeability, and it’s your job as a programmer to find it.” - **Excerpt From: Sandi Metz, Katrina Owen. “99 Bottles of OOP”**

Code is more often read than it is written. And for each abstraction that we'd introduce there is a level of indirection along with it. These indirections in code add to the mental cost for a reader. The reader would need to know about the contracts that these abstractions honors to extend or modify it. Unless the code is to be reused or extended elsewhere, it is better to stay away from it. Moreover trying to do abstractions early could result in having the wrong ones in the code. It becomes more of a boon than bane.

## Patterns to add abstraction with simplicity

As the codebases grow with time, abstractions would eventually be necessary to DRY out the code to provide more changeability. We must do it in a way such that code comprehension is not affected by much. There are certain community adopted abstraction patterns which can be made use of to tackle this. And these are pretty much the patterns everyone agrees on. This is great, as we can leverage these patterns to increase readability and composability in our codebase. Moreover, they can be called as unofficial conventions within Rails community which mostly everyone would know of.

### Service Objects

Service object helps you to extract out the logic that shouldn't necessarily reside within your model or controller. Thus keeping your controller and models lean. These can be logic for making an API call, specific model callbacks or any other domain-specific action. If this logic is to be shared b/w controllers or models then [concerns](https://api.rubyonrails.org/v6.0.0/classes/ActiveSupport/Concern.html) are what you are looking for rather than service objects.

Say you have a blog app wherein once a post is published, you'd want to notify your slack workgroup, create a tweet and then also send out emails to all the users subscribed to your mailing list. Of course you could add these all cases onto the `Post` model callbacks. But I'd advise you not to as callbacks make your code tightly coupled and will later be a pain in the ass. Why do you ask? Check [here](http://samuelmullen.com/2013/05/the-problem-with-rails-callbacks/).

Let's assign this responsibilty to a `PORO` like this:

```ruby
class PostNotificationService
  def initialize(post)
    @post = post
  end

  def perform
    return false unless post.published?

    send_notifications
  end

  private

  def send_notifications
    SlackNotificationService.new(post).perform
    TwitterNotificationService.new(post).perform
    MailingListNotificationService.new(post).perform
  end
end
```

Things to adhere to:

- Have a separate directory to create `app/services/*`
- Keep the class name descriptive to imply its responsibility explicitly.
- Have a single signature such as `#perform` across all services.
- Stick with single responsibility for the service at hand for more composability.
- Exceptions from its responsibility are to be handled within the service itself.
- Introduce namespaces when necessary.

### Form Objects

Ever had trouble when dealing with multiple forms for a single model such that you'd need to perform validations based on the form context? Or say you want certain actions to be performed after a particular form is submitted?

Fear not! Form objects help you to decouple such contexts or actions out from your model. And with `ActiveModel::Model` you still get the attribute assignment and validations like with a model on a `PORO`.

Let us consider an example wherein a user creates a post. The form object for it would look like this:

```ruby
class PostCreateForm
  include ActiveModel::Model

  attr_accessor :title, :content

  validates :title, :content, presence: true

  def initialize(user, params)
    @user = user
    super(params)
  end

  def submit
    return false if invalid?

    @user.posts.create(title: @title, content: @content)
  end
end
```
This includes the validations necessary for the creation and also conforms with `#save` as used with a model. It'd return `false` if any of the validations fail else will create the record and return `true`.

Now let us consider an example for updating the form and making it published. We need to make sure that only the author of the post can update it and send out notifications if it is being published for the first time.

```ruby
class PostUpdateForm
  include ActiveModel::Model

  attr_accessor :title, :content, :published

  validates :title, :content, :published, presence: true
  validate :user_authorized

  def initialize(user, post, params)
    @user = user
    @post = post
    @currently_published = @post.published
    super(params)
  end

  def submit
    return false if invalid?

    result = update_post
    send_notifications unless @currently_published
    result
  end

  private

  def user_authorized
    errors.add(:user_id, :not_authorized) if @post.user != @user
  end

  def update_post
    @post.update(
      title: @title,
      content: @content,
      published: @published,
      published_at: @post.published_at.presence || published_at
    )
  end

  def published_at
    Time.current unless @published
  end

  def send_notifications
    PostNotificationService.new(@post).perform
  end
end
```
I hope you see the flexibility achieved here. We also plugged in the `PostNotificationService` conveniently.

Things to adhere to:

- Have a separate directory to create `app/forms/*`
- Keep the class name to specify form at hand.
- Have a single signature such as `#submit` across all forms.
- The invoking signature should conform to the model's `#save` API.

The corresponding `PostsController` would now look something like this now:

```ruby
class PostsController < BaseController
  ........

  def create
    form = PostCreateForm.new(current_user, params)

    if form.submit
      flash[:success] = "Post created sucessfully"
      redirect_to posts_path
    else
      flash.now[:errors] = form.errors.full_messages.join(', ')
      render locals: { form: form }
    end
  end

  def update
    form = PostUpdateForm.new(current_user, @post, params)

    if form.submit
      flash[:success] = "Post updated sucessfully"
      redirect_to posts_path
    else
      flash.now[:errors] = form.errors.full_messages.join(', ')
      render locals: { form: form }
    end
  end

  ........
end
```

### Decorators

This is a pattern which helps us to decouple view specific logic from the model. It is a bad smell to keep such decoration logic within the model which should contain only business-related logic.

And yes, we do have helpers for this purpose. But the thing with helpers is that it is made available to all the views globally. Furthermore, you can make use of [`#helpers`](https://apidock.com/rails/ActionController/Helpers/ClassMethods/helpers) method to even access it outside the views. It makes no sense to have a certain view specific details to be made available globally.

This is where decorators enter. You can implement easily with the help of [`SimpleDelegator`](https://ruby-doc.org/stdlib-2.5.1/libdoc/delegate/rdoc/SimpleDelegator.html). Have a PORO to inherit from this class and we are good to go.

```ruby
class PostDecorator < SimpleDelegator
  def title
    super.upcase
  end

  def published_at
    I18n.l(super, format: '%d %b %Y')
  end

  def author
    "#{user.first_name} #{user.last_name}"
  end

  def likes
    "#{likes_count} likes"
  end
end
```

All the methods called on this method will be delegated onto the object we pass onto it's constructor. And we also get an opportunity to extend upon it. Pretty neat I would say.

The controller and the corresponding view would be something like this:

```ruby
class PostsController < BaseController
  ........

  def show
    post = Post.find(params[:id])
    post_decorator = PostDecorator.new(post)

    render locals: { post_decorator: post_decorator }
  end

  ........
end
```

```erb
<div class="container">
  <div class="post-title">
    <%= post_decorator.title %>
  </div>
  <div class="post-content">
    <p>
      <%= post_decorator.content %>
    </p>
  </div>
  <div class="post-info">
    <span class="likes"><%= post_decorator.likes %></span>
    <span class="author"><%= post_decorator.author %></span>
  </div>
</div>
```


## Conclusion

By implementation of these patterns, we get to make our code loosely coupled, easily manageable, sanely testable and extendable. Since we honor the contracts for each of these abstractions and being widely adopted within the community - one can effortlessly navigate the codebase.
