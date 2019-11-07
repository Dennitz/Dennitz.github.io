---
layout: post
title:  "Using Action Cable with Stimulus"
description: "How to use Action Cable with Stimulus in Rails 6."
author: "Dennis Hellweg"
comments: true
---

Action Cable integrates WebSocket functionalities into Rails. With the help of 
Action Cable, one can update the page of a browser client without the client making a request first.
This comes in handy for all kind of real-time applications, for example chat apps
or for notifications.

By using Stimulus to grab the data that is sent by Action Cable, you get all the advantages
that Stimulus provides, mainly readable and structured JavaScript.


## What we will build
<video autoplay loop muted playsinline controls class="video">
  <source src="/assets/actioncable_stimulus_result.mp4" type="video/mp4">
</video>

This is a simple example, where any connected browser client can create a new message.
The goal is to update the page for all clients, whenever a new message is created by
any of the clients. By using Action Cable, this can happen in real-time, without
a page reload.

## The Channel

For this example a channel name `MessageChannel` is used. The boilerplate for the
channel can be generated with `rails generate channel Message`.

In Rails 6 this creates two files, `app/channels/message_channel.rb` for server-side code
and `app/javascript/channels/message_channel.js` for client-side code.

For this example the `MessageChannel` sets up all subscribed clients to `stream_from 'message_channel'`.

{% highlight ruby linenos=table %}
class MessageChannel < ApplicationCable::Channel
  def subscribed
    stream_from 'message_channel'
  end

  def unsubscribed
    stop_all_streams
  end
end
{% endhighlight %}
app/channels/message_channel.rb
{:.filename}


The following JavaScript file is generated. We will not use it though and instead 
use a Stimulus controller for the client-side code.

{% highlight js linenos=table %}
import consumer from "./consumer"

consumer.subscriptions.create("MessagesChannel", {
  connected() {
    // Called when the subscription is ready for use on the server
  },

  disconnected() {
    // Called when the subscription has been terminated by the server
  },

  received(data) {
    // Called when there's incoming data on the websocket for this channel
  }
});
{% endhighlight %}
app/javascript/channels/message_channel.js
{:.filename}

## The Stimulus Controller
The Stimulus controller works similar to the one from the 
[previous post]({% post_url 2019-10-29-rails-remote-forms-with-turbolinks-and-stimulus %}).
It has an `input` target, which represents the form field to create a new message, and
a `messages` target, which is the element containing all the messages.

In the `connect` method the channel subscription is created, similar to how it is done 
in the generated JavaScript file. For each of the `connected`, 
`disconnected` and `received` methods, a class method is defined on the Stimulus
controller and passed in when creating the subscription.

The `received` method is the only one that is actually used in this example. It
is supposed to be called with an object that contains a new *rendered* message
on the `message` key, which is then appended to the `messagesTarget`.

{% highlight js linenos=table %}
import { Controller } from 'stimulus';
import consumer from '../channels/consumer';

export default class extends Controller {
  static targets = ['input', 'messages'];

  connect() {
    this.channel = consumer.subscriptions.create('MessageChannel', {
      connected: this._cableConnected.bind(this),
      disconnected: this._cableDisconnected.bind(this),
      received: this._cableReceived.bind(this),
    });
  }

  clearInput() {
    this.inputTarget.value = '';
  }

  _cableConnected() {
    // Called when the subscription is ready for use on the server
  }

  _cableDisconnected() {
    // Called when the subscription has been terminated by the server
  }

  _cableReceived(data) {
    // Called when there's incoming data on the websocket for this channel
    this.messagesTarget.innerHTML += data.message;
  }
}
{% endhighlight %}
app/javascript/controllers/message_list_controller.js
{:.filename}

This Stimulus controller is used in the index view, the view that is shown in the
video. It sets up the required targets and adds an action to clear the input
after successful form submission.

{% highlight eruby linenos=table %}
<h1>Messages</h1>

<div data-controller='message-list'>
  <div data-target='message-list.messages'>
    <%= render @messages %>
  </div>

  <%= form_with(model: Message.new,
                data: { action: 'ajax:success->message-list#clearInput' }
      ) do |form| %>

    <%= form.text_area :content, data: { target: 'message-list.input' } %>
    <%= form.submit style: 'display: block' %>
  <% end %>
</div>
{% endhighlight %}
app/views/messages/index.html.erb
{:.filename}

## Adding a New Message

When the form is submitted to create a new message, the `create` action of the
`MessagesController` is called. 

{% highlight ruby linenos=table %}
class MessagesController < ApplicationController
  def index
    @messages = Message.all
  end

  def create
    @message = Message.new(params.require(:message).permit(:content))
    @message.save!
    ActionCable.server.broadcast('message_channel', message: (render @message))
    head :ok
  end
end
{% endhighlight %}
app/controllers/messages_controller.rb
{:.filename}

It first creates a new message and saves it. Then to update the page on all clients,
a broadcast on the `message_channel` is made on line 9. There, the rendered message partial is
sent along under the `message` key. This will in turn call the `_cableReceived` method 
of the aforementioned Stimulus controller on all subscribed clients.

The controller then simply responds with `head :ok` as no redirect has to be made
and nothing has to be rendered.

