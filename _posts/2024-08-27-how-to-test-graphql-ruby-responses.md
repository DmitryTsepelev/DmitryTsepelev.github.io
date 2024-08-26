---
title: "How to test graphql-ruby responses"
date: 2024-08-27 09:00:00 +0300
permalink: 'how-to-test-graphql-ruby-responses'
description: 'How to test graphql-ruby responses, coming from types, mutations and subscriptions'
tags: ruby rails graphql
---

Most of software engineers write tests. Some people use Ruby. Some of them use GraphQL. While testing queries and mutations has little differences compared to controller specs (at the end of the day—they both use HTTP requests), subscriptions are a bit more tricky. In this short post I'll show you how I test my GraphQL backends.

I will use [RSpec](https://rspec.info), but you can easily port these approaches to any other testing framework you prefer.

# Queries and types

There are two approaches of testing data fetch: you can check either requests or types. In case of requests, you can have a single spec file that fetches a lot of data. If you decide to go with types—you should have a separate spec per type.

I personally prefer the second approach: it's more verbose (and you end up with more files) but also gives you more granular control over the spec and makes it harder to miss something.

> You can even set up Rubocop to enforce every type have a corresponding spec file

What is the responsibility of the GraphQL type? Technically speaking, it's a serializer: you give it some data, it transforms the data and returns the result. You can even extract the transformation to the separate PORO class and test it separately.

Here is an example of the spec I usually create:

```ruby
describe Types::PostType do
  # you can configure RSpec to add this to all specs
  # inside spec/graphql folder
  include GraphqlHelpers

  let(:query) do
    <<~GRAPHQL
      query Post($postId: ID!) {
        post(postId: $postId) {
          id
          title
          content
          author {
            id
          }
        }
      }
    GRAPHQL
  end

  let(:post) { create :post }
  let(:variables) { { postId: post.id } }

  specify do
    perform_request

    expect(gql_errors).to eq(nil)

    resolved_object = gql_data.dig("post")

    # Ideally you should also make sure that you fetch all fields that are
    # resolved by this type
    expect(resolved_object).to match(
      "id" => post.id.to_s,
      "title" => post.title,
      "content" => post.content,
      "author" => {
        "id" => post.author.id.to_s
        # note that we do not check contents of the author field, because
        # it's more straightforward to test Author separately
      }
    )
  end
end
```

There are three things that I care about:

- HTTP request is performed;
- there are no errors (or there are expected errors, in rare cases);
- there is some data returned and contents make sense.

> Do not confuse top–level errors with data–level errors—check out my huge [post](./errors-in-graphql) on that topic

If you do have some complex logic inside the type—you can also add some checks to this spec.

Now let's inspect the `GraphqlHelpers` module I used:

```ruby
module GraphqlHelpers
  extend ActiveSupport::Concern

  included do
    let(:variables) { { input: input } }
    let(:input) { {} }
  end

  def perform_request
    # you will probably handle auth here
    post "/graphql", # replace with your endpoint
      params: JSON.dump(request_params),
      xhr: true
  end

  def json_response_body
    JSON.parse(response.body)
  end

  def gql_data
    json_response_body["data"]
  end

  def gql_errors
    json_response_body["errors"]
  end

  private

  def request_params
    { query: query }.tap do |params|
      params[:variables] = variables if defined?(variables)
    end
  end
end
```

This module encapsulates the boilerplate logic of performing HTTP request and parsing the response, which should be common for all your type specs.

# Mutations

Imagine that we have all types covered with specs, and now we want to test mutations. What is mutation in GraphQL? It's the same thing as query, but we also expect some changes in the state.

As a result, we only care about three facts:

- data was updated;
- mutation returns some data (but we don't need to test the whole type!);
- no errors were returned.

For example:

```ruby
describe Mutations::CreatePost do
  # you can configure RSpec to add this to all specs
  # inside spec/graphql folder
  include GraphqlHelpers

  let(:query) do
    <<~GRAPHQL
      mutation CreatePost($title: String!, $content: String!) {
        createPost(title: $title, content: $content) {
          id
        }
      }
    GRAPHQL
  end

  let(:title) { "First post" }
  let(:content) { "Bla bla bla" }
  let(:variables) { { title:, content: } }

  specify do
    expect { perform_request }.to change(Post, :count).by(1)

    Post.last.tap do |created_post|
      expect(created_post).to have_attributes(title:, content:, author: current_user)

      expect(gql_errors).to eq(nil)

      resolved_object = gql_data.dig("post")

      expect(resolved_object).to match(
        "id" => created_post.id.to_s
      )
    end
  end
end
```

# Subscriptions

Subscriptions are a bit more tricky: the problem is that they can return data two times: first time when client is subscribed (via `def subscribe`) and second time when subscription is triggered. The first case is simple: you just need to perform the query and check the response, like we did for types.

However, how can we test the payload that comes when subscription is triggered?

Let's start with the spec file:


```ruby
describe Api::Types::DailyGoal::EarnTicketEventType do
  include GraphqlSubscriptionHelpers

  let(:query) { <<~GQL }
    subscription {
      postCreated {
        post {
          id
        }
      }
    }
  GQL

  let(:post) { create :post }
  let(:current_user_id) { 42 }
  let(:context) { { channel: mock_channel, current_user_id: current_user_id } }

  specify do
    subscribe(query, context:)
    trigger_subscription(:post_created, payload: { post: }, scope: current_user_id)

    event = mock_channel.mock_broadcasted_messages.first.dig("data", "post")
    expect(event).to match("id" => post.id.to_s)
  end
end
```

We expect the event that was sent to the user to be inside the message queue of the `mock_channel`. In order to get it we need to subscribe to the subscription and emit the event.

All the magic sits in the `GraphqlSubscriptionHelpers`:

```ruby
module GraphqlSubscriptionHelpers
  extend ActiveSupport::Concern

  included do
    let(:mock_channel) { MockSubscriptionCable.fetch_mock_channel }

    # we want queue to be empty when individual example is run
    before { MockSubscriptionCable.clear_mocks }
  end

  def subscribe(query, context:)
    MockSubscriptionCable::GraphqlSchema.execute(query, context:)
  end

  def trigger_subscription(subscription, payload:, scope:)
    MockSubscriptionCable::GraphqlSchema.subscriptions.trigger(:post_created, {}, , scope:)
  end
end
```

We have an accessor to get the mocked channel, and we have two methods—`subscribe` and `trigger_subscription` that imitate the real subscription flow.

Now we need to implement that `MockSubscriptionCable` module. Turns out, that [graphql-ruby gem](https://github.com/rmosolgo/graphql-ruby) has the code we need right in the repo—they need to test subscription [as well](https://github.com/rmosolgo/graphql-ruby/blob/f06879943cd5916bafcdc44b58c7e28d24b2626f/spec/graphql/subscriptions/action_cable_subscriptions_spec.rb). Now we can build our own version inspired by their implementation of the mock subscription queue:

```ruby
class MockSubscriptionCable
  class MockChannel
    attr_reader :mock_broadcasted_messages

    def initialize
      @mock_broadcasted_messages = []
    end

    def stream_from(stream_name, coder: nil, &block)
      block ||= ->(msg) { @mock_broadcasted_messages << msg[:result] }
      MockSubscriptionCable.mock_stream_for(stream_name).add_mock_channel(self, block)
    end
  end

  class MockStream
    def initialize
      @mock_channels = {}
    end

    def add_mock_channel(channel, handler)
      @mock_channels[channel] = handler
    end

    def mock_broadcast(message)
      @mock_channels.each_value { |handler| handler&.call(message) }
    end
  end

  class << self
    def clear_mocks
      @mock_streams = {}
    end

    def server
      self
    end

    def broadcast(stream_name, message)
      stream = @mock_streams[stream_name]
      stream&.mock_broadcast(message)
    end

    def mock_stream_for(stream_name)
      @mock_streams[stream_name] ||= MockStream.new
    end

    def fetch_mock_channel
      MockChannel.new
    end

    def mock_stream_names
      @mock_streams.keys
    end
  end

  # you can inherit from your app's schema or just build a new one
  class GraphqlSchema < ::GraphqlSchema
    use GraphQL::Subscriptions::ActionCableSubscriptions,
      action_cable: MockSubscriptionCable,
      action_cable_coder: JSON
  end
end
```

---

In this post we learned how to test various responses from your Ruby backend powered by GraphQL Ruby.
