---
layout: post
title:  "Remote Forms with Stimulus.js"
author: "Dennis Hellweg"
comments: true
---
![result gif](/assets/remoteformsstimulusresult.gif)

What is shown in the video can be achieved with remote forms in Rails.

First, let's get this working without using a remote form, meaning that 
there will be a page reload when the _Create Message_ button is pressed.

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
app/javascript/channels/message_list_controller.js
{:.filename}
