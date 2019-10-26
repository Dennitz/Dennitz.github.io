---
layout: post
title:  "Remote Forms with Stimulus.js"
author: "Dennis Hellweg"
comments: true
---
<!-- ![result video](/assets/remoteformsstimulusresult.mp4) -->

<video autoplay loop muted playsinline controls onclick="this.play()" style="width: 80%">
  <!-- <source src="/assets/remoteformsstimulusresult.mp4" type="video/webm">   -->
  <source src="/assets/remoteformsstimulusresult.mp4" type="video/mp4">
</video>

What is shown in the video can be achieved with remote forms in Rails.

First, let's get this working without using a remote form, meaning that 
there will be a page reload when the _Create Message_ button is pressed.

The model used for this example is a `Message` model which has a column named `content`
to store the actual message.

{% highlight eruby linenos=table %}
<h1>Messages</h1>
<%= render @messages %>

<%= form_with(model: Message.new) do |form| %>
  <%= form.text_area :content %>
  <%= form.submit style: 'display: block' %>
<% end %>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}

{% highlight eruby linenos=table %}
<div>
  <%= message.created_at %>: <%= message.content %>
</div>
{% endhighlight %}
app/views/messages/_message.html.erb
{:.filename}

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


