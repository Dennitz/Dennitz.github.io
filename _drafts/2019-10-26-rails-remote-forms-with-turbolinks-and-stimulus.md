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

In Rails, forms can be submitted without a page reload by using remote forms.
When using a remote form, Rails submits the form trough an ajax request. Any updates
to displayed view, have then to be done with Javascript. This can either be
automatically handled by Turbolinks or one can manually handle it by listening
to the `ajax:success` and `ajax:error` events. This post shows both approaches and 
specifically how to do the manual handling of the events with Stimulus.

But first, let's get the basics of what is shown in the video working without using a remote form, meaning that 
there will be a page reload whenever a new message is inserted.

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

When the _Create Message_ Button is pressed, a request is made, leading to the `create` action being called.
This action creates a new message and redirects to `messages_path` so that the 
index action is called and the same page, including the new message, is rendered again.
So now the functionality is there, but only with a page reload:
<video autoplay loop muted playsinline controls class="video-w80">
  <source src="/assets/remoteformsreload.mp4" type="video/mp4">
</video>

## Remote Forms with Turbolinks
When using Turbolinks, removing the page reload is as easy as removing `local: true` or 
adding `remote: true`, in case you are using the `form_tag` or `form_for` helper or `simple_form_for` with the `simple_form` gem.
The form will then be submitted remotely and turbolinks will handle the redirect.

In an example as simple as this one, this is probably the best way, as no further 
code changes are needed. But for each new message, the HTML for the complete page,
including the new message, is sent from the server to the client. For a more
complex page, sending just the parts that changed, in this case the new message,
might be advantageous.


## Remote Forms with Stimulus
With this approach, the controller will only return the rendered partial 
of the new message:
{% highlight ruby linenos=table %}
class MessagesController < ApplicationController
  def index
    @messages = Message.all
  end

  def create
    @message = Message.new(params.require(:message).permit(:content))
    @message.save!
    render @message
  end
end

{% endhighlight %}
app/controllers/messages_controller.rb
{:.filename}

Insertion of this partial has to be handled manually, by listening to the 
`ajax:success` event. You could just write some
jQuery to do this, but I'm using Stimulus to keep everything nice and organized.
The view file, when using the Stimulus controller looks as follows:
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
    <%= form.submit style: 'display: block' %>
  <% end %>
</div>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}

In this case, a Stimulus controller named `message-list` is used. It has a 
`messages` target, which holds all the messages, and an `input` target which is
the input field for new messages. When the `ajax:success` event is fired, the
`append` method of the Stimulus controller is called.

{% highlight js linenos=table %}
import { Controller } from 'stimulus';

export default class extends Controller {
  static targets = ['input', 'messages'];

  append(event) {
    const [data, status, xhr] = event.detail;
    this.messagesTarget.innerHTML += xhr.response;

    this.inputTarget.value = '';
  }
}
{% endhighlight %}
app/javascript/controllers/message_list_controller.js
{:.filename}

The `append` method takes the returned partial of the new message and inserts
it after the last message. Then it resets the input.

### Handling Errors
To handle errors one can use a second partial to render an error message and
listen for the `ajax:error` event. The updated controller looks as follows:

{% highlight ruby linenos=table %}
class MessagesController < ApplicationController
  def index
    @messages = Message.all
  end

  def create
    @message = Message.new(params.require(:message).permit(:content))
    if @message.save
      render @message
    else
      render partial: 'error', comment: @comment, status: :bad_request
    end
  end
end

{% endhighlight %}
app/controllers/messages_controller.rb
{:.filename}

And the view file like this:
{% highlight eruby linenos=table %}
<h1>Messages</h1>

<div data-controller='message-list'>
  <div data-target='message-list.messages'>
    <%= render @messages %>
  </div>

  <%= form_with(model: Message.new,
        data: { action: 'ajax:success->message-list#append 
                         ajax:error->message-list#showError' }
      ) do |form| %>
    <%= form.text_area :content, data: { target: 'message-list.input' } %>
    <%= form.submit style: 'display: block' %>
  <% end %>
</div>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}

The Stimulus controller would then have a `showError` method, which would handle
the insertion of the `error` partial.
