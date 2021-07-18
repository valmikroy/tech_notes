# Notes on ruby better spec 

Derived from [here](https://www.betterspecs.org/)



### Describe Your Methods

- Be clear about what method you are describing. 
- `.` (or `::`) when referring to a class method's name and 
- `#` when referring to an instance method's name.

```ruby
describe '.authenticate' do
describe '#admin?' do
```

### Use contexts

When describing a **context**, start its description with '**when**', '**with**' or '**without**'.

```ruby
context 'when logged in' do
  it { is_expected.to respond_with 200 }
end

context 'when logged out' do
  it { is_expected.to respond_with 401 }
end
```

### Keep your description short

A spec description should never be longer than 40 characters and designed to read in the way it appeares in the output 

```ruby
context 'when not valid' do
  it { is_expected.to respond_with 422 }
end
```

Output will look like 

```ruby
when not valid
  it should respond with 422
```

### Single expectation test

Isolate expectation based on how broad is your expressions 

```ruby
# isolated
it { is_expected.to respond_with_content_type(:json) }
it { is_expected.to assign_to(:resource) }

# not isolated
it 'creates a resource' do
  expect(response).to respond_with_content_type(:json)
  expect(response).to assign_to(:resource)
end
```

### Test all possible cases

Every outcome of the instance method gathered under single describe scope

```ruby
describe '#destroy' do

  context 'when resource is found' do
    it 'responds with 200'
    it 'shows the resource'
  end

  context 'when resource is not found' do
    it 'responds with 404'
  end

  context 'when resource is not owned' do
    it 'responds with 404'
  end
end
```

### Use new rspec syntax

Adding following to make sure you use new rspec syntax (syntax without `should`)

```ruby
# spec_helper.rb
RSpec.configure do |config|
  # ...
  config.expect_with :rspec do |c|
    c.syntax = :expect	
  end
end
```

New syntax examples 

```ruby
it 'creates a resource' do
  expect(response).to respond_with_content_type(:json)
end

context 'when not valid' do
  it { is_expected.to respond_with 422 }
end
```

### Use Subject

If you have several tests related to the same subject use `subject{}` to DRY them up.

```ruby
subject(:hero) { Hero.first }
it "carries a sword" do
  expect(hero.equipment).to include "sword"
end
```

### Use let and let!

Use `let` instead of `before` to initialize variables before tests.

```ruby
describe '#type_id' do
  let(:resource) { FactoryBot.create :device }
  let(:type)     { Type.find resource.type_id }

  it 'sets the type_id field' do
    expect(resource.type_id).to eq(type.id)
  end
end
```

Use `let` to initialize actions that are lazy loaded to test your specs.

```ruby
context 'when updates a not existing property value' do
  let(:properties) { { id: Settings.resource_id, value: 'on'} }

  def update
    resource.properties = properties
  end

  it 'raises a not found error' do
    expect { update }.to raise_error Mongoid::Errors::DocumentNotFound
  end
end
```

### Mock or not to mock

As general rule do not (over)use mocks and test real behavior when possible, as testing real cases is useful when validating your application flow.

```ruby
# simulate a not found resource
context "when not found" do

  before do
    allow(Resource).to receive(:where).with(created_from: params[:id])
      .and_return(false)
  end

  it { is_expected.to respond_with 404 }
end
```

### Create only the data you need

TODO: Factorybot

```ruby
describe "User" do
  describe ".top" do
    before { FactoryBot.create_list(:user, 3) }
    it { expect(User.top(2)).to have(2).item }
  end
end		
```

### Use factories and not fixtures

```ruby
user = FactoryBot.create :user		
```

### Easy to read matchers

```ruby
expect { model.save! }.to raise_error Mongoid::Errors::DocumentNotFound

```

### Shared example

Rspec shared [example usage](https://relishapp.com/rspec/rspec-core/v/3-2/docs/example-groups/shared-examples) 

```ruby
include_examples "name"      # include the examples in the current context
it_behaves_like "name"       # include the examples in a nested context
it_should_behave_like "name" # include the examples in a nested context
matching metadata            # include the examples in the current context
```

```ruby
describe 'GET /devices' do

  let!(:resource) { FactoryBot.create :device, created_from: user.id }
  let!(:uri)       { '/devices' }

  it_behaves_like 'a listable resource'
  it_behaves_like 'a paginable resource'
  it_behaves_like 'a searchable resource'
  it_behaves_like 'a filterable list'
end		
```

### Don't use should

Do not use should when describing your tests. Use the third person in the present tense. Even better start using the new [expectation](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax) syntax.

```ruby
it 'does not change timings' do
  expect(consumption.occur_at).to eq(valid.occur_at)
end
```

### Automatic tests with guard

```ruby
guard 'rspec', cli: '--drb --format Fuubar --color', version: 2 do
  # run every updated spec file
  watch(%r{^spec/.+_spec\.rb$})
  # run the lib specs when a file in lib/ changes
  watch(%r{^lib/(.+)\.rb$}) { |m| "spec/lib/#{m[1]}_spec.rb" }
  # run the model specs related to the changed model
  watch(%r{^app/(.+)\.rb$}) { |m| "spec/#{m[1]}_spec.rb" }
  # run the view specs related to the changed view
  watch(%r{^app/(.*)(\.erb|\.haml)$}) { |m| "spec/#{m[1]}#{m[2]}_spec.rb" }
  # run the integration specs related to the changed controller
  watch(%r{^app/controllers/(.+)\.rb}) { |m| "spec/requests/#{m[1]}_spec.rb" }
  # run all integration tests when application controller change
  watch('app/controllers/application_controller.rb') { "spec/requests" }
end
```

### Stubbing HTTP requests

Sometimes you need to access external services. In these cases you can't rely on the real service but you should stub it with solutions like webmock.

```ruby
context "with unauthorized access" do

  let(:uri) { 'http://api.lelylan.com/types' }
  before    { stub_request(:get, uri).to_return(status: 401, body: fixture('401.json')) }

  it "gets a not authorized notification" do
    page.driver.get uri
    expect(page).to have_content 'Access denied'
  end
end
```

### Useful formatter

```ruby
# Gemfile
group :development, :test do
  gem 'fuubar'

# .rspec configuration file
--drb
--format Fuubar
--color
```








