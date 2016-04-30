---
layout: post
title:  关于Rails ActiveRecord模型关联的dependent选项
date:   2014-04-29 22:30:25
categories: Rails ActiveRecord Association
---

让我们考虑这种情况：
{% highlight ruby %}
  class User < ActiveRecord::Base
    has_one :user_infor
  end

  class UserInfor < ActivRecord::Base
    belongs_to :user
  end

  # some place in controller
  user = User.first
  user.create_user_infor params #=> created a user infor

  # another place in controller
  user.create_user_infor another_parms #=> created a new user infor

{% endhighlight %}
如果在user已经存在一个user_infor的情况下，调用user.create_user_infor，会发生什么情况？在我的机器上（Rails 4.2.6， MySQL）上，会生成如下的sql语句：
{% highlight SQL %}
   BEGIN
  SQL INSERT INTO `user_infors` (`user_id`, `created_at`, `updated_at`) VALUES (2, '2016-04-29 23:55:31', '2016
-04-29 23:55:31')
   COMMIT
   BEGIN
  SQL UPDATE `user_infors` SET `user_id` = NULL, `updated_at` = '2016-04-29 23:55:31' WHERE `user_infors`.`id` = 1
   COMMIT
{% endhighlight %}
可以看到，Rails新建一条user_infor，然后把之前关联的那条user_infor的user_id置为空，产生了一条孤儿数据。这里不做需求辩论，只是设想，如果我需要在调用user.create_user_infor的时候，删除那条老的关联数据，该怎么做？
有些同学说，为了达到这个目的，你可以这样调用：
{% highlight ruby %}
  user.user_infor = UserInfor.new params
  user.save
{% endhighlight %}
association= 这个方法，其实跟调用create_association没多大区别，干的是同样的事情：
{% highlight SQL %}
   (0.1ms)  BEGIN
  SQL (0.4ms)  UPDATE `user_infors` SET `user_id` = NULL, `updated_at` = '2016-04-30 00:30:22' WHERE `user_infors`.`id` = 7
  SQL (0.3ms)  INSERT INTO `user_infors` (`user_id`, `created_at`, `updated_at`) VALUES (2, '2016-04-30 00:30:22', '2016-04-30 00:
30:22')
   (2.7ms)  COMMIT
{% endhighlight %}
还不过顺序调换了下。

还有一个办法是重写create_user这个方法：
{% highlight ruby %}
  def create_user_infor
    user_infor.destory if user_infor.present?
    super
  end
{% endhighlight %}
重写后，调用user.create_user_infor，sql输出：
{% highlight SQL %}
   BEGIN
  SQL DELETE FROM `user_infors` WHERE `user_infors`.`id` = 3
   COMMIT
   BEGIN
  SQL INSERT INTO `user_infors` (`user_id`, `created_at`, `updated_at`) VALUES (2, '2016-04-30 00:13:50', '2016-04-30 00:
13:50')
   COMMIT
{% endhighlight %}
目的达成！但是，这个方法有一个明显的缺点，你只能在调用create_user_infor的时候才能达到这个效果，如果你的同事调用上面提过的 user_infor= 这个方法，你就前功尽弃了。除非你重写全部的写入方法，否则这个缺陷无法避免。
有没有更好的方法？有！就是使用建立模型关联的dependent这个option：
{% highlight ruby %}
  class User < ActiveRecord::Base
    has_one :user_infor, dependent: :destroy
  end

  class UserInfor < ActivRecord::Base
    belongs_to :user
  end
{% endhighlight %}
让我们再次调用user.create_user_infor, SQL输出：
{% highlight SQL %}
   (0.1ms)  BEGIN
  SQL (0.3ms)  INSERT INTO `user_infors` (`user_id`, `created_at`, `updated_at`) VALUES (2, '2016-04-30 00:24:42', '2016-04-30 00:
24:42')
   (4.7ms)  COMMIT
   (0.2ms)  BEGIN
  SQL (0.3ms)  DELETE FROM `user_infors` WHERE `user_infors`.`id` = 4
   (3.4ms)  COMMIT
{% endhighlight %}
任务完成。现在试一试调用user.user_infor= ：
{% highlight ruby %}
  user.user_infor = UserInfor.new params
{% endhighlight %}
SQL输出：
{% highlight SQL %}
   (0.1ms)  BEGIN
  SQL (0.3ms)  DELETE FROM `user_infors` WHERE `user_infors`.`id` = 8
  SQL (0.2ms)  INSERT INTO `user_infors` (`user_id`, `created_at`, `updated_at`) VALUES (2, '2016-04-30 00:34:33', '2016-04-30 00:
34:33')
   (3.3ms)  COMMIT
{% endhighlight %}
同样达到目的。
对于has_many的关系，经过不严格的测试，在调用associations= 及 association_ids= 这两个方法的时候，option这个选项的值，也会产生上述一样的行为。
这说明，建立模型关联的时的dependent选项，不仅仅是在删除模型自身时，影响关联模型，也会对关联模型替换产生影响。而且根据上面的测试可以初步推测，has_one和has_many的dependent选项，如果不设置，默认值为nullify。