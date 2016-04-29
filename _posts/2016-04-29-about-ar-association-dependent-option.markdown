---
layout: post
title:  "关于Rails ActiveRecord关系建立的dependent选项"
date:   2014-04-29 22:30:25
categories: Rails ActiveRecord Association
---

在Rails中使用ActiveRecord，当我们建立关系的时候，不论是has_many，has_one还是belongs_to的关系，Rails都提供了一个dependent的option。那么这个option在各个关系中都有什么作用，以及在不同关系中的行为有哪些不同呢？下面以user及order这两个关联模型来探索一下。
假设user与order的关系如下：
{% highlight ruby %}
  # user.rb
  class User < ActiveRecord::Base
    has_many :orders
  end

  # shop.rb
  class Shop < ActiveRecord::Base
    has_many :orders
  end

  # order.rb
  class Order < ActivRecord::Base
    belongs_to :user
    belongs_to :shop
  end
{% endhighlight %}
不论是has_many，has_one还是belongs_to，dependent option的取值包含destroy和delete这两个值，这两个值得含义很直白，就是在删除（destroy）模型的时候，对关联的模型调用对应的方法（destroy或者delete）。