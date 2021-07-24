# RSpec Runner
This note is about running spec through ruby code. This will also describe on
how we can modify rspec test output.


## To run spec file

```ruby
require 'rspec'

# Rspec configuration block 

RSpec.configure do |c|
  c.add_formatter(documentation)
  c.fail_fast = true 
  c.color = true
end


ENV['RSPEC_ARG'] = 'rspec argument'

RSpec::Core::Runner.run(['rspec_example_spec.rb'])

RSpec.clear_examples
```

## Modify output format


```ruby
class MyOwnFormatter
  RSpec::Core::Formatters.register self, :dump_summary, :close,  :example_failed, :example_passed

  def initialize(output)
    @output = output
  end

  def example_failed(notification) # FailedExampleNotification
    e = notification.example
    @output << "FAILED #{e.full_description} - #{e.execution_result.exception.message} \n"
  end

  def example_passed(notification) # ExampleNotification
    @output << "OK #{notification.example.full_description}\n"
  end

  def dump_summary(notification) # SummaryNotification
    #@output << "\n\nFinished in #{notification.duration}."
    @output << ""
  end

  def dump_failures(notification) # ExamplesNotification
    @output << "\nFAILING\n\t"
    # For every failed example...
    @output << notification.failed_examples.map do |example|
      full_description = example.full_description
      location = example.location
      message = example.execution_result.exception.message
      "#{full_description}"
    end.join("\n\n\t")
  end

  def close(notification)
    @output << ""
  end
end
```
