---
layout: post
title:  "Custom Attachments in Action Text"
description: "How to use your own models as attachments in Action Text with Rails 6."
author: "Dennis Hellweg"
comments: true
---

Custom attachments allow you use your own models as attachments in Action Text documents.
This, for example, can be helpful when implementing @mentions or if you want to highlight some of your products 
in a blog post.

You can find the code for this post at [https://github.com/Dennitz/ActionTextAttachments](https://github.com/Dennitz/ActionTextAttachments)

## What we will build
We will build a simple project with a `Post` and a `Product` model. The `Post`
model has an Action Text document, in which we'd like to be able to insert a `Product`. 
More specifically, it should be possible to show the `products/product` partial
inside the Action Text document. The result looks like this:

<video autoplay loop muted playsinline controls class="video-w80 shadow">
  <source src="/assets/action_text_attachments.mp4" type="video/mp4">
</video>

When any update is made on the attached model instance, it is automatically reflected in 
the Action Text document:
<video autoplay loop muted playsinline controls class="video-w80 shadow">
  <source src="/assets/action_text_edit.mp4" type="video/mp4">
</video>

## The Models

First, let's create the `Post` model with its controller, views, etc. 
```
rails generate scaffold Post
```

Then add a rich text field named `content`.
{% highlight ruby linenos=table %}
class Post < ApplicationRecord
  has_rich_text :content
end
{% endhighlight %}
app/models/post.rb
{:.filename}

Secondly, create the `Product` scaffold:
```
rails generate scaffold Product name:string 'price:decimal{8,2}'
```

To be able to use a model for custom attachments, the model must have the `attachable_sgid`
method. So let's make it available on the `Product` model by including
`ActionText::Attachable`:

{% highlight ruby linenos=table %}
class Product < ApplicationRecord
  include ActionText::Attachable

  has_one_attached :image
end
{% endhighlight %}
app/models/product.rb
{:.filename}

The `attachable_sgid` method is needed because Action Text uses Signed Global IDs (*sgid*) to identify attachments and their models.
By using the sgid, Action Text can re-render the model's partial every time, instead of just storing its HTML. Because of this, updates to an attached model
are reflected in the Action Text document. 

## The Form

The `Post` form calls `form.rich_text_area` to create the Action Text input. Below that it
has a Bootstrap dropdown, which includes a button for each product name. Each of
those buttons has the product's id added to the button's dataset so that the id
can be used in JavaScript code.

The form is connected to the `attachments` Stimulus controller, which is used to insert
the custom attachment whenever one of the buttons in the dropdown is clicked. 

{% highlight eruby linenos=table %}
<%= form_with(model: post, local: true, 
              data: { controller: "attachments"} ) do |form| %>
  <div class="field">
    <%= form.label :content %>
    <%= form.rich_text_area :content, data: { target: "attachments.editor" } %>
  </div>

  <div class="dropdown">
    <button class="btn btn-secondary dropdown-toggle" 
            type="button" 
            id="dropdownMenuButton" 
            data-toggle="dropdown" 
            aria-haspopup="true" 
            aria-expanded="false">
      Add model attachment
    </button>
    <div class="dropdown-menu" aria-labelledby="dropdownMenuButton">
      <% @products.each do |product| %>
        <%= tag.button product.name, 
          class: "dropdown-item", 
          type: "button", 
          data: { action: "attachments#attach", product_id: product.id } %>
      <% end %>
    </div>
  </div>

  <div class="actions mt-4">
    <%= form.submit %>
  </div>
<% end %>
{% endhighlight %}
app/views/posts/_form.html.erb
{:.filename}

In a real project, instead of using a dropdown to show all product names, you
can, for example, do an autocomplete search through all the models you might want
to include in an Action Text document.

## The Stimulus Controller

Whenever a button in the dropdown is clicked, the `attach` method
of the following Stimulus controller is called.

{% highlight js linenos=table %}
import { Controller } from 'stimulus';
import Trix from 'trix';

export default class extends Controller {
  static targets = ['editor'];

  attach(event) {
    const { productId } = event.target.dataset;

    fetch(`/products/${productId}.json`)
      .then(response => response.json())
      .then(product => this._createAttachment(product))
      .catch(error => {
        console.log('error', error);
      });
  }

  _createAttachment(product) {
    const editor = this.editorTarget.editor;

    const attachment = new Trix.Attachment({
      sgid: product.sgid,
      content: product.content,
    });

    editor.insertAttachment(attachment);
    editor.insertString(' ');
  }
}
{% endhighlight %}
app/javascript/controllers/attachments_controller.js
{:.filename}

The `attach` method gets the product id from the dataset of the clicked button. It
then does a fetch using this id to receive the sgid and the rendered content for that product. 
The fetch triggers the `show` action of the `ProductsController` to be called. 
As the json format is requested, this will render the following file:

{% highlight ruby linenos=table %}
json.partial! "products/product", product: @product
{% endhighlight %}
app/views/products/show.json.jbuilder
{:.filename}

This just renders the following jbuilder partial:

{% highlight ruby linenos=table %}
json.sgid product.attachable_sgid
json.content render(
  partial: 'products/product',
  locals: { product: product },
  formats: %i[html]
)
{% endhighlight %}
app/views/products/_product.json.jbuilder
{:.filename}

This will include the `sgid` of the product and store the rendered HTML partial of
the product under the `content` key.

Next, a `Trix.Attachment` is created in the Stimulus controller using the fetched
sgid and content. This attachment is then inserted into the editor, which will
then show the HTML stored under the `content` key.

Note that this HTML partial is only shown when first inserted as an attachment. 
On subsequent renders, Action Text will use the sgid to
figure out that it is working with a `Product` and thus has to render the
`products/product` partial. This ensures that it will always show 
up-to-date data and allows updates to a product to be reflected in the Action Text
document automatically.

## Styling
Custom CSS may be needed for elements inside an Action Text document. The content
of an Action Text document is wrapped with the `.trix-content` class, so other CSS
can be scoped under this class, e.g.
{% highlight scss linenos=table %}
.trix-content {
  .card {
    max-width: 14rem;
    display: inline-block;
  }
}
{% endhighlight %}
app/assets/stylesheets/actiontext.scss
{:.filename}


## Grouping Attachments
When images are placed side by side in an Action Text document, the image attachments
are grouped with a surrounding *div* that has the `.attachment-gallery` class. By
default, custom attachments are not grouped in a surrounding div.

If you want them to be grouped, you have to adjust Trix's config by calling:
```js
Trix.config.attachments.content = { presentation: 'gallery' }
```
This can, for example, be done in the `connect` method of the Stimulus controller.
