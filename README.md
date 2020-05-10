# Testing best practices
(Mostly for Ruby and Rspec)

First of all, before writing tests, we should change our mindset. The test is an isolated document. 
They are not a code that you need to improve with principles like DRY and etc.

## Test suite
To create a good test suite you should consider a testing pyramid and TDD as the main approach for writing tests. 

There are a lot of resources, I just add which I liked the most.  
 - https://martinfowler.com/bliki/TestPyramid.html
 - https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530

## Mystery Guests
The most noticeable examples of the wrong mindset are Mystery Guests.

If you have a test similar to this one below, you definitely need to rewrite your tests:
```ruby
  describe 'GET #index' do
    subject { get :index, params: params }

    let(:account) { create :account }
    let(:building) { create :building, account: account }
    let!(:models) { create_list :unit, 6, building: building }
    let(:params) { { page: 1, per_page: 5 } }

    context 'when admin' do
      login_user :user

      before { create(:profile, account: account, user: current_user, role: :admin) }

      include_examples 'search by column examples', :unit_number, 5
      include_examples 'validate resources types', [
        id:              :integer,
        building:        :string,
        number:          :string,
        unit_size:       :string,
        number_of_users: :integer
      ]

      context 'sorting' do
        context 'unit_number' do
          let(:params) { super().merge(order_by: 'unit_number desc') }

          it 'return correct order' do
            subject
            expected = models.map(&:unit_number).sort.reverse[0..4]
            expect(json_body[:resources].map { |unit| unit[:number] }).to eq expected
          end
        end
      end
    end
```


There are several problems with these tests most of them because they are not isolated.

The first problem, when you need to add some new test or change existed one you need to figure out 
what each `let` do and if you need it or not. 

The second problem, readability. It is really hard to understand, in which circumstance, for example, the last test should work. 
You need to understand each `let`, `before`, and `shared examples`. Also `super` is really awful.

There are cool articles wrote by ThoughtBot, which helps you to avoid Mystery Guests:
(DISCLAIMER: `let`, `before` and `shared examples` in most cases are evil )
 - https://thoughtbot.com/blog/mystery-guest
 - https://thoughtbot.com/blog/my-issues-with-let
 - https://thoughtbot.com/blog/lets-not

## BDD

If we open the RSpec home page, there is a title ‘Behaviour Driven Development for Ruby’. 

So, for writing tests in BDD style, you don’t need cucumber or any other frameworks. 

Please, just use the `describe` block to test your feature or method, then inside use the `context` block, which should be starting with when, when, with, without, if, unless, for keywords. 

For auto check it in your project you can use rspec-rubocop gem and use the [next cop](https://www.rubydoc.info/gems/rubocop-rspec/RuboCop/Cop/RSpec/ContextWording).

Please, don’t forget that test is a document, so don’t write description just to fill quotes, please think about it as you think about the naming of variables, methods and classes.
 
Example of a good test: 

```ruby
describe 'user authentication' do
  context 'when a password is incorrect' do 
    it 'shows wrong password popup' do 
       …
    end
   end
end
```
 
## API calls

There are lots of tools that help you to stub and mock API calls. For me, the best is [WebMock](http://github.com/bblimke/webmock) and [VCR](https://github.com/vcr/vcr). 

WebMock gives you opportunities to mock any API call with the ease and flexible interface, for examples if you want to mock connection timeout, you just need to write:
```ruby
stub_request(:any, 'www.example.net').to_timeout
```
You can find lots of examples on how to mock API calls to behave as you want in README for [webmock](http://github.com/bblimke/webmock).

VCR records your test suite's HTTP interactions and replay them during future test runs. For me, it is the truest way to test API calls as VCR writes the real response. We don’t need to create a fake response when we can miss some info. But VCR has a pure functionality to test API errors when you need to mock response. 

The best way to use both VCR and WebMock.

When you install them, there can be some errors as they both trying to manage API calls from your tests.
So, I decided to use WebMock as the main tool and when I need to test real API call I use VCR with adding `:use_vcr` attribute to test.
My `rspec_helper.rb` (if you have a better approach, please share it with me) : 

```ruby
require 'vcr'

VCR.configure do |c|
  c.cassette_library_dir = 'spec/fixtures/vcr_cassettes/'
  c.hook_into :webmock
  c.default_cassette_options = { serialize_with: :yaml }

  c.before_record do |i|
    i.request.headers.delete('Authorization')
    i.response.body.force_encoding('UTF-8')
  end
end

require 'webmock/rspec'

RSpec.configure do |config|
  VCR.turn_off!

  config.before(:each, :use_vcr) do
    VCR.turn_on!
  end

  config.after(:each, :use_vcr) do
    VCR.turn_off!
  end
end
```

## Code architecture
If you notice that you need to write lots of stubs and mock to write a test, It means that you need to change architecture, in most cases adding an abstraction or writing code through DI will help you.

Also without tests, you can not refactor your code, as you are scared of broke some logic. The only good test suite can answer you if you broke something or not when your refactoring.

## Private methods
[Shoud I  test private methods?](http://shoulditestprivatemethods.com/) thanks to [Kent Beck](https://twitter.com/KentBeck).

Private methods are just a implementation details. The behavior of the object has already been tested. 

If you want to test private methods it indicates that something with a separate responsibility wants to be extracted and given a public interface. 

## Performance
If you have problems with performance of tests, consider how much DB calls do you have in your unit tests? Ideally, there are should be no DB calls or any other calls to a third party in unit tests.
Also, a good tool [TestProf](https://test-prof.evilmartians.io/#/). It has nice tools for profiling to improve performance and good practices. 
But be careful, in most cases if you decided to use these practices it means your tests have a smell.

## Test Coverage 
Actually, if you use TDD you don’t need to check test coverage. But if you really need to check it use a tool like a [mutant](https://github.com/mbj/mutant). It shows you more realistic numbers than [SimpleCov](https://github.com/colszowka/simplecov). Consider, that mutation testing is slow and try to use it for a certain feature not for the whole project to reduce time.
