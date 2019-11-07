---
layout: post
title:  "Using Action Cable with Stimulus"
description: "How to use Action Cable with Stimulus in Rails 6."
author: "Dennis Hellweg"
comments: true
---

Action Cable integrates WebSocket functionalities into Rails. With the help of 
it, one can update the page of a client without them making a request first.
This comes in handy for all kind of real-time applications, for example chat apps
or for notifications.

## What we will build
<video autoplay loop muted playsinline controls class="video">
  <source src="/assets/actioncable_stimulus_result.mp4" type="video/mp4">
</video>

The goal is to update the page for all clients whenever a new message is created by
any of the clients. By using Action Cable, this can happen in real-time, without
a page reload.

By using Stimulus to grab data from Action Cable, you get all the advantages
that Stimulus provide, mainly readable and structured Javascript.

I will now show how to replace the javascript at `app/javascript/channels/message_channel.js` with a
Stimulus controller.

## The Channel

For this example a channel name `MessageChannel` is used. It can be generated

By default, the channel generator in Rails 6 generates two files. For example if
you want to generate a Message channel, you can run `rails generate channel Message`,
which creates the files `app/channels/message_channel.rb` and `app/javascript/channels/message_channel.js`.

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

{% highlight eruby linenos=table %}
<h1>Messages</h1>

<div data-controller='message-list' data-message-list-user='1'>
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
