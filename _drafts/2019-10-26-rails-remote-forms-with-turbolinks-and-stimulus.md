---
layout: post
title:  "Rails Remote Forms with Turbolinks and Stimulus"
description: "This post shows how to submit a form without reloading the page using either Turbolinks or Stimulus."
author: "Dennis Hellweg"
comments: true
---

<video autoplay loop muted playsinline controls class="video-w80">
  <source src="/assets/remoteformsresult.mp4" type="video/mp4">
</video>

This post will show you how to submit a form in rails without a page reload.
Two approaches are shown, one using Turbolinks and one using Stimulus.

What is shown in the video can be achieved with remote forms in Rails.

First, let's get this working without using a remote form, meaning that 
there will be a page reload when the _Create Message_ button is pressed.

The model used for this example is a `Message` model which has a column named 
`content` to store the actual message. As seen in the video, the index view shows 
all messages and a form to create a new message. In this first version without a
remote form, the following view file is used for that:
{% highlight eruby linenos=table %}
<h1>Messages</h1>
<%= render @messages %>

<%= form_with(model: Message.new, local: true) do |form| %>
  <%= form.text_area :content %>
  <%= form.submit style: 'display: block' %>
<% end %>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}
Notice the `local: true`. When using the `form_with` helper, which is used by the view generator
in Rails 5 and 6, forms are submitted remotely by default. To turn this off,
`local: true` has to be set (which by default is also done by the view generator).

With `<%= render @messages %>` each message is rendered according to the following partial,
which simply shows the message's `created_at` and `content` values:
{% highlight eruby linenos=table %}
<div>
  <%= message.created_at %>: <%= message.content %>
</div>
{% endhighlight %}
app/views/messages/_message.html.erb
{:.filename}

And the controller looks as follows:
{% highlight ruby linenos=table %}
class MessagesController < ApplicationController
  def index
    @messages = Message.all
  end

  def create
    @message = Message.new(params.require(:message).permit(:content))
    @message.save!
    redirect_to messages_path
  end
end
{% endhighlight %}
app/controllers/messages_controller.rb
{:.filename}

When the _Create Message_ Button is pressed, the `create` action will be called.
This creates a new messages and redirects to `messages_path`, which corresponds to
the index page (the same page we were on). So now the functionality is there, but
only with a page reload:
<video autoplay loop muted playsinline controls class="video-w80">
  <source src="/assets/remoteformsreload.mp4" type="video/mp4">
</video>

## Remote Forms with Turbolinks
When using Turbolinks, removing the page reload is as easy as removing `local: true`.
The form will then be submitted remotely and turbolinks will handle the redirect.

In an example as simple as this one, this is probably the best way, as no further 
code changes are needed. 


## Remote Forms with Stimulus
{% highlight eruby linenos=table %}
<h1>Messages</h1>

<div data-controller='message-list'>
  <div data-target='message-list.messages'>
    <%= render @messages %>
  </div>

  <%= form_with(model: Message.new,
        data: { action: 'ajax:success->message-list#append' }
      ) do |form| %>

    <%= form.text_area :content, data: { target: 'message-list.input' } %>

    <div>
      <%= form.submit %>
    </div>
  <% end %>
</div>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}


