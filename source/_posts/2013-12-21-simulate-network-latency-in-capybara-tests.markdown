---
layout: post
title: "Simulate network latency in Capybara tests"
date: 2013-12-21 21:58:24 +0100
comments: true
categories: ["test", "rails", "capybara", "ajax"]
---

Providing feedback to users when they interact with a website
is an essential part of every modern web application. This is usually done
with some bunch of javascript, coupled with ajax.

This blog post explains how we can write an integration test with Capybara
to test the DOM state between a request send asynchronously to the
rails server, and the response.

<!-- more -->

The obvious test looks like this:

```ruby spec/features/note_creation_spec.rb
feature "Post a note", js: true do
  let(:note) { build(:note) }

  scenario "an ajax spin appears on the form when is submitting" do
    visit new_note_path
    
    page.should have_no_css('img#load')

    within("#new_note") do
      fill_in "note_text", with: note.text
      find('button').click
    end

    page.should have_css('img#load')
  end
end
```

We're initialy testing the non-presence of the load image, this test can be separate in two
parts to keep only one assertion by test, but this is not the point here.

In the Rails world, the `jquery-ujs` gem is the main way to perform
asynchronous request to the server, thanks to the `data-remote` HTML5 custom data
attribute.

``` haml The dom
= form_for @note, remote: true do |f|
  = image_tag('ajax-load.gif', id: :load)
  = f.text_field :note
  = f.button "Save"
```

And for the javascript implementation, we'll use [some custom events](https://github.com/rails/jquery-ujs/wiki/ajax)
added to the defaut jQuery ajax events by `jquery-ujs`:

```javascript The minimal implementation
$('form#new_comment').on('ajax:before', function() {
  $('img#load').show();
});

$('form#new_comment').on('ajax:complete', function(xhr, status) {
  $('img#load').hide();
});
```

However, **our test does not pass**, the request was already performed
when the test evaluated the dom : our spin is not visible at this
moment (and if it's not the case for you, this bevahiour can be random).


`alias_method` to the rescue!
-----------------------------

After some experimentations with [requests mocking on the browser side](https://github.com/oesmith/puffing-billy),
I finally used a method based on `alias_method`, in order to insert a fake latency (with sleep)
before the action treatment.

```ruby spec/support/features/simulate_latency.rb
module Features
  module SimulateLatency
    def simulate_latency_for(controller, method, time=1)
      controller.send(:alias_method, "old_#{method}", method)
      controller.send(:define_method, method) do 
        sleep time
        send("old_#{method}")
      end
    end

    def restore_latency_for(controller, method)
      controller.send(:alias_method, method, "old_#{method}")
    end
  end
end

RSpec.configure do |config|
  config.include Features::SimulateLatency, type: :feature
end
```

In order to respect test isolation, the second method restores the
behaviour of the original one.

The final test uses methods above in rspec hooks:

``` ruby The final test 
feature "Post a note", js: true do
  let(:note) { build(:note) }

  context "when network is slown down" do
    before do
      simulate_latency_for(NotesController, :create)
    end

    after do
      restore_latency_for(NotesController, :create)
    end

    scenario "an ajax spin appears on the form" do
      visit new_note_path
      
      page.should have_no_css('img#load')

      within("#new_note") do
        fill_in "note_text", with: note.text
        find('button').click
      end

      page.should have_css('img#load')
    end
  end
end
```

The sleep isn't a problem here thanks to the test implementation: in fact, no
need to wait for the server response, the test succeeded once the request is sending.

Happy testing!
