---
layout: post
title:  "How to DRY api specs with rspec shared contexts and metadata"
date:   2015-07-11 19:35:13
categories: rails
---

Shared context/shared examples and metadata are great features of RSpec
which can be used to DRY your specs.

Today, I'd like to share some basic usages of them to DRY api specific specs.

## DRY API GET REQUESTS
Before:

~~~ruby
  describe "GET /articles" do
    it "returns status code 200" do
      get articles_path
      expect(response).to have_http_status(200)
    end
  end

  describe "GET /posts" do
    it "returns status code 200" do
      get posts_path
      expect(response).to have_http_status(200)
    end
  end
~~~

After:

~~~ruby
  RSpec.shared_context 'api get request', :get do
    it "returns status code 200" do
      get url
      expect(response).to have_http_status(200)
    end
  end

  describe "GET /articles", :get do
    let(:url) { '/articles'  }
  end

  describe "GET /posts", :get do
    let(:url) { '/posts'  }
  end
~~~

## DRY API POST REQUESTS

Before:

~~~ruby
  describe "POST /articles"  do
    context 'with valid params' do
      let(:valid_params) { { content: 'text' } }
      it 'returns status code 201' do
        post '/articles', valid_params
        expect(response).to have_http_status(201)
      end
    end
    context 'with invalid params' do
      let(:invalid_params) { { content: '' } }
      it 'returns status code 422' do
        post '/articles', invalid_params
        expect(response).to have_http_status(422)
      end
    end
  end

  describe "POST /posts"  do
    context 'with valid params' do
      let(:valid_params) { { content: 'text' } }
      it 'returns status code 201' do
        post '/posts', valid_params
        expect(response).to have_http_status(201)
      end
    end
    context 'with invalid params' do
      let(:invalid_params) { { content: '' } }
      it 'returns status code 422' do
        post '/posts', invalid_params
        expect(response).to have_http_status(422)
      end
    end
  end
~~~

After:

~~~ruby
  RSpec.shared_context 'api post request', :post do
    context 'with valid params' do
      it 'returns status code 201' do
        post url, valid_params
        expect(response).to have_http_status(201)
      end
    end
    context 'with invalid params' do
      it 'returns status code 422' do
        post url, invalid_params
        expect(response).to have_http_status(422)
      end
    end
  end

  describe "POST /posts", :post do
    let(:url) { '/posts'  }
    let(:valid_params) { { text: 'text' } }
    let(:invalid_params) { { text: '' } }
  end

  describe "POST /articles", :post do
    let(:url) { '/articles'  }
    let(:valid_params) { { text: 'text' } }
    let(:invalid_params) { { text: '' } }
  end
~~~

## DRY API AUTHENTICATED REQUESTS

Before:

~~~ruby
  describe "DELETE /articles/:id" do
    let(:article) { Article.create(content: 'text') }
    let(:user) { create(:user) }

    context 'when not authenticated' do
      it 'returns http status code 401' do
        delete "/articles/#{article.id}"
        expect(response).to have_http_status(401)
      end
    end

    context 'when authenticated' do
      before :each do
        login_as(user)
      end
      it 'returns http status code 200' do
        delete "/articles/#{article.id}"
        expect(response).to have_http_status(200)
      end
      it 'deletes the article' do
        pending
      end
    end
  end

  describe "PUT /posts/:id/archive" do
    let(:post) { Post.create(content: 'text') }
    let(:user) { create(:user) }

    context 'when not authenticated' do
      it 'returns http status code 401' do
        put "/posts/#{post.id}/archive"
        expect(response).to have_http_status(401)
      end
    end

    context 'when authenticated' do
      before :each do
        login_as(user)
      end
      it 'returns http status code 200' do
        put "/posts/#{post.id}/archive"
        expect(response).to have_http_status(200)
      end
      it 'archives the post' do
        pending
      end
    end
  end
~~~

After:

~~~ruby
  RSpec.shared_context "authenticated api request", :auth do
    let(:user) { create(:user) }
    before :each do
      login_as(user)
    end

    after :each do
      Warden.test_reset!
    end

    context 'when not authenticated' do
      before :each do
        logout
        send(method, url)
      end
      it 'returns http status code 401' do
        expect(response).to have_http_status(401)
      end
    end

    it 'returns http status code 200' do
      send(method, url)
      expect(response).to have_http_status(200)
    end
  end

  describe "DELETE /articles/:id", :auth do
    let(:article) { Article.create(content: 'text') }
    let(:method) { :delete }
    let(:url) { "/articles/#{article.id}" }

    it 'deletes the article' do
      pending
    end
  end

  describe "PUT /posts/:id/archive", :auth do
    let(:post) { Post.create(content: 'text') }
    let(:method) { :put }
    let(:url) { "/posts/#{post.id}/archive" }

    it 'archives the post' do
      pending
    end
  end
~~~
