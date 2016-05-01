---
layout: post
title:  重写ActiveRecord模型getter的几种方法
date:   2016-05-02 00:02:25
categories: Rails ActiveRecord attribute
---
设想你需要重写一个ActiveRecord模型的属性的getter方法，比方说name这个方法，这时候很多人都会想到使用read_attribute和[]这两个方法，代码如下：
{% highlight ruby %}
  # overwrite a attribute getter, which's column name is "name"
  def name
    trans_name || read_attribute(:name)
  end

  def name
    trans_name || self[:name]
  end
{% endhighlight %}
但是其实有一种更加直白的方法，在ruby中说到重写（overwriting），怎么能够忘记super？
{% highlight ruby %}
  # overwrite a attribute getter, which's column name is "name"
  def name
    trans_name || super
  end
{% endhighlight %}
这种方式之所以能够工作，是因为getter方法其实不是在ActiveRecord::Base中定义的，而是定义在其他的模块中，然后被include进来的，被include进来的模块成了ActiveRecord::Base的ancestor，所以直接使用super来引用原来的那个方法。
read_attribute，[]和“name”这个三个方法的联系是这样的：
1.[]调用了read_attribute方法。
2.read_attribute方法调用了_read_attribute方法。
3.getter方法（这里是”name“方法）调用了_read_attribute方法。
效果差不多，在实现的细节上还是有一点不同。使用super的优点是，继承了原方法，而不是使用另外一个方法去替代它。