---
layout: post
title:  前端模板 VS SJR（Server-generated JavaScript Responses）
date:   2016-05-07 07:12:00
categories: Rails SJR frontend
---
在使用AJAX跟新数据的时候，经常遇到这种情况：首次渲染模型的模板是Rails模板，而后，将通过AJAX更新或者添加新数据返回的JSON数据使用前端模板（underscore的template）来渲染到DOM中，这样就会需要分别写后端和前端两套模板。这两套模板基本相同，但是一个是Rails模板，一个是js模板，这样产生了重复代码，也加大了工作量和维护难度。

todos/index.html.erb
{% highlight ruby %}
  <h1>Listing Todos</h1>

  <table>
    <tbody>
      <% @todos.each do |todo| %>
        <%= render todo %>
      <% end %>
    </tbody>
  </table>

  <%= render partial: "form", locals: { todo: Todo.new } %>

  <%= render "javascript_template" %>

  <script type="text/javascript">
    $(function() {
      var $tbody = $('tbody');

      $("form").on("ajax:success", function(e, data, status, xhr) {
        appendTodo(data);
        resetNewTodoForm();
      });

      function template() {
        return _.template($("#todo-template").html());
      }

      function appendTodo(todo) {
        var html = template()(todo);
        $tbody.append(html);
      }

      function resetNewTodoForm() {
        $("form")[0].reset();
      }
    });
  </script>
{% endhighlight %}

todos/_form.html.erb
{% highlight ruby %}
  <%= form_for todo, remote: true, format: 'json' do |f| %>
    #.....
  <% end %>
{% endhighlight %}

todos/_todo.html.erb
{% highlight ruby %}
  <tr>
    <td><%= todo.title %></td>
    <td><%= todo.status %></td>
    <td><%= todo.description %></td>
    <td><%= link_to 'Show', todo %></td>
    <td><%= link_to 'Edit', edit_todo_path(todo) %></td>
    <td><%= link_to 'Destroy', todo, method: :delete, data: { confirm: 'Are you sure?' } %></td>
  </tr>
{% endhighlight %}

todos/_javascript_template.html.erb
{% highlight ruby %}
  <% id_placeholder = "id-placeholder" %>
  <script type="text/template" id="todo-template">
    <tr>
      <td>{% raw %}{{ title }}{% endraw %}</td>
      <td>{% raw %}{{ status }}{% endraw %}</td>
      <td>{% raw %}{{ description }}{% endraw %}</td>
      <td><%= link_to 'Show', todo_path(id_placeholder) %></td>
      <td><%= link_to 'Edit', edit_todo_path(id_placeholder) %></td>
      <td><%= link_to 'Destroy', todo_path(id_placeholder), method: :delete, data: { confirm: 'Are you sure?' } %></td>
    </tr>
  </script>
script
  < type="text/javascript">
    $(function() {
      configTodoTemplate('<%= id_placeholder %>');

      function configTodoTemplate(idPlaceholder) {
        var re = new RegExp(idPlaceholder, 'g');
        var html = $('#todo-template').html().replace(re, "{% raw %}{{ id }}{% endraw %}");
        $("#todo-template").html(html);
      }
    });
  </script>
{% endhighlight %}

解决这个问题，有两种方案：完全使用前端模板，和完全使用后端模板。前者我们使用underscore模板，后者使用Rails的erb模板和SJR来实现，SJR即[Server-generated JavaScript Responses](https://signalvnoise.com/posts/3697-server-generated-javascript-responses)。

方案一：全部使用前端模板，列表中所有的todos的都使用前端模板的方式来渲染。删除了后端的todos/_todo.html.erb文件。

todos/index.html.erb
{% highlight ruby %}
<h1>Listing Todos</h1>

<table>
  <tbody>
    <%# empty now %>
  </tbody>
</table>

<%= render partial: "form", locals: { todo: Todo.new } %>

<%= render "javascript_template" %>

<script type="text/javascript">
  $(function() {
    var $tbody = $('tbody');
    var todos = <%= @todos.to_json.html_safe %>;

    _.each(todos, appendTodo);

    $("form").on("ajax:success", function(e, data, status, xhr) {
      appendTodo(data);
      resetNewTodoForm();
    });

    function template() {
      return _.template($("#todo-template").html());
    }

    function appendTodo(todo) {
      var html = template()(todo);
      $tbody.append(html);
    }

    function resetNewTodoForm() {
      $("form")[0].reset();
    }
  });
</script>
{% endhighlight %}

方案二：使用后端模板渲染加上SJR动态更新。

todos/index.html.erb
{% highlight ruby %}
<h1>Listing Todos</h1>

<table>
  <tbody>
    <% @todos.each do |todo| %>
      <%= render todo %>
    <% end %>
  </tbody>
</table>

<%= render partial: "form", locals: { todo: Todo.new } %>

todos/_form.html.erb
<%= form_for todo, remote: true do |f| %>
  #.......
<% end %>
{% endhighlight %}

todos_controller.rb
{% highlight ruby %}
def create
  @todo = Todo.new(todo_params)

  respond_to do |format|
    if @todo.save
      format.html { redirect_to @todo, notice: 'Todo was successfully created.' }
      format.json { render :show, status: :created, location: @todo }
      format.js
    else
      format.html { render :new }
      format.json { render json: @todo.errors, status: :unprocessable_entity }
      format.js
    end
  end
end
{% endhighlight %}

todos/create.js.erb
{% highlight ruby %}
<% if @todo.errors.blank? %>
  (function() {
    var html = "<%= j(render partial: 'todo', locals: { todo: @todo }) %>";
    $("tbody").append(html);
  })();
<% else %>
  (function() {
    var form = "<%= j(render partial: 'form', locals: { todo: @todo }) %>";
    $('form').replaceWith(form);
  })();
<% end %>
{% endhighlight %}